# NInfer 支持 Codex 工具的改造方案（custom 工具 + 客户端工具放行）

> 状态：**已实现**（`custom-tool-support.patch`，6 文件 / 367 行插入，基于上游
> `81c8ce09` 的 `git diff --full-index` 增量）。目标：让 NInfer 的 OpenAI
> Responses 端点接受 Codex 发出的**完整请求**——
> (1) `type:"custom"` 工具通过「降级为 function 工具」让引擎生成调用、客户端执行、
> 响应以 `custom_tool_call` 回显；
> (2) Codex 附带的新工具类型（`web_search`/`tool_search` 等客户端工具）与软性字段
> （`verbosity`、非空 `include`、custom 工具的 `format` 语法成员）做**放行/忽略**，
> 使 `wire_api="responses"` 下 Codex 正常跑通。
> 改动集中在 responses 服务 wire 翻译层，**引擎/模型零改动**。

## 1. 问题现象

Codex 用 Responses API 时，默认把内置能力（shell、file-edit、apply-patch 等）
声明为 `type:"custom"` 工具，用 `input_schema` 字段描述入参（不是 `parameters`）。
NInfer 端点直接 400：

```json
{"error":{"code":"tool_type_not_supported",
          "message":"tool type 'custom' requires an executor that NInfer does not provide",
          ...}}
```

NInfer 是纯推理引擎，没有 Codex 那种「由客户端执行工具、引擎只负责生成调用」
的 executor，所以拒绝 `custom`。要让它能跑，就是把 `custom` 降级成普通的
`function` 工具（引擎生成 `function_call`，调用方自己执行）。

## 2. 根因定位

### 2.1 报错点

`src/serve/openai_responses_request.cpp` 的 `parse_tools()`（约 line 796）：

```cpp
if (type != "namespace") {
    bad_request("tool type '" + type +
                    "' requires an executor that NInfer does not provide",
                "tools", "tool_type_not_supported");
}
```

只放行 `namespace`，`custom` 走不到后面的 function 解析，直接 400。

### 2.2 function 工具解析只认 `parameters`

`parse_function_tool()` 的 `allowed_members` 只接受：
`{"type","name","description","parameters","strict","allowed_callers","defer_loading","output_schema"}`
——只读 `parameters`，而 `custom` 工具用的是 `input_schema`，字段名对不上。

### 2.3 输入侧只认 `function_call` / `function_call_output`

`parse_input()`（约 line 638）只处理 `function_call` / `function_call_output`
两种 item；`parse_function_call_item()`（line 390）与
`parse_function_call_output_item()`（line 440）都按 function 语义解析。
Codex 多轮回放时，input 里**既有 assistant 侧的 `custom_tool_call`，也有
`custom_tool_call_output`**，两者都要新增分支（原方案只提了 output，漏了 call）。

### 2.4 响应侧只发 `function_call`

`src/serve/openai_responses_response.cpp`（line 139-140、452、473）
只在 `type:"function_call"` 处生成 item 和 SSE（`function_call_arguments.delta/.done`），
没有 `custom_tool_call` 的路径。

### 2.5 相关既有测试（**无需翻**，本次澄清）

- `tests/test_openai_responses.cpp` line 782：测的是 **namespace 里的嵌套 custom**，
  本次只放开**顶层** custom、嵌套仍拒 → 该断言**保持有效**。
- `tests/test_openai_schema.cpp` line 389：它的 `parse` 返回 `OpenAIChatRequest`，
  走的是 **Chat Completions** 路径（`openai_chat_request.cpp`），本次**未改** chat，
  故「chat 拒 custom」断言**保持有效**。

结论：两条既有断言都不用动；新增 `tests/test_openai_responses.cpp::test_custom_tool_lowering`
覆盖顶层 custom 的接受 / 降级 / 出参 / 多轮回放。

### 2.6 响应侧如何知道某个 call 是 custom

引擎内部对 function 与 custom 一视同仁：模型只产出
`GeneratedToolCall{name, arguments_json}`（`include/ninfer/types.h:288`），
function/custom 的区分纯粹是**出参 wire 翻译**问题。响应侧
`openai_responses_response.cpp` 拿到的是 `OpenAIResponsesCreateRequest`
（含 `tool_identities`，见 `openai_responses.h:49`）和 `outcome.tool_calls`，
需要按 `call.name`（engine 名）判断是否 custom。

