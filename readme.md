# APISIX UI

## 1. 项目概况

APISIX UI 是一个纯 HTML 单文件管理界面，面向 APISIX 3.X 的 Admin API。CSS、JavaScript 和部署帮助均内嵌在文件中，不需要打包、前端框架、CDN 或自建后端。

```text
浏览器中的 HTML + JavaScript
  ├─ localStorage：连接信息、当前环境和 TOKEN
  └─ fetch + X-API-KEY → APISIX Admin API（可经管理代理）
                               └─ APISIX 配置存储
```

本项目不实现服务端用户体系、RBAC、审计、密钥托管或反向代理。所有管理操作最终由 APISIX 校验并执行。

### 1.1 文件与入口

| 文件                   | 说明                                       |
| ---------------------- | ------------------------------------------ |
| `index.html`           | 应用入口，包含页面、样式、逻辑和部署帮助。 |
| `docs/研发维护手册.md` | 本文。                                     |

当前目录未发现测试框架、依赖清单、构建脚本或 Trellis 项目配置。不要假设两份 HTML 会自动保持同步。

### 1.2 运行前提

- 现代浏览器，支持 `fetch`、`dialog`、`localStorage`、`TextEncoder`、`TextDecoder`、`crypto.getRandomValues` 等 API。
- 管理端需允许浏览器访问；跨域访问须配置 CORS。直接使用 `file://` 时，浏览器对存储、来源和剪贴板的处理可能不同，推荐使用 HTTPS 静态站点。
- 列表实现要求 Admin API 的 v3 分页格式：`total` 和 `list`。实际 APISIX 版本、资源类型和空数据返回格式应通过集成测试确认。
- 页面可以从 OSS、自定义域名、Nginx 等静态服务提供，也可由 APISIX 代理静态文件。

## 2. 功能清单

| 菜单       | Admin API 资源    | 创建                              | 查看 / 更新                                 | 删除 | 其他                               |
| ---------- | ----------------- | --------------------------------- | ------------------------------------------- | ---- | ---------------------------------- |
| 服务       | `services`        | JSON，POST，由 APISIX 分配 ID     | JSON，PUT 整体替换                          | 是   | 不提供复制                         |
| 路由       | `routes`          | JSON，ID 可选；未填时 POST        | JSON，PUT 整体替换                          | 是   | 复制创建；状态上下线；按标签键筛选 |
| 上游       | `upstreams`       | JSON，ID 可选；未填时 POST        | JSON，PUT 整体替换                          | 是   | 复制创建                           |
| SSLs       | `ssls`            | 证书和私钥上传，计算 ID 后 PUT    | 不提供                                      | 是   | 展示 SNI、证书到期时间             |
| 全局规则   | `global_rules`    | JSON，用户填写 ID                 | JSON，PUT 整体替换                          | 是   | 展示插件名称                       |
| 插件元数据 | `plugin_metadata` | JSON，填写插件名称作为标识        | JSON，PUT 整体替换                          | 是   | 标识是插件名                       |
| 插件配置   | `plugin_configs`  | JSON，页面生成 UUID v4 后 PUT     | JSON，PUT 整体替换                          | 是   | 不提供复制                         |
| Proto      | `protos`          | 上传文件，页面生成 UUID v4 后 PUT | 只读 JSON；通过独立表单更新名称、描述及文件 | 是   | 不提供复制                         |

除 SSL 外，各类资源使用统一列表、分页和关键词搜索。删除前弹窗确认，**不传 `force=true`**；资源仍被引用等错误由 APISIX 返回并展示。

页面还提供环境管理、名称悬停显示 ID、复制 ID、右上角部署帮助和官方文档入口。表格内容在容器内部滚动，表头固定。

## 3. 环境与鉴权

### 3.1 生命周期

