---
name: web-login-js-burp-audit
description: 当用户提供 Web URL、登录页、H5/前端静态包、本地前端目录或备份包地址，并要求使用 Burp 访问分析、落地前端 JS、审计敏感信息/接口、做最小化验证并在当前工作目录输出 Markdown 报告时使用。重点适用于登录框渗透测试、前端源码泄露、硬编码 key、未授权接口、CORS、source map、备份文件泄露、验证码/短信/注册/重置流程线索梳理等中文渗透测试工作流。
---

# Web 登录页 JS Burp 审计

## 目标

用户只给一个 URL 或本地前端目录时，默认按这个流程开始：

- 通过 Burp 访问目标，解析并落地 HTML、JS、CSS、公开配置文件；
- 从前端资源中提取源码路径、接口路径、`funcCode`、硬编码 key、token、`appId`、云存储配置、加密材料、测试账号、备份/source map 线索；
- 结合登录框测试思路，对能低影响验证的点做最小化验证；
- 在当前工作目录生成 `audit_summary.md`。

默认边界：不爆破、不撞库、不轰炸短信、不批量注册、不提交真实凭据、不调用删除/审核/重置/踢人/改权限/上传/新增/修改类接口。优先使用 `GET`、`HEAD`、`OPTIONS`、空参只读 `POST`、`pageSize: 1` 只读 `POST`。

## 标准流程

1. **创建审计目录**
   - 将 host/path 规范成 `{host}-audit` 或 `{host}-{path}-audit`。
   - 已有落地文件时先复用；用户要求刷新时再重新抓取。

2. **Burp 入口访问**
   - 用 Burp 请求用户给定 URL，并补充常见入口：`/`、`/manager/`、`/h5/`、`/portal/front/index.html`、登录 hash 路由原路径。
   - 保存入口 HTML。
   - 记录状态码、服务端、标题、跳转、Cookie、CSP、X-Frame、HSTS、CORS 相关响应头。

3. **落地前端资源**
   - 从 HTML 中解析 `<script src>`、`<link href>`、`modulepreload`、动态配置、legacy chunk。
   - 下载主业务 JS、manifest/runtime chunk、公开配置文件和必要 CSS。
   - 大型站点优先落地主业务包和 manifest；vendor 包只抓与 `crypto`、`config`、`axios/request`、`router`、`store`、`auth` 相关的部分。

4. **静态提取**
   - 提取并统计：
     - 绝对 URL、后端基址、网关地址；
     - API 路径、`funcCode`、路由名、源码路径、组件路径；
     - 登录、注册、重置、短信、验证码、token、用户、角色、菜单、文件、上传、下载、订单、支付、后台等关键词；
     - `appId`、`clientId`、`frontKey`、`secret`、`privateKey`、`publicKey`、`accessKey`、`token`、`password`、`SM2/SM3/SM4/AES/RSA`、`sign`、`timestamp`、`nonce`；
     - `.env`、`.map`、`.bak`、`.zip`、`/assets/`、`/static/`、OSS/COS/OBS 配置。
   - 如果 bundle 是超长单行，不要整行打印；写本地只读提取脚本输出摘要、计数、去重列表和命中上下文。

## 登录框最小化测试矩阵

先从 JS 推导流程，再决定是否碰接口。把“登录页面登录框渗透测试思路”中的点按下面规则执行。

### 默认可以做

- **验证码回显/注册码回显**：抓取验证码、注册校验、找回密码相关响应，检查 JSON、HTML、JS、localStorage 初始化值中是否直接出现 code、captcha、smsCode、registerCode。只读观察，不爆破。
- **验证码是否前端校验**：检查 JS 是否在前端比较验证码、注册码、短信码；如果服务端接口可只读请求，则只做一次空参/缺字段请求观察错误顺序。
- **登录框持久/测试账号**：检查 HTML、JS、localStorage、默认表单值、注释、mock 数据中是否写死账号、手机号、密码、`admin`、`test`、`demo`。
- **错误信息泄露**：只使用明显虚构账号做一次低影响登录失败请求，观察是否区分“用户不存在/密码错误/验证码错误/账号锁定”。不要枚举真实用户。
- **未授权访问/API 读取**：从 JS 中提取菜单、用户信息、字典、列表、文件展示、配置读取接口，使用空参或 `pageSize: 1` 验证是否无认证返回数据。
- **URL 拼接/跳转**：检查 `redirect`、`returnUrl`、`callback`、`target`、`url`、`uid`、`userId`、`phone`、`customerId`、`userNo` 等参数；只用 `GET/HEAD` 或无害外链 `https://audit.invalid` 验证是否可控跳转。
- **敏感信息泄露**：检查登录失败响应、配置接口、公开 JS 是否泄露手机号、身份证、token、路径、堆栈、密钥、云存储地址。
- **前端权限控制**：提取菜单/按钮权限、路由守卫、`role`、`permission`、`authBtnList`、`checkScope`、`tokenFlag` 等逻辑；再对只读接口做无 token 验证。
- **source map/备份文件**：对 `{js}.map`、`.env`、`.env.production`、`index.html.bak`、`{path}.zip`、`assets/`、`static/` 做少量 `HEAD/GET`，确认是否真实文件或 SPA fallback。
- **CORS/Host 低影响检查**：对高价值接口做 `OPTIONS`，使用 `Origin: https://audit.invalid`；Host 头只做 `HEAD/GET` 观察重定向或绝对 URL 生成。
- **硬编码加密材料利用性**：只做本地解密公开配置、复现签名/加密格式、读取非敏感字典枚举。不要调用重置密钥、设置密钥、授权变更接口。
- **文件 show/download**：对前端硬编码 showId/downloadId 使用 `HEAD` 或 `Range: bytes=0-0`，避免完整下载。

