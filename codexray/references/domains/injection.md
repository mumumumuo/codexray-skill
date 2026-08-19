# 注入类漏洞深度审计

> **风险等级**：🟠 严重 — 可导致数据泄露/篡改
> **域 ID**：`injection`

## 审计范围

- **SQL 注入**：所有数据库操作点（原生 SQL、字符串拼接 SQL、动态 ORM 查询）
- **NoSQL 注入**：MongoDB、Redis、Elasticsearch 等 NoSQL 数据库的查询拼接
- **LDAP 注入**：LDAP filter 拼接
- **XPath 注入**：XPath 查询拼接
- **XXE（XML 外部实体注入）**：解析器允许外部实体或 DTD，详见下方专项

## 任务要求

- 枚举所有数据库查询操作点
- **区分参数化查询（安全）与字符串拼接（危险）**：参数化查询不输出 finding，仅在拼接场景标记为漏洞
- 识别所有 NoSQL / LDAP / XPath 查询拼接位置
- 排查所有 XML 解析点的 XXE（见专项）
- 为每个漏洞点找到所有可触发的 URL 端点
- 生成每个端点的完整 HTTP 攻击数据包

## 全面性要求

- **方法级追踪**：DAO 层方法存在漏洞，追踪所有 Service / Controller 调用路径
- **端点完全枚举**：所有触发同一漏洞的端点必须全部列出
- **参数全覆盖**：同一端点不同的注入参数分别报告

## XXE 专项（XML 外部实体注入）

XXE 与其他注入的形态不同：**危险不在于拼接，而在于解析器配置**。用户提交的 XML 本身可以完全合法，只要解析器允许 DTD 与外部实体，`<!ENTITY xxe SYSTEM "file:///etc/passwd">` 就会被展开。

### 找解析点

XML 入口远不止 `Content-Type: application/xml`。逐个排查：

- 请求体直接是 XML / SOAP / SAML 断言 / RSS / WebDAV
- **文件上传**：`.xml`、`.svg`（SVG 就是 XML）、`.xsd`、`.xsl`、`.wsdl`、`.gpx`、`.kml`
- **OOXML 办公文档**：`.xlsx` / `.docx` / `.pptx` / `.ods`（见下方专项）
- 配置导入、数据迁移、第三方回调（支付/物流异步通知常用 XML）
- URL 参数里传 XML 字符串（少见但存在）

### 各语言的解析器与安全默认值

**默认是否安全差异极大，这是判定的核心依据，不能一概而论：**

| 语言/库 | 危险 sink | 默认安全性 |
|---|---|---|
| Java `DocumentBuilderFactory` / `SAXParserFactory` / `XMLInputFactory` / `TransformerFactory` / `SchemaFactory` / dom4j `SAXReader` / JDOM `SAXBuilder` / JAXB `Unmarshaller` | 全部 | **默认不安全** —— 需显式 `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`。注意 `setExpandEntityReferences(false)` **不足以**防护 |
| Python `lxml.etree` | `parse` / `fromstring` | 默认 `no_network=True` 但 `resolve_entities=True` —— **可读本地文件**。安全写法：`XMLParser(resolve_entities=False, load_dtd=False, no_network=True)` |
| Python `xml.dom.minidom` / `xml.sax` / `xmltodict` | `parse` / `parseString` | 默认解析外部实体，**不安全** |
| Python `xml.etree.ElementTree` | `parse` / `fromstring` | 不支持外部实体 → **classic XXE 不成立**，但仍可 billion laughs（DoS） |
| PHP `simplexml_load_string` / `DOMDocument::loadXML` / `xml_parse` | 传入 `LIBXML_NOENT`/`LIBXML_DTDLOAD` 时 | libxml2 ≥ 2.9 默认关闭外部实体；**显式传危险 flag 才成立** |
| .NET `XmlDocument` / `XmlTextReader` / `XmlReaderSettings` | `DtdProcessing = Parse`、`XmlResolver` 非 null | .NET Framework ≥ 4.5.2 默认 `XmlResolver = null`（安全）；旧版本或显式改回则危险 |
| Node `libxmljs` / `xmldom` | `noent: true` | 需显式开启才危险；`fast-xml-parser` 不做实体展开 |
| **Go `encoding/xml`** | — | **根本不支持外部实体 → Go 项目的 classic XXE 一律误报**，不要输出 finding |