1. 页面启动时从 `localStorage` 的 `apisix-manager-environments-v2` 读取连接列表和当前连接。
2. 如果无有效连接，隐藏主界面并强制打开新增连接窗口；帮助入口仍可使用。
3. 每个连接包含名称、标准化 API 前缀和 TOKEN。新增或修改时以 `GET /routes?page=1&page_size=10` 验证，并通过 `normalizePage` 校验响应。
4. 验证成功后保存并切换到该环境；左侧可切换连接、打开设置，以及编辑或删除连接。删除最后一个连接会返回强制配置状态。
5. 页面不设置 TOKEN 过期时间。旧版存储格式不会自动迁移。

验证成功只证明对应路由列表读取和认证可用，**不保证其他资源的权限**。

### 3.2 URL 规范化

`normalizeApi` 接受 HTTP / HTTPS 地址，拒绝 URL 中的用户名、密码、查询参数与片段，移除路径末尾 `/`。路径不以 `/apisix/admin` 结尾时自动追加。

| 用户填写                                 | 实际 API 前缀                                       |
| ---------------------------------------- | --------------------------------------------------- |
| `https://admin.example.com`              | `https://admin.example.com/apisix/admin`            |
| `https://admin.example.com/apisix/admin` | 不重复追加                                          |
| `https://mgt.example.com/apisix-admin`   | `https://mgt.example.com/apisix-admin/apisix/admin` |

例如代理规则剥离 `/apisix-admin` 时，页面请求 `/apisix-admin/apisix/admin/routes`，上游收到 `/apisix/admin/routes`。连接配置和已有代理路径依赖此行为，修改前应准备兼容及迁移方案。

### 3.3 风险

- TOKEN 以明文形式保存在浏览器本地存储，无自动过期；同源脚本及拥有设备访问权限的人可能读取。不要在公共或不可信设备上使用，泄露后应在 APISIX 服务端轮换。
- 请求统一发送 `X-API-KEY`；有 JSON 请求体时额外发送 `Content-Type: application/json`。页面不支持配置第二个代理认证凭据或自定义认证头。
- 当前管理 API 为 HTTP 时，主界面顶部显示 TOKEN 明文传输警告。HTTPS 页面请求 HTTP API 还可能被浏览器混合内容策略阻止。
- 环境切换后 `request` 会在响应返回时检查环境是否仍相同；这不能撤销已经发出的写请求。

## 4. 列表、搜索和标签

### 4.1 分页

默认每页 15 条，可选 10、15、20、50、100、500。普通无关键词列表请求：

```http
GET /apisix/admin/{resource}?page=1&page_size=15
X-API-KEY: <admin-key>
```

`normalizePage` 要求非负整数 `total`。`list` 为数组时必须不超过请求页大小；`total=0` 或请求超出末页时，也接受缺失、`null` 或空对象形式的列表。它不会将缺失 `total` 的响应直接当作空列表。

删除后如果当前页超出末页，`loadList` 会回退到最后一页重新请求。异步加载通过 `state.revision` 丢弃过期响应，避免切换资源后旧数据覆盖新界面。

### 4.2 关键词搜索

点击“搜索”或按回车后，关键词转小写，在浏览器中匹配资源 ID 或 `value` 的 JSON 字符串。实现通过 `fetchAll` 以每页 500 条逐页获取**所有符合服务端条件的资源**，再在浏览器过滤和分页；每次搜索、翻页或更改页大小都可能重新扫描。大数据量、配置快速变化或大对象可能导致延迟、内存和请求量增加，也不提供跨页快照一致性。

### 4.3 路由标签

- 输入框支持输入或选择标签**键**；请求时传递 `label=<key>`，按 Admin API 的服务端键过滤，不按标签值精确匹配。
- 候选键从当前已加载的路由结果中提取，按键去重并排序；翻页累积，不会为了构造候选而主动读取所有路由。切换资源或环境会清空候选。若同时使用关键词，匹配流程仍会扫描全部服务端结果。
- 列表标签显示键和值，点击复制键名；空标签显示 `—`。候选仅覆盖当前会话访问过的结果，不能代表全部标签。