推荐：在 `OpenAIResponsesCreateRequest` 上新增
`std::unordered_set<std::string> custom_tool_engine_names`（与 `tool_identities` 并列），
由 `parse_tools` 填入。**不要**扩展 `OpenAIResponsesFunctionIdentity` 加 bool 标记——
它的 `operator==` 是默认生成的（`openai_responses.h:29`），被
`lower_function_identity` 的冲突检测（`request.cpp:77`）依赖，加字段会搅动去重语义。

### 2.7 其他受影响的边角（低风险，需一并处理）

- `openai_responses_http.cpp:159` `redact_input_image_urls` 只识别
  `function_call_output`；stored response 列表/详情里 custom 的 output 图片不会被脱敏。
  补一个 `custom_tool_call_output` 分支即可（低优先级）。
- `openai_responses_state.cpp` 的 `normalize_call_graph` 是**类型无关**的
  （按解析后的 `ChatTurn.tool_calls` / `role==Tool` 校验），把 custom item 映射成
  同一套 `ToolCall`/Tool turn 后**无需改动**。
- `filter_allowed_tools`（`request.cpp:838`）仍只接受 `function` 条目；
  Codex 对内置工具用 `tool_choice:"auto"`，不在 `allowed_tools` 里，
  维持现状即可，文档注明。

## 3. 结论：`--chat-template` 解决不了

`qwen3.8-tolerant-sys.jinja` 那类 chat template 只作用于「messages → prompt token」
这一层，属于提示词拼接。而 `custom` 报错发生在 **API 层的工具校验**（`parse_tools`），
早于任何模板渲染。换模板不动报错点，**必须改 NInfer 源码**。

## 4. 改造方案（约 140 行，4 个文件 + 测试）

改动集中在 responses 服务的 wire 翻译层，**引擎/模型零改动**：
`openai_responses_request.cpp`、`openai_responses_response.cpp`、
`openai_responses_http.cpp`（边角）、`openai_responses.h`（加一个字段）。

核心思路：**在入口把 `custom` 当 `function` 处理**——`input_schema` 映射为 `parameters`，
内部标记「这是 custom」，出参和回传都走 `custom_tool_call` 的语义，其余复用现有 function 通道。

### 4.1 `openai_responses_request.cpp`（约 70 行）

1. **`parse_tools()` 新增 `custom` 分支**（line 794 的 `if (type != "namespace")` 处）：
   - `type=="custom"` 不再 400，改走 function 解析路径。
   - `parse_function_tool()` 的 `allowed_members`（line 675）增加 `input_schema`；
     读取参数时**custom 优先取 `input_schema`，缺省回落 `parameters`**，
     结果照旧存入 `parsed.definition.input_schema_json`（该字段已存在，`request.h:82`）。
   - 把 `lower_function_identity` 返回的 engine 名插入
     `out`（`ParsedPromptFields`）新增的 `std::unordered_set<std::string> custom_tool_names`；
     最终随 request 落到 `OpenAIResponsesCreateRequest::custom_tool_engine_names`（§2.6）。
   - custom 的 `canonical` wire item 用 `{"type","custom_tool_call"}` 而非 `"function"`，
     且把 `input_schema` 原样回显，保持 wire 忠实。

2. **`parse_input()` 新增两个分支**（line 638 处，紧跟 `function_call_output`）：
   - `custom_tool_call`（assistant 侧，多轮回放）：仿 `parse_function_call_item`
     （line 390），但读 `input`（字符串，非 `arguments`）填 `call.arguments_json`，
     canonical 用 `{"type","custom_tool_call"}`；经 `append_call` 汇入 `AssistantInputRun`，
     该 run 是阶段机（Empty/Reasoning/Content/Calls，line 537），类型无关，无需改。
   - `custom_tool_call_output`：复用 `parse_function_call_output_item`（line 440）逻辑，
     仅把 canonical 的 `type` 与报错文案改为 `custom_tool_call_output`。

### 4.2 `openai_responses_response.cpp`（约 45 行）

1. **built 响应**（`build_response`，item 循环 line 131-146）：
   判定 `request.custom_tool_engine_names.contains(call.name)`；
   命中则 item `type` 发 `custom_tool_call`，字段用 `input`（= 原始 `call.arguments_json`
   JSON 字符串，Codex 期望 raw string，非对象），不再用 `arguments`。
2. **SSE 流**（`finish_terminal` 的 call 循环 line 445-481）：
   function 走 `function_call` item + `response.function_call_arguments.delta/.done`
   （delta 在 line 461-464，done 在 line 466-471）；
   custom 改发 `custom_tool_call` item + `response.custom_tool_call_input.delta/.done`
   （`delta`/`input` 字段），与 function 事件区分。
