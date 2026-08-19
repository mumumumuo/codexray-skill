# 注入类漏洞深度审计

> **风险等级**：🟠 严重 — 可导致数据泄露/篡改
> **域 ID**：`injection`

## 审计范围

- **SQL 注入**：所有数据库操作点（原生 SQL、字符串拼接 SQL、动态 ORM 查询）
- **NoSQL 注入**：MongoDB、Redis、Elasticsearch 等 NoSQL 数据库的查询拼接
- **LDAP 注入**：LDAP filter 拼接
- **XPath 注入**：XPath 查询拼接

## 任务要求

- 枚举所有数据库查询操作点
- **区分参数化查询（安全）与字符串拼接（危险）**：参数化查询不输出 finding，仅在拼接场景标记为漏洞
- 识别所有 NoSQL / LDAP / XPath 查询拼接位置
- 为每个漏洞点找到所有可触发的 URL 端点
- 生成每个端点的完整 HTTP 攻击数据包

## 全面性要求

- **方法级追踪**：DAO 层方法存在漏洞，追踪所有 Service / Controller 调用路径
- **端点完全枚举**：所有触发同一漏洞的端点必须全部列出
- **参数全覆盖**：同一端点不同的注入参数分别报告

## 越权分析要求

- **认证状态**：明确标注「鉴权」或「未授权」
- **权限要求**：触发漏洞所需的最小权限

## 判定纪律

- ORM `.Where("field = ?", userInput)` 参数化 → 安全，不输出 finding
- 框架自动预编译所有查询（如 GORM 默认行为）→ 安全
- 输入经严格白名单（只能是整数 ID）→ 安全
- 查询在 `if debug_mode` 分支内、生产不执行 → 不可达
- 用户输入被拼到结构部分（关键字/表名/列名/ORDER BY/LIMIT 等不可参数化位置）才算注入；作为数据值被参数化传入不算

## 边界

SQL / NoSQL / LDAP / XPath 注入归本域。

以下类型**绝对不要**输出 finding（见 [domain-boundaries.md](../domain-boundaries.md)）：

- 命令注入 / 模板注入 / 表达式语言注入 → `rce`
- 文件路径拼接导致遍历 → `file`
- 业务流程绕过（跳过校验、状态字段可控）→ `business`

越界线索写进 COVERAGE，例如："在 service/auth.go:233 发现命令拼接，建议 rce 域关注"。

字段 schema、定级原则、POC 要求见 [finding-schema.md](../finding-schema.md)。