## 5. 创建、更新及状态变更

### 5.1 JSON 编辑

`openEditor` 获取单项最新配置，移除顶层 `id`、`create_time`、`update_time` 后递归排序对象键并显示；数组顺序不改变。编辑器使用原生 textarea 与同步滚动的高亮层，不加载第三方库。

保存时要求合法 JSON 对象，再次移除上述顶层字段，使用 PUT 整体替换。嵌套同名字段保留；服务端负责业务 schema 校验。路由和上游的“复制”会重新获取资源并填充创建窗口，不直接提交。

### 5.2 ID 与并发

- 路由、上游：填 ID 时先查询再 PUT；不填则 POST，由 APISIX 分配。
- 服务：POST，由 APISIX 分配。
- 全局规则：用户指定 ID；插件元数据：用户指定插件名。
- 插件配置、Proto：通过 `crypto.getRandomValues` 生成 UUID v4，然后先 GET，确认 404 后 PUT。
- SSL：按第 6 节生成 ID，同样先查再 PUT。

**检查不存在后 PUT 并非原子创建。** 并发客户端可能在检查与写入之间创建相同 ID，从而被 PUT 覆盖。更新同样是整体替换，可能覆盖其他客户端同时做出的更改。后续如需严格防覆盖，应由服务端提供原子条件写入或版本控制，不能只依靠浏览器预检查。

### 5.3 路由状态

`status` 缺省或为 `1` 时显示在线，`0` 显示已下线；其他值显示未知且禁用切换。上下线需确认，分别发送：

```http
PATCH /apisix/admin/routes/{id}
Content-Type: application/json

{"status":0}
```

`1` 表示上线。下线保留配置，只使该路由不参与匹配；其他路由仍可能匹配同一请求。成功后刷新当前列表；失败展示 API 错误。

## 6. SSL 实现与 ID 契约

### 6.1 上传

支持粘贴 PEM 内容或选择证书和私钥文件。SNI 可手动输入（换行、英文逗号或中文逗号分隔）；留空时从**第一张 PEM 证书**的 X.509 SAN 扩展中提取 DNS 项。没有可解析的 DNS SAN 时提示手填。页面校验域名形态和 PEM 标记，证书、私钥和域名匹配等最终由 APISIX 校验。

提交内容形如：

```json
{"cert":"<PEM 证书>","key":"<PEM 私钥>","snis":["*.example.com","api.example.com"]}
```

列表不主动展示私钥；不支持 SSL 更新或查看完整 JSON。上传请求会将私钥发送到 Admin API，必须使用可信 HTTPS 通道和可信网关。

### 6.2 SSL ID 计算

`normalizeSnis` 与 `md5` 实现：

1. 每个 SNI 去除首尾空格，转小写。
2. 去重，按 JavaScript 默认字典序排序。
3. 使用英文逗号连接，不添加空格。
4. 对结果的 UTF-8 字节计算 MD5，输出 32 位小写十六进制。

```text
输入：B.example.com、a.example.com、a.example.com
规范化原文：a.example.com,b.example.com
SSL ID：3ce4d4169322ffc5d7764da33ff3aa0b
```

通配符 `*.example.com` 作为普通字符串参与排序。手输和 SAN 提取最终遵循相同规则；MD5 仅用于确定性标识，**不是安全签名或密码哈希**。已有 ID 时拒绝主动上传，但仍存在第 5.2 节的并发窗口。

### 6.3 到期时间

内置 DER 解析器读取证书的 `notAfter`（UTC 或 GeneralizedTime）；多证书时取最早到期时间。已过期、30 天内到期及正常状态以颜色区分；内容缺失或解析失败显示“无法解析”。解析依赖服务端返回 PEM；Secret 引用、特殊时间格式或异常证书可能无法显示。

## 7. Proto