3. **history 回放**（`build_response` line 148-168 的 `history.tool_calls`）：
   内部 `ToolCall` 结构不变（`GeneratedToolCall` 复用），wire 层的 custom 化已在
   item/SSE 步骤完成，`history` 只供下一轮 prompt，无需区分。

### 4.3 边角修正（约 5 行）

- `openai_responses_http.cpp:159`：`redact_input_image_urls` 增补
  `custom_tool_call_output` 分支（stored response 脱敏一致性，低优先级）。

### 4.4 测试（约 40 行）

- **既有断言零翻转**（§2.5 澄清：nested 与 chat 两条都仍有效）。
- 新增 `tests/test_openai_responses.cpp::test_custom_tool_lowering`：
  1. 顶层 custom 工具被接受 → `generation.tools` 有该名、`custom_tool_engine_names` 含它、
     wire `tools` 回显 `type:"custom"` + `input_schema`（无 `parameters`）。
  2. 出参：outcome 里同名 tool_call → output item `type=="custom_tool_call"` 且 `input` 为原始
     JSON 字符串（非 `arguments`）。
  3. 多轮回放：input 含 `custom_tool_call` + `custom_tool_call_output`，
     断言解析成 assistant ToolCall + Tool turn。
- 回归：纯 function 工具场景输出零 diff（既有 function 用例不变即覆盖）。

## 5. 风险点

| 风险 | 说明 | 规避 |
|---|---|---|
| 工具名冲突 | function 与 custom 同名会撞 | 已由 `declared_names`/`lower_function_identity` 的 `duplicate_tool_name` 兜底 |
| `input` 类型 | Codex 要 raw JSON string | 响应侧 custom 的 `input` 直接透传 `arguments_json` 字符串，不再对象化 |
| 多轮回放 | history 里 custom 调用格式必须和响应一致 | 响应与回放共用 `custom_tool_engine_names` 判定 |
| 标记放哪 | 别搅动 identity 去重 | 用独立 `custom_tool_engine_names` 集合（§2.6），不扩 `OpenAIResponsesFunctionIdentity` |
| `allowed_tools` | custom 进不了 `filter_allowed_tools` | Codex 用 `auto`，非阻塞；文档注明 |
| 向后兼容 | 现有 function 工具不受影响 | 改动全部 gate 在 `type=="custom"` 分支，function 路径零改动 |

## 6. 工作量与验证

- 代码量：~140 行（request ~70 + response ~45 + http 边角 ~5 + header 字段 + 测试 ~20）。
  引擎/模型零改动，纯 serving wire 层。
- 验证顺序：
  1. 单测：翻掉的两条断言 + 新增「custom 工具 → `custom_tool_call` 输出（`input` 为 raw string）」
     + 多轮「`custom_tool_call` + `custom_tool_call_output` 入参」用例。
  2. 集成：Codex `codex-local dash`（走 cloud-proxy）不受影响；
     `codex-local`（local-llm，NInfer 端点）应能正常发 shell 工具调用。
  3. 回归：纯 function 工具场景输出零 diff。

## 7. 相关文件索引

- `src/serve/openai_responses_request.cpp`
  — `parse_tools` (767) / `parse_input` (593) / `parse_function_call_item` (390) /
  `parse_function_call_output_item` (440) / `parse_function_tool` (671)
- `src/serve/openai_responses_response.cpp`
  — built item 循环 (131-146) / SSE call 循环 (445-481)
- `src/serve/openai_responses_http.cpp` — `redact_input_image_urls` (148)
- `src/serve/openai_responses.h` — `OpenAIResponsesCreateRequest` 加 `custom_tool_engine_names` (49 附近)
- `src/serve/openai_responses_state.cpp` — `normalize_call_graph` (43)，**类型无关，无需改**
- `tests/test_openai_responses.cpp` — 新增 `test_custom_tool_lowering` +
  `test_client_side_tool_passthrough`；`:792`（namespace 嵌套 custom）不动
- `tests/test_openai_schema.cpp` — **不动**（属 Chat Completions 路径，其
  「custom tools rejected」断言针对 Chat，仍有效）
- `src/serve/openai_responses_http.cpp` — 新增 env 门控诊断 `NINFER_DUMP_TOOLS`
  （`handle_responses` 里把请求 body dump 到 stderr；默认关，仅诊断用，可删）