### 需要用户提供测试账号/手机号后才做

- **短信发送限制**：最多一次发送到用户明确提供的自有手机号/邮箱；只观察是否返回验证码、倒计时、频率限制字段。默认不触发。
- **验证码复用/关系校验**：需要用户提供自有测试账号、测试手机号或测试邮箱；只做一组受控请求，不做循环。
- **密码重置流程**：需要用户确认测试账号归属；只验证流程字段、关系绑定、是否回显验证码，不真正修改密码。
- **水平/垂直越权**：需要至少两个授权测试账号或用户提供的合法 token；默认只从前端和未授权接口做线索判断。

### 默认不做，只记录风险线索

- 验证码爆破、用户名爆破、密码爆破、弱口令字典、批量注册、短信轰炸。
- 用户名覆盖、注册覆盖、账号锁定 DoS。
- SQL 注入、XSS、RCE/nday 的主动 payload 测试，除非用户明确授权并限定目标；默认只记录框架版本、输入点、反射点、报错线索。
- 返回包修改绕过：只记录“疑似依赖前端响应判断”，不把本地改包当成漏洞结论。

## Burp 最小验证规则

- source map：`HEAD/GET {js}.map`，确认响应是否为真实 JSON/source map；若返回首页 HTML，记为 fallback。
- 备份暴露：只探测少量明显候选，不递归扫描。
- CORS：记录 `Access-Control-Allow-Origin`、`Access-Control-Allow-Credentials`、`Access-Control-Allow-Headers`、`Access-Control-Allow-Methods`。
- 未授权读取：只测查询、列表、字典、用户信息、菜单、文件展示类接口；写入类接口只做静态记录。
- 参数格式：遇到“参数格式错误/缺少所属应用/缺 token”时，优先用前端硬编码的 `appId`、公开 header、`func-code` 复现前端格式，再做一次只读请求。
- 返回量控制：列表接口统一带 `pageNum: 1`、`pageSize: 1`；文件接口用 `Range: bytes=0-0`。
- 证据保留：报告写请求方法、路径、关键 header/body、状态码、业务 code、是否返回数据；不贴大段响应。

## 利用性分级

- **已确认**：低影响请求无认证返回数据、真实 source map 下载成功、备份包暴露、文件读取成功、硬编码 key 可解密本地/运行时配置。
- **有利用条件**：JS 暴露接口/key，但实测还需要 token、正确加密体、业务参数或合法账号。
- **未确认**：接口返回 401/403/业务鉴权错误、参数格式错误、超时、或只是 SPA fallback。
- **仅线索**：只在 JS/HTML 中发现接口、账号字段、弱逻辑或历史漏洞关键词，未做动态验证。

## 报告落地

输出 `{audit-folder}/audit_summary.md`，必须包含：

- 目标和已落地文件；
- 一句话漏洞定性；
- 静态发现和数量统计；
- 敏感值留痕，按用户习惯可保留完整值，但最终聊天回复默认脱敏；
- 代表性接口、源码路径、业务模块；
- 登录框最小化验证结果：验证码、短信、注册/重置、错误信息、跳转、未授权、CORS、source map、备份；
- Burp 最小验证请求和响应结论；
- 可利用性判断和验证边界。

最终回复保持短：报告路径、最强确认发现、哪些没有验证。

## 报告话术模板

确认前端/备份泄露：

```markdown
一句话概括：{target} 属于前端静态资源/备份/源码信息泄露，包含 {接口路径、源码路径、硬编码 appId/key/token/测试账号等}，并经最小验证确认 {未授权接口/文件读取/CORS/source map/备份文件}。
```

只确认静态泄露：

```markdown
当前仅确认前端静态资源泄露和接口/密钥线索暴露，未证明可绕过 token 或访问受保护业务数据。
```

确认未授权接口：

```markdown
当前已确认硬编码值可用于未登录调用部分只读接口，属于前端信息泄露叠加未授权读取风险。
```

确认备份包泄露：

```markdown
该问题属于前端备份文件泄露，备份包中包含可还原的前端源码/构建产物、接口地址、业务路径和硬编码配置，应下线备份文件并轮换暴露密钥。
```

登录框专项：

```markdown
登录框侧重点：已从前端 JS 提取 {登录/短信/注册/重置/token/菜单} 流程，最小验证确认 {验证码回显/错误信息泄露/未授权读取/跳转可控/未确认}，未进行爆破、短信轰炸或真实账号提交。
```

## 安全边界

- 不爆破用户名、密码、验证码或短信码。
- 不触发短信/邮件发送，除非用户明确提供自己的测试手机号/邮箱并要求一次受控请求。
- 不调用疑似新增、修改、删除、审核、重置密钥、踢用户、改角色、上传文件、改认证配置的接口。
- 不确定接口性质时，先用 `OPTIONS`、`HEAD`、空 `GET` 或 `pageSize: 1` 的 `POST`。
- 聊天中默认脱敏 key；本地报告可按用户要求保留完整值。