- 创建：必须选择 `.proto` 或 `.pd`，可填名称和描述；随机生成 UUID，查重后 PUT。
- `.proto` 按严格 UTF-8 文本解码；`.pd` 按二进制读取并编码为 Base64。这里的 `.pd` 指 protobuf descriptor set，不自动运行 `protoc`。
- 文件不得为空、原始文件不得超过 1 MiB，编码后的内容 UTF-8 字节也不得超过 1 MiB；文件扩展名不匹配或编码错误时在浏览器拒绝。最终 Proto 语义由 APISIX 校验。
- 查看 JSON 为只读；点击“更新 Proto”可修改名称、描述或替换文件，不选新文件保留原 `content`，同时保留原有其他业务字段。更新使用原 ID 的 PUT。

APISIX 或前置代理可能另有限制（例如完整请求体大小），应在目标环境实测。更改已被路由或服务引用的 Proto 可能影响运行中的 gRPC 转码。

## 8. 部署与安全

页面右上角 `?` 提供内嵌部署说明和可复制示例。部署帮助和本文都需要随实现同步维护。

### 8.1 静态页面

- 推荐通过 HTTPS 提供 HTML，并返回 `text/html; charset=utf-8`。使用 OSS 时检查自定义域名、证书、对象读取权限、Host、SNI、上游 TLS 校验及缓存失效。
- APISIX 代理静态 HTML 时应尽量限定域名、路径和 `GET` / `HEAD`；开启缓存需考虑发布后旧文件的 TTL。
- 不要把管理 TOKEN 打包在 HTML、OSS 对象、日志或仓库中。

### 8.2 Admin API 代理

业务端口可将受保护的管理路由转发到 Admin API；页面帮助中的示例使用 `/apisix-admin/*`，转发到同网络命名空间的 `127.0.0.1:9180`。CORS 应限制真实页面 Origin，并允许 `Content-Type,X-API-KEY` 及 `GET,POST,PUT,PATCH,DELETE,OPTIONS`。

**管理代理仍是高权限入口。** CORS 不是鉴权；应采用 HTTPS、强 Admin Key、可信网络或 VPN、访问审计及最小权限，不缓存管理 API 响应。`ip-restriction` 的判断依赖可信的客户端真实 IP 配置，不能盲目信任外部请求头；预检成功也不代表实际请求通过 IP 和鉴权检查。

帮助中的配置示例保持 Admin API 监听 `127.0.0.1:9180`，并采用 `allow_admin: [0.0.0.0/0]` 以兼容可能传递客户端真实 IP 的代理场景。**该配置不再按 IPv4 来源限制 Admin API**：任何能直连同网络命名空间中 9180 的进程都可能绕过代理路由的 IP 限制，必须通过端口不对外发布、容器 / 主机隔离、可信进程和 Admin Key 保护。初始化路由需进入容器执行；分离部署、IPv6 和不同 real-IP 设置必须按实际拓扑重新评估和验证。

## 9. 代码结构与扩展指南

`index.html` 从上至下依次为 CSS、页面与弹窗、帮助内容、JavaScript。关键函数：

| 职责             | 函数 / 状态                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------------- |
| 资源注册与菜单   | `RESOURCES`、`selectResource`                                                                  |
| 环境存储及连接   | `readEnvironments`、`normalizeApi`、`persistEnvironments`、`lock`                              |
| API 请求和错误   | `request`、`showError`、`notify`                                                               |
| 列表和分页       | `normalizePage`、`fetchPage`、`fetchAll`、`loadList`、`renderList`                             |
| 标签选择         | `labelKeys`、`renderLabelOptions`、`runSearch`                                                 |
| JSON 操作        | `businessConfig`、`sortedJson`、`openEditor`、`openCreate`、`highlightJson`                    |
| 新资源与冲突检查 | `randomGuid`、`ensureAbsent`                                                                   |
| Proto            | `openProtoForm`、`readProtoFile`、`protoForm` 提交事件                                         |
| 路由状态         | `openRouteStatus`、`routeStatusForm` 提交事件                                                  |
| 证书解析和 ID    | `derNode`、`certificateFields`、`certificateExpiry`、`extractDnsNames`、`normalizeSnis`、`md5` |
| 帮助与复制       | `copyToClipboard`、`.help-launch`、`#helpDialog`                                               |