## 8. 后续扩展：放行 Codex 完整请求（本轮）

custom 工具放行后，Codex 仍 400。用 `NINFER_DUMP_TOOLS=1` 抓到 Codex 的**真实请求体**，
发现 `tools` 之外还有多处硬校验拦截（按 parser 处理顺序）：

| # | 位置 | Codex 发的 | 原 400 | 本轮处理 |
|---|---|---|---|---|
| 1 | `tools` | `apply_patch`（`type:"custom"`）带 `format`（Lark 语法对象）| `unsupported tools member: format` | `format` 进 `parse_function_tool` 的 `allowed_members`，当**不透明成员忽略**（引擎不强制语法）|
| 2 | `tools` | `tool_search`（`execution:"client"`）、`web_search`（`external_web_access:false`）—— 两种 `function`/`custom`/`namespace` 之外的**新类型**，且**无 `name`**（靠 `type` 标识）| 会命中 `tool_type_not_supported` | `parse_tools` 里 `type != "namespace"` 的拒绝改成「**客户端工具**」分支：按 `name`（缺省回落 `type`）降成可调用 `ToolDefinition`（取 `parameters` schema，无则空对象），**原样回显**进响应 `tools`，其余成员（`execution`/`external_web_access`…）忽略。调用由客户端执行，NInfer 只负责让引擎能提出调用 |
| 3 | `text` | `verbosity:"low"` | `verbosity_not_supported`（原只认 `medium`）| 接受 OpenAI 枚举 `low`/`medium`/`high` 并**忽略**（引擎无 verbosity 控制）|
| 4 | `include` | `["reasoning.encrypted_content"]`（非空）| `include_not_supported` | 接受字符串数组并**忽略**字段值（Responses 响应字段集固定；`client_metadata` 同类，早已当不透明 hint）|

### 8.1 为什么客户端工具用「降级 + 原样回显」而非丢弃

- 丢弃会让模型丢失 Codex 期望的工具（`web_search`/`tool_search`），改变语义；
- 降级成（可能无 schema 的）function 后，引擎可照常提出 `function_call`，
  Codex 客户端执行并回传 `function_call_output`（走既有 function 通道，无需新增 item 类型）；
- 原样回显保证响应 `tools` 数组与请求一致，多轮回放不丢字段。
- `web_search` 无 `parameters` → 空对象 schema；`schema_param` 置空，不触发约束解码。

### 8.2 契约/文档/测试同步

- `docs/serving.md`：Responses 字段表更新 `tools`（+custom/客户端）、新增 `text.verbosity` 行、
  `include` 行改为「接受、字段省略」；「Function tools」与「Unsupported Create fields」两节
  把 custom/客户端工具从「不支持」改述为「降级 + 回显 / 客户端执行」，
  OpenAI-hosted / remote MCP 执行器**仍不支持**。
- 新增 `tests/test_openai_responses.cpp::test_client_side_tool_passthrough`：
  JSON 字面量还原 Codex 的 3 类工具（custom+format、nameless `web_search`、带 schema 的
  `tool_search`）+ `verbosity:"low"` + 非空 `include`，断言：3 个工具按 name/type 降级、
  custom 仍进 `custom_tool_engine_names`、nameless 工具原样回显不造字段、custom 的
  `format` 不回显、未知 verbosity / 非 string include 条目仍拒。
- Chat 端点（`test_openai_schema.cpp`）**不动**：客户端工具放行只作用于 Responses。

### 8.3 验证

- 相关 host 测试全绿：`openai_responses` / `openai_responses_store` / `openai_schema` /
  `anthropic_schema` / `http_transport` 均 `ok`。
- 集成：重启 server 后 Codex（`wire_api="responses"`，local-llm）请求通过、正常出参。

### 8.4 重放方法（拉取最新代码后）

1. 在最新代码上 `git apply --3way custom-tool-support.patch`（或逐文件套用）；
   若上游又动了 `parse_function_tool` 签名 / `allowed_members` / `ToolDefinition`，
   按 §2/§4 的锚点手工对齐（custom 分支 `schema_param` 用 `tools/<i>/input_schema`，
   客户端工具用 `tools/<i>/parameters`，两者都保留）。
2. 编译：`cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release`（加
   `-DBUILD_TESTING=ON` 若要跑测试），`cmake --build build -j`。
3. 跑 `./build/tests/ninfer_openai_responses_test`（含两条新用例），再用 Codex 实跑验证。
