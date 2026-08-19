# 配置与组件安全审计

> **风险等级**：🔵 低中危 — 可导致信息泄露/间接利用
> **域 ID**：`config`

## 审计范围

- **敏感信息泄露**：硬编码密码 / API Key / Token / 私钥 / 数据库连接串
- **CORS 配置错误**：`Access-Control-Allow-Origin: *` + `Allow-Credentials: true` 等危险组合
- **调试模式开启**：DEBUG=True / Actuator endpoints / pprof / phpinfo
- **不安全 TLS 配置**：`InsecureSkipVerify: true` / 自签证书 / 弱加密套件
- **Cookie 安全**：缺失 `Secure` / `HttpOnly` / `SameSite`
- **第三方组件漏洞**（CVE）：依赖文件中的过期版本
- **错误信息泄露**：堆栈跟踪 / SQL 错误 / 路径泄露暴露给用户

## 任务要求

- 检查所有配置文件：
  - `config.yaml` / `application.properties` / `.env` / `settings.py` / `config.go` 等
  - 数据库密码、API 密钥、第三方服务凭证
  - **特别留意空字符串、常见默认弱密码（admin/123456/password 等）**
- 分析依赖组件安全状况：
  - `go.mod` / `package.json` / `pom.xml` / `requirements.txt` / `Gemfile.lock`
  - 已废弃 / 高危 CVE 的库（如 `dgrijalva/jwt-go`、`log4j 2.x`、`fastjson 1.x`）
  - **查到 CVE 后必须配合代码定位调用点和触发接口**
- 识别信息泄露点：
  - 调试接口（pprof、actuator、swagger 在生产暴露）
  - 错误响应是否回显堆栈 / SQL 错误
  - 注释中的敏感信息（账号密码、内部 URL）
- CORS 中间件配置：`Allow-Origin: *` + `Allow-Credentials: true` 是高危组合
- TLS 客户端配置：`InsecureSkipVerify` 全局禁用证书验证
- 提供组件升级建议（升级到具体版本号）

## 输出要求

- 第三方组件 CVE 必须包含：
  - CVE 编号
  - 影响版本
  - 当前版本
  - 调用点（file:line）
  - 触发接口（如能定位）
  - 升级建议版本

## 判定纪律

- `password: changeme` / `secret: your-secret-key` / `xxx` / `test` 等明显占位符 → 不是漏洞
- `.env.example` / `config.example.yaml` 中的示例值 → 不是漏洞
- 测试文件中的 `test_password = "test123"` → 不是漏洞
- 密钥长度极短（<8 字符）且无实际服务使用证据 → 可能误报，写 suspected 或不报
- 框架版本自带的已知 CVE 但代码未调用受影响功能 → 降级说明，不机械套高危

## 边界

- 「业务逻辑层的 SSRF」→ `business`
- 「TLS 客户端 InsecureSkipVerify 配置」→ 本域
- 「认证机制本身的密钥配置（如 JWT 密钥为空、硬编码密钥）」→ `auth`（本域不审计认证相关密钥）
- 「数据库密码、第三方 API Key、Token 等非认证类凭证」→ 本域

完整映射见 [domain-boundaries.md](../domain-boundaries.md)。越界线索写进 COVERAGE。

字段 schema、定级原则、POC 要求见 [finding-schema.md](../finding-schema.md)。
