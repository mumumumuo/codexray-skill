# RCE 漏洞深度审计

> **风险等级**：🔴 致命 — 可直接接管服务器
> **域 ID**：`rce`

## 审计范围

- 远程命令执行（系统命令注入）：`exec` / `system` / `popen` / `Runtime.exec` / `os.system` / `subprocess` / `shell_exec` 等
- 代码注入：`eval` / `assert` / `Function` 构造器 / `exec()` / `compile()` / 动态 require/import
- 反序列化 RCE：`ObjectInputStream.readObject` / `pickle.loads` / `unserialize` / `Marshal.load` / 不安全的 YAML/JSON 反序列化
- 模板注入 RCE：模板引擎中可执行任意代码的特性（Velocity、Freemarker、Jinja2 SSTI、Go text/template 的危险函数）
- 表达式语言注入：OGNL、SpEL、MVEL、JEXL、EL
- 动态代码加载与反射调用

## 任务要求

- 识别所有命令执行函数调用点
- 分析所有代码执行函数（eval、动态函数等）
- 检查所有反序列化操作点
- 识别模板引擎的代码执行能力
- 分析表达式语言执行点
- 检查动态类加载和反射调用
- 为每个 RCE 漏洞点找到所有可触发的 URL 端点
- 生成每个端点的完整 HTTP 攻击数据包

## 全面性要求

- **方法级追踪**：某方法存在漏洞，必须追踪所有调用该方法的上游路径
- **端点完全枚举**：每个 RCE 类型的所有攻击端点必须全部列出，不得合并
- **参数全覆盖**：同一端点的不同 RCE 触发参数要分别报告
- **权限要求**：触发漏洞所需的最小权限

## 判定纪律

- 命令参数全部硬编码、不含用户输入 → 不是漏洞
- `subprocess.run([...], shell=False)` 列表传参 → 安全模式
- 反序列化标了 `allowedClasses` / `__safe_to_deserialize__` → 不是漏洞
- `eval` 输入完全由常量/配置决定、无用户输入 → 不是漏洞
- 命令执行调用在注释代码、未启用功能分支、受限环境中 → 不可达

## 边界

命令注入 / 代码执行 / 反序列化 RCE / 模板注入 RCE / 表达式语言注入 / 动态加载与反射归本域。

以下类型**绝对不要**输出 finding（见 [domain-boundaries.md](../domain-boundaries.md)）：

- SQL / NoSQL 注入 → `injection`
- 文件上传导致执行（先上传再触发）→ 看落地：若是上传校验缺陷本身 → `file`；若上传安全但服务端对已落地的文件内容做了 eval/反序列化 → 本域
- 模板注入导致 XSS（不执行任意代码只弹框）→ `xss`
- 硬编码的命令执行密钥 / 配置 → `config`

越界线索写进 COVERAGE，例如："在 controller/X.java:88 发现 SQL 拼接，建议 injection 域关注"。

字段 schema、定级原则、POC 要求见 [finding-schema.md](../finding-schema.md)。
