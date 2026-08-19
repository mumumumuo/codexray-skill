# codexray-kill — 多 Agent 并行源码安全审计

把一个 agentic 编码助手变成一支 **7 域并行** 的白盒审计小队：Phase 1 架构理解 → Phase 2 七个子 Agent 同时深挖各自攻击面 → Phase 3 复核（误报过滤 / 定级复核 / 攻击链组合），最终交付**一个交互式 HTML 审计报告**。

```
开工（cp 模板建报告骨架）
  └─ Phase 1 架构理解（1 个架构子 agent：技术栈 / 路由 / 中间件）
  └─ Phase 2 分域并行审计（7 个域子 agent，同一条消息一次性派发）
  └─ Phase 3 复核与攻击链（误报过滤 / 去重 / 定级复核 / 缺口审查 / 链组合）
  └─ 收尾（唯一产物：一个 HTML 报告）
```

## 七个审计域

| 域 | 覆盖 |
|---|---|
| `rce` | 命令执行、代码注入、反序列化 RCE、模板注入（SSTI）、表达式语言注入 |
| `injection` | SQL / NoSQL / LDAP / XPath 注入 |
| `file` | 任意文件上传 / 下载 / 读取、路径遍历、zip slip |
| `auth` | 未授权访问、认证绕过、水平 / 垂直越权、注册提权、JWT 密钥、密码存储 |
| `business` | 业务流程绕过、金额篡改、状态机绕过、条件竞争、幂等性缺失、SSRF |
| `xss` | 反射型 / 存储型 / DOM 型 XSS |
| `config` | 硬编码凭证、CORS、TLS、调试模式、组件 CVE、错误信息泄露 |

### Claude Code 及其他 agentic CLI

把本目录放入对应 CLI 的 skill 目录（如 Claude Code 的 `~/.claude/skills/codexray`）。skill 只依赖五项通用能力（读文件 / 检索内容 / 执行 shell / 写改文件 / 启动并行子代理），具备即可运行；无子代理能力时自动降级为串行逐域审计。

## 使用

```
/codexray d <目标路径>        # 直接开扫
/codexray                    # 加载后输出使用说明，用自然语言描述需求
```

示例：

```
/codexray d D:\projects\my-app
/codexray d /srv/web
```

也支持自然语言触发：

> 帮我审计一下 D:\web 这个项目的安全性
> 只看看这个项目有没有 SQL 注入和越权   ← 仅派发 injection + auth 两域，Phase 1 / Phase 3 照常

要求中文报告：漏洞标题、描述、定级理由、修复建议均为中文；代码片段、HTTP 数据包、路径等技术标识符保持原样。
