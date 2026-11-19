# php-proxy-app 项目分析文档

> 分析日期：2026-11-19　|　项目版本：基于 php-proxy 库 5.2.0　|　分支：dev / master

---

## 1. 项目概述

本项目是 [athlon1600/php-proxy-app](https://github.com/Athlon1600/php-proxy-app)，一个**可直接部署到 Web 服务器公共目录的 Web 代理应用（Project，而非 Library）**。它的作用是：用户在本站输入目标网址，本站代替用户向目标服务器发起 HTTP 请求，并将响应（HTML/资源/流媒体）回传给用户，期间对页面内的链接、静态资源、Cookie 等做"代理化改写"，使其继续经由本代理访问。

- **技术栈**：PHP 7.4 / 8.x + cURL 扩展 + Composer
- **核心依赖**：`athlon1600/php-proxy: ^5.0.1`（全部代理逻辑都在这个库里）
- **典型用途**：网页反代/翻墙类 Web Proxy（如 demo 站 unblockvideos.com）

---

## 2. 目录结构

```
php-proxy-app/
├── index.php               # 唯一入口文件（前端控制器），所有请求从这里开始
├── config.php              # 应用配置文件，加载进全局 Config 类
├── composer.json / .lock   # 依赖声明（仅 php-proxy 一个核心包）
├── Makefile                # 本地开发命令：make serve 启动内置服务器
├── setup.txt               # 初始化脚本：composer create-project 后自动生成随机 app_key 写入 config.php
├── README.md / LICENSE     # 官方说明 / MIT 许可
├── .github/workflows/
│   └── build.yml           # CI：多 PHP 版本(7.4~8.5)安装依赖 + php -l 语法检查
├── plugins/                # 【自定义插件目录】
│   ├── TestPlugin.php      # 插件开发示例骨架（四个生命周期钩子）
│   └── UrlFormPlugin.php   # 在代理页 HTML 顶部注入地址栏表单
├── templates/              # 视图模板（README 说明：本应叫 views）
│   ├── main.php            # 首页/错误页（中央地址输入框）
│   └── url_form.php        # 注入到代理页面顶部的悬浮地址栏
└── vendor/                 # Composer 依赖
    ├── autoload.php
    └── athlon1600/php-proxy/src/   # ★ 核心库，代理引擎
        ├── Proxy.php               # 引擎主体：cURL 转发 + 事件分发器
        ├── Config.php              # 静态配置存取（get/set/load）
        ├── helpers.php             # 全局函数：URL 加解密、proxify_url、模板渲染等
        ├── Html.php
        ├── Redis.php
        ├── Event/ProxyEvent.php    # 事件对象（ArrayAccess 容器）
        ├── Http/
        │   ├── Request.php         # 请求封装（createFromGlobals）
        │   ├── Response.php        # 响应封装（headers + content + send）
        │   └── ParamStore.php      # GET/POST/参数容器
        └── Plugin/
            ├── AbstractPlugin.php      # 插件基类：自动把 onXxx 注册为监听器
            ├── HeaderRewritePlugin.php # 内置：改写响应头（CSP/CORS 等）
            ├── StreamPlugin.php        # 内置：大文件/流媒体边收边发
            ├── CookiePlugin.php        # 内置：Cookie 转发改写
            ├── ProxifyPlugin.php       # 内置：改写 HTML 内的 URL（a/img/form...）
            └── BlockListPlugin.php     # 内置：黑名单拦截
```

**分层一目了然**：应用层只有 `index.php + config.php + plugins/ + templates/` 四个东西；其余全部逻辑在 `vendor/` 的 php-proxy 库中。日常开发主要就是**写插件**。

---

## 3. 代码结构（核心组件职责）

| 组件 | 位置 | 职责 |
|---|---|---|
| `index.php` | 应用层 | 入口路由：区分 首页 / 表单提交 / 代理请求 三种情况；装配插件；调用引擎 |
| `Config` | 库 | 静态键值配置容器，`Config::load()` 合并 config.php 返回的数组 |
| `Proxy` | 库 | 引擎核心：持有 request/response，配置 cURL，**同时自身就是事件分发器** |
| `Request` / `Response` | 库 | HTTP 报文封装；`Request::createFromGlobals()` 从 PHP 超全局变量构建 |
| `ProxyEvent` | 库 | 事件载体，可用 `$event['request']` / `$event['response']` 数组式访问 |
| `AbstractPlugin` | 库 | 插件基类：通过反射把 `onBeforeRequest` 等方法自动挂到对应事件上 |
| `helpers.php` | 库 | 关键函数：`url_encrypt/decrypt`、`proxify_url`、`rel2abs`、`render_template` |

### 3.1 URL 加密机制（本项目的特色点）

代理 URL 形如 `index.php?q=<加密串>`，加密方式由 `url_mode` 配置决定（config.php 第 15~23 行）：

| url_mode | 行为 |
|---|---|
| 0 | 不编码，直接 `?q=http://yahoo.com`（可被外链） |
| 1 | 仅 Base64（URL-Safe），可被外链 |
| 2 | Base64 + 按 `md5(app_key + 客户端IP)` 派生的密钥做字符旋转加密 → URL 仅对该 IP 有效，**当前使用此模式** |
| 3 | 密钥再混入 `session_id()`，会话结束 URL 即失效（需开启 session） |

加密本身是 `str_rot_pass()`：逐字符按密钥 ASCII 值做加减位移，再 Base64-URL 编码（helpers.php 第 66~171 行）。

---

## 4. 程序流程

### 4.1 一次完整代理请求的处理链路

```
浏览器请求 index.php?q=<加密URL>
        │
        ▼
┌─ index.php 启动阶段 ─────────────────────────────┐
│ 1. 检查 cURL 扩展                                 │
│ 2. Config::load('./config.php')                   │
│    Config::load('./custom_config.php')  ← 可选覆盖│
│ 3. 校验 app_key 非空                              │
│ 4. 按 url_mode 派生 encryption_key（IP/会话绑定） │
│ 5. session_write_close() 及早释放会话锁           │
└──────────────────────────────────────────────────┘
        │
        ▼
┌─ 路由分支 ───────────────────────────────────────┐
│ POST 了 url？ → 302 跳转到 proxify_url(url)       │
│ 没有 ?q？    → 渲染首页 main.php（或 index_redirect 跳走）
│ 否则        → 继续 ↓                              │
└──────────────────────────────────────────────────┘
        │
        ▼
┌─ 插件装配 ───────────────────────────────────────┐
│ 遍历 config['plugins']，依次：                    │
│  · 先在 ./plugins/ 找同名文件并 require           │
│  · 按类名顺序探测：插件名 / XxxPlugin / 库内类    │
│  · $proxy->addSubscriber(new $class())            │
│ 当前顺序：HeaderRewrite → Stream → Cookie →       │
│           Proxify → UrlForm                       │
└──────────────────────────────────────────────────┘
        │
        ▼
┌─ 转发阶段（Proxy::forward）──────────────────────┐
│ 1. Request::createFromGlobals() 构建请求对象      │
│    并清空 GET 参数（剥离 ?q=）                    │
│ 2. 触发事件 request.before_send                   │
│    （插件可改 URL / 附加头 / 甚至短路）           │
│ 3. 设置 cURL：                                    │
│    · 超时 10s 连接 / 不限总时长                   │
│    · 关闭 SSL 校验、关闭自动重定向（自己处理）    │
│    · header_callback / write_callback 回调       │
│ 4. curl_exec 发起真实请求                         │
│    · 头回调：解析状态码与响应头，头结束触发        │
│      request.sent 事件（onHeadersReceived）       │
│    · 写回调：每块数据触发 curl.callback.write     │
│      （onCurlWrite）；StreamPlugin 在此直接       │
│      echo 边收边发（流式时脚本不再返回）          │
│ 5. 非流式：缓冲区内容写入 response                │
│ 6. 触发 request.complete（onCompleted）           │
│    · ProxifyPlugin 改写 HTML 内所有 URL          │
│    · CookiePlugin 改写 Cookie                    │
│    · UrlFormPlugin 注入顶部地址栏                │
└──────────────────────────────────────────────────┘
        │
        ▼
index.php 补 CORS 响应头 → $response->send() 回传浏览器
        │
        ▼
异常？→ error_redirect 跳转 或 渲染 main.php 显示错误信息
```

### 4.2 插件生命周期（事件 ↔ 钩子对照）

| 事件名（库内触发） | 插件钩子 | 时机 |
|---|---|---|
| `request.before_send` | `onBeforeRequest` | cURL 请求发出前，最后修改请求的机会 |
| `request.sent` | `onHeadersReceived` | 响应头刚收完，发送给用户前的最后修改机会 |
| `curl.callback.write` | `onCurlWrite` | 响应体逐块到达时（流式场景唯一钩子） |
| `request.complete` | `onCompleted` | 完整响应就绪（仅非流式响应触发） |

`AbstractPlugin` 会自动扫描子类的 `onXxx` 方法并注册为对应事件监听器，所以**写一个插件 = 继承 AbstractPlugin + 覆写需要的钩子 + 在 config.php 的 plugins 数组中登记**。

---

## 5. 配置说明（config.php 关键项）

| 配置项 | 当前值 | 说明 |
|---|---|---|
| `app_key` | `bc24116c8d09` | 应用唯一标识，也是加密密钥的种子，**不可为空** |
| `encryption_key` | `''` | 运行时按 url_mode 自动派生，无需手填 |
| `url_mode` | `2` | URL 按 IP 绑定加密（见 3.1 节） |
| `session_enable` | `false` | 关闭 PHP session（性能 + 避免缓存头）；url_mode=3 时必须开启 |
| `expose_php` | `true` | 保留 X-Powered-By 头 |
| `plugins` | 5 个 | 加载顺序即数组顺序；前两个（HeaderRewrite/Stream）官方要求不可禁用 |
| `curl` | 空 | 可附加任意 CURLOPT_*（如 CURLOPT_PROXY 上游代理） |
| `error_redirect` / `index_redirect` | 注释 | 可选：出错/首页重定向，支持 `{error_msg}` 占位符 |

另有一个可选的 `custom_config.php` 会被第二次 `Config::load()` 合并，适合部署脚本写入环境差异化配置而不污染 config.php。

---

## 6. 运行与部署

```bash
# 本地开发（Makefile）
make serve          # 等价于 php -S localhost:9000

# 手动安装（正式部署，直接放进 Web 根目录）
composer create-project athlon1600/php-proxy-app:"^4.0" /var/www/

# Docker
docker run -d -p 8080:80 -p 8443:443 dwaaan/php-proxy
```

- **CI**（.github/workflows/build.yml）：push 时在 PHP 7.4/8.0/8.2/8.4/8.5 矩阵上执行 `composer install` + `php -l` 语法检查（无功能测试）。
- **升级方式**：应用层极少变动，主要靠 `composer update` 更新 php-proxy 库。

---

## 7. 值得注意的问题与风险点（接手必读）

1. **vendor 目录被本地改动**：[Proxy.php](file:///e:/workspace/php-proxy-app/vendor/athlon1600/php-proxy/src/Proxy.php#L192) 的 `forward()` 末尾新增了一行 `echo ($this->response->getContent());`，而 [index.php](file:///e:/workspace/php-proxy-app/index.php#L119) 之后还会调用 `$response->send()`。**这会导致非流式响应内容被输出两次**（且改动位于 composer 管理的 vendor 内，`composer update/install` 后会被覆盖丢失）。如果这行是你需要的临时补丁，建议改为通过插件或应用层实现；如果不是，应删掉。
2. **SSL 校验被全局关闭**（Proxy.php 中 `CURLOPT_SSL_VERIFYPEER/HOST => false`），代理到 HTTPS 站点时不验证证书。
3. **`app_key` 已提交进仓库**（config.php），且 setup.txt 生成的密钥会被覆盖；仓库若公开，此密钥即泄露，建议改为部署时注入（custom_config.php 正是为此准备的通道）。
4. **CORS 全放开**（index.php 第 113~116 行 `Access-Control-Allow-Origin: *`），代理站可被任意第三方页面跨域调用，容易被滥用刷流量。
5. **无功能测试**：CI 只有语法检查；`vendor/.../tests/` 中仅有库自带的 TestCase 骨架。
6. **Web 代理的固有限制**（README 明确说明）：JS 重度/动态渲染复杂的站点（如部分 SPA）无法正常工作，此类需求应使用真正的代理服务器而非本应用。
7. **Git 状态**：仓库存在 master 与 dev 两个分支，origin 远端已配置。

---

## 8. 二次开发指引（最常见的改动场景）

- **新增一个插件**：参考 [plugins/TestPlugin.php](file:///e:/workspace/php-proxy-app/plugins/TestPlugin.php) 的骨架，继承 `AbstractPlugin`，覆写 `onBeforeRequest / onHeadersReceived / onCurlWrite / onCompleted` 中的钩子，把类名加进 config.php 的 `plugins` 数组即可（文件放 ./plugins/ 会自动 require）。
- **改首页/错误页**：编辑 `templates/main.php`；改代理页顶部地址栏：`templates/url_form.php`。
- **调整代理行为**：优先写插件挂事件；引擎级改动（cURL 参数）可走 config.php 的 `curl` 数组，无需改库。
- **环境差异配置**：写入 `custom_config.php`（会自动覆盖 config.php 中的同名键）。