新增资源时至少检查：资源路径及服务端支持的方法、ID 生成方式、schema、分页格式、引用与删除行为、显示列、排序、列表和写入错误、环境切换并发，以及是否需要特殊的文件或敏感字段处理。显示外部数据时继续使用 `textContent` / DOM 节点，避免将 API 内容拼进 HTML。

## 10. 验证与排障

当前仓库没有持久化的自动化测试套件；历史验证包含浏览器与临时模拟 API 测试，但不能替代真实 APISIX 3.X 集成测试。修改后至少覆盖：

1. 首次配置、验证失败、多个环境切换、删除最后一个环境及 HTTP 警告。
2. 各资源的空列表、非空列表、多页、切换页大小、关键词及路由标签键查询；确认服务端 `total/list` 格式与真实实例一致。
3. 创建、更新、删除、引用冲突和 401/403/404/500；核对请求路径、方法、请求头、JSON body 和当前环境。
4. 路由上下线的确认取消、成功、失败及其他业务字段不变。
5. JSON 排序、格式化、非法 JSON、顶层审计字段移除及嵌套字段保留；路由 / 上游复制不立即写入。
6. SSL 手工 SNI、SAN 提取、通配符、ID 已存在、实际证书到期和异常证书；同时验证私钥不出现在列表和日志中。
7. Proto UTF-8、无效 UTF-8、`.pd` Base64、空文件、大小边界、上传失败、更新不选文件保留内容。
8. HTTPS / CORS 预检和实际方法、代理路径、受限 IP、未授权来源、管理 API 权限及部署缓存刷新。
9. 直接打开和静态服务器打开的浏览器兼容性，特别是剪贴板、本地存储和模态弹窗。

常见问题：

| 现象                      | 排查方向                                                                                             |
| ------------------------- | ---------------------------------------------------------------------------------------------------- |
| 验证失败或网络错误        | API 地址与代理路径、HTTPS、DNS、CORS、浏览器混合内容、Admin Key、权限和 IP 白名单。                  |
| 列表提示缺少 v3 总数      | 查看实际资源接口响应，确认 `admin_api_version`、服务版本及代理是否改写；不要盲目将未知响应当空列表。 |
| 搜索慢                    | 当前关键词搜索逐页获取全部匹配资源后在浏览器过滤。                                                   |
| 标签下拉缺少选项          | 下拉只累积已加载结果的键，可手动输入未加载的键。                                                     |
| 上传 SSL 无法自动提取 SNI | 检查第一张 PEM 的 DNS SAN；可手动填写。                                                              |
| SSL 到期显示无法解析      | 检查列表返回是否包含 PEM、证书是否为受支持的 X.509 编码。                                            |
| 删除失败                  | 查看 APISIX 引用检查与权限错误；页面不会强制删除。                                                   |

## 11. 已知限制与维护风险

- 当前单文件架构无模块隔离、自动构建或持久化测试；功能增长时要关注 UI 状态和事件处理的耦合，但重构不得改变接口行为。
- 各资源分页实现依赖 APISIX 的 v3 响应；不同资源或代理可能存在差异，特别是空资源集合。缺少真实环境证据时不要假定任意异常响应表示空数据。
- 环境 TOKEN 长期保存在浏览器本地；同源 XSS、设备共享或被植入脚本均可能泄密。
- JSON PUT 和 Proto 更新是整体替换，不是并发安全的合并；随机 ID 查重也不是原子创建。
- 证书的 DER 解析、MD5 和 JSON 高亮为自包含实现，不等同于完整的 X.509、加密或编辑器库；任何修改应加入真实证书、边界向量及浏览器测试。
