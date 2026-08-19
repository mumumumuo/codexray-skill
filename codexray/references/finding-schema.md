# Finding 字段 schema 与定级纪律

每条漏洞都按下面的字段输出。字段不全的 finding 等于没写完 —— 读报告的人拿不到复现路径，修的人不知道改哪里。

## 字段清单

| 字段 | 必填 | 说明 |
|------|------|------|
| `vuln_type` | ✅ | 漏洞类型（SQL注入 / 命令执行 / 越权 …） |
| `severity` | ✅ | `critical` / `high` / `medium` / `low` |
| `confidence` | ✅ | `confirmed` / `suspected` |
| `title` | ✅ | 漏洞标题（中文），点明接口 + 参数 + 类型 |
| `file_path` | ✅ | 漏洞所在文件路径 |
| `line_number` | ✅ | 行号 |
| `min_privilege` | ✅ | `none` / `user` / `admin` —— 触发所需最小权限 |
| `description` | ✅ | 详细描述（中文）：输入从哪来、怎么流到 sink、为什么能利用 |
| `severity_reason` | ✅ | 定级理由，见下方定级原则 |
| `code_trace` | ✅ | `source`（用户输入来源）、`sink`（触发点代码）、`call_chain`（调用链数组）、`route`（HTTP 路由） |
| `evidence` | ✅ | 关键代码片段原文 |
| `poc` | ✅ | raw 格式 HTTP 请求包，或 Python 脚本，或发现位置说明 |
| `expected_result` | 条件 | 与 poc 对应的 HTTP 响应预期。可触发类必填，配置类可省 |
| `fix_suggestion` | ✅ | 修复建议（中文），给出具体写法而非「建议加强校验」 |

### confidence 的分界

- `confirmed` —— 通过代码追踪确认可利用：输入可控、路径可达、无有效防护，三者都验证过
- `suspected` —— 存在风险但需特定运行时条件，或代码证据不足以完全确认

不可利用、安全模式、误报、纯设计建议**不得**包装成 finding。这类观察写进覆盖度小结。

## 定级原则

严禁按漏洞类型机械套高危。核心依据是**实际影响** —— 能否修改数据、接管账号、泄露凭证/密钥、批量导出敏感数据、组成攻击链。

- **critical** —— 可直接导致 RCE、服务器/集群接管、管理员或系统身份接管、大规模关键数据破坏，或完整攻击链闭环
- **high** —— 高权限操作、账号接管、关键业务篡改、资金/订单等核心业务影响、凭证或密钥泄露、敏感数据批量导出
- **medium** —— 影响有限但有明确安全后果：无认证或越权场景下泄露手机号、邮箱、身份证、地址等个人信息；登录用户可越权读取他人较敏感数据；局部业务绕过；需特定权限或条件才能利用
- **low** —— 只读低影响信息查看、普通低敏字段枚举、单点低敏信息泄露、无修改/接管/批量导出证据的访问控制瑕疵

三条硬底线：

1. 只读低敏查看，没有证明可修改数据、接管账号、泄露凭证、批量导出或组成攻击链 → 通常只能是 `low`
2. 无认证或越权场景下泄露个人信息（手机号、邮箱、身份证、地址）→ 最低 `medium`
3. 泄露密码、Token、Cookie、密钥等凭证 → 最低 `high`

定为 `medium` 或 `high` 时，`severity_reason` 必须列出具体证据，覆盖可达性、权限要求、影响范围、数据敏感度、能否修改/接管/批量泄露/组成攻击链。

## POC 要求

**可触发类**（注入、RCE、XSS、越权、SSRF、文件操作等）必须给 raw 格式 HTTP 请求包，以及对应的 `expected_result`。`expected_result` 要包含状态行、关键响应头、关键 JSON/HTML 字段或数据变化 —— 不要只写「成功」或一句文字结论。

```
POST /api/logs/search HTTP/1.1
Host: target
Content-Type: application/json

{"query": "<注入 payload>"}
```

**脚本类 POC（Python）** —— 当 POC 无法用单个 HTTP 请求表达时，`poc` 字段给完整可执行的 Python 脚本。典型场景：

- 需要代码构造 payload：JWT 空密钥伪造、反序列化 gadget 构造、签名绕过
- 需要并发或时序操作：条件竞争、并发重复提交
- 需要多接口联动：先注册再提权、先取 token 再利用

统一用 Python，不要混用 Bash / Go / Java。

**配置类与信息泄露类**（硬编码密钥、调试模式、CORS 错误等）无需 HTTP POC，在 `poc` 字段写清发现位置和内容即可，`expected_result` 可省略。

## 审计深度要求

- **方法级追踪**：一个方法存在漏洞，必须追踪所有调用该方法的上游路径
- **端点完全枚举**：每个漏洞类型的所有攻击端点必须全部列出，不得合并
- **参数全覆盖**：同一端点的不同漏洞参数要分别报告
- **不要只给示例**：禁止「如 xxx 等」的写法，必须逐一列出所有发现
- **主动扩展**：起点线索只是起点，应主动扩展追踪更多调用链

## 输出形态

子代理按 FINDINGS / COVERAGE / CROSS 三节返回纯文本（见各域派发 prompt），编排者负责写进 HTML 报告。上面的 schema 字段与报告漏洞卡片（`<div class="vuln-card">`）的字段块一一对应：

| schema 字段 | HTML 字段块 |
|---|---|
| `title` | vuln-title 文本 |
| `vuln_type` | vuln-type-tag |
| `severity` | 卡片类名 `vuln-{severity}` + 属性 `data-sev` + sev-tag 类名 `sev-{severity}`，标签文本 CRITICAL/HIGH/MEDIUM/LOW |
| `confidence` = suspected | `<span class="suspected-tag">疑似</span>` + 卡片属性 `data-confidence="suspected"` |
| `min_privilege` | priv-tag |
| `description` | 「描述」detail-text |
| `severity_reason` | 「定级评判」reason-box |
| `poc` | 「POC」poc-code（带复制按钮） |
| `expected_result` | 「预期响应」expected-response-code（配置类可省） |
| `file_path` + `line_number` | 「位置」location-code |
| `code_trace` | 「调用链」trace-box：Route / Source / 链路（chain-fn + chain-sep）/ Sink（sink-hl） |
| `evidence` | 「证据」evidence-code |
| `fix_suggestion` | 「修复建议」fix-text |
| 归属审计域 | 卡片属性 `data-stage`（rce/injection/file/auth/business/xss/config/other） |
| 编号 | 卡片属性 `data-fid`（V-001 依次） |

计数标签、审计汇总、漏洞清单表、修复建议列表由页面 JS 从卡片 DOM 自动生成，不要手填。HTML 报告是唯一交付物，不输出 JSON 等其他格式。

## 覆盖度小结

每个域审完后附一段小结，说明：

- 检查了哪些文件和调用链
- 范围内未审到的部分（写清哪些区域没有覆盖到）
- 越界发现：属于其他域的漏洞线索，写清位置 + 现象 + 建议归属域，**不要**在本域输出 finding

已完整覆盖且未发现漏洞时，明确写「已完整覆盖」并说明检查了什么 —— 这与「没审到」是两件事，最终输出的「分域」一行要用 `0(已覆盖)` 与 `0(缺口)` 区分开。

## 一致性要求

如果已经输出过 finding，就**不能**在总结里写「未发现可利用漏洞」「无漏洞」这类表述 —— 自相矛盾会误导读报告的人。finding 数量必须与实际输出的去重后条数一致。