修复形态统一：禁用 DTD / 关闭外部实体 / Python 换 `defusedxml`，而不是「过滤输入」。

### Excel / OOXML 导入 XXE（重点，最易漏）

`.xlsx` 本质是 **ZIP 包套 XML**（`xl/workbook.xml`、`xl/sharedStrings.xml`、`[Content_Types].xml`、`docProps/core.xml`）。攻击者上传一个**完全合法的 xlsx**，只在内部某个 XML 部件里插入 DTD 实体声明。

**为什么上传侧校验全部失效**：扩展名是 `.xlsx`、MIME 是 `application/vnd.openxmlformats-...`、ZIP 魔数 `PK\x03\x04` 正确、文件能被 Excel 正常打开 —— file 域那套「白名单/文件头/MIME」清单一条都拦不住。**漏洞在解析环节，不在上传环节。**

危险 sink：

- Java：`WorkbookFactory.create` / `XSSFWorkbook` / EasyExcel `EasyExcel.read` / `POIFSFileSystem`（POI 旧版本有 XXE CVE，也要看依赖文件里的版本）
- Python：`openpyxl.load_workbook` / `pandas.read_excel` / `python-docx`
- 同族格式：`.docx`、`.pptx`、`.ods`
- **例外**：`.xls`（HSSF）是 OLE2 二进制格式，不是 XML → 不是这个路径

这类漏洞通常**盲打**：导入结果不回显内容，需用外带 DTD（攻击者服务器）或 DNS 回连确认。POC 要给出构造 xlsx 的 Python 脚本（往合法 xlsx 的 XML 部件注入实体后重新打包），而不是单个 HTTP 请求 —— 参见 finding-schema 的脚本类 POC 要求。

### 定级

按实际影响，不按「XXE 一律高危」：读到 `/etc/passwd`、云元数据、源码或配置密钥 → `high` 及以上；仅能 SSRF 探测内网端口 → 按可达资产判断；只能 billion laughs 打 DoS → 通常 `medium`。

## 越权分析要求

- **认证状态**：明确标注「鉴权」或「未授权」
- **权限要求**：触发漏洞所需的最小权限

## 判定纪律

拼接不等于可注入。输出 finding 前确认三件事：拼进 SQL 的值来自用户输入、中途没有被参数化或强类型转换（如已 `strconv.Atoi`）、没有白名单枚举校验。只满足「看起来是拼接」就报，是最常见的误报来源。

ORM 的 `Where("id = ?", x)` 安全，`Where(fmt.Sprintf(...))` 危险 —— 区分清楚再报。

XXE 同理：**先确认该语言/库的默认值，再看代码有没有显式加固**。在 Go 项目里报 classic XXE、或在 PHP 里没看到危险 flag 就报 XXE，都是误报。

## 边界

SQL / NoSQL / LDAP / XPath 注入 **与 XXE** 归本域。命令注入 / 模板注入 / 表达式语言注入 → `rce`，本域不涉及；追踪注入链路时若发现命令执行函数，在覆盖度小结提及即可，不输出 finding。

XXE 的**后果**跨域，但 finding 只在本域出一条 —— 根因是解析器配置这一处：

- XXE 读任意文件 → 不在 `file` 重复报
- XXE 打内网 / 云元数据 → 不在 `business` 以 SSRF 重复报
- XXE billion laughs → 不在 `business` 以 DoS 重复报
- 例外：XML 解析后的对象进入反序列化链（如 `XMLDecoder`）→ 那是 `rce`

完整映射见 [domain-boundaries.md](../domain-boundaries.md)。

字段 schema、定级原则、POC 要求见 [finding-schema.md](../finding-schema.md)。
