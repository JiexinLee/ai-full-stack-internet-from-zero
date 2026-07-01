# 第二章：HTTP 到底在“聊”什么？📮

## 先把大脑从“术语地狱”里拽出来

你打开网页时，浏览器和服务器其实在做一件很朴素的事：

> 浏览器：我想要这个资源  
> 服务器：行，给你（或者不给你）

这个“聊天规则”，就是 HTTP。

这一章继续保持“大白话 + 讲故事”的风格：不背概念，不抄百科。我们站在开发者视角，带你看清：

- 你在 DevTools 里看到的 Request/Response Headers 到底对应啥
- Cookie / Session / JWT / OAuth 这些“登录相关的妖怪”到底谁在干什么
- 缓存、压缩、跨域这些“明明写了代码却不生效”的坑到底是怎么回事
- HTTP/1.1、HTTP/2、HTTP/3、HTTPS、TLS 到底解决了什么现实问题

---

## 本章按三个问题来拆解

### 1) 为什么会出现 HTTP？🤔

很久很久以前（也没那么久），互联网里想干一件事很麻烦：

- 两台机器要通信，得先建立连接（TCP）
- 建完连接之后，得约定“怎么说话”
- 否则你发一句“我要首页”，对面听成“我要删库”，那就很尴尬 😅

于是需要一个统一的、跨语言、跨平台的“说话协议”：

- 浏览器能说
- Node / Java / Go / Python 也能说
- Nginx、CDN、负载均衡也能听懂

HTTP 就是那套规则：请求（Request）和响应（Response）。

你可以把 HTTP 理解成“快递面单”：

- 你寄快递：填面单（方法、路径、头信息、包裹内容）
- 快递站点：看面单决定怎么转运（代理、网关）
- 收件人：按面单处理并回执（状态码、响应头、响应体）

### 2) 为什么选择 HTTP？有没有别的选择？🧰

HTTP 之所以成为 Web 的主角，是因为它非常适合“开放世界”：

- 简单：一问一答（Request/Response）
- 可扩展：靠 Header 不断加能力（缓存、认证、压缩、跨域……）
- 可分层：中间可以插很多“中间人”（CDN、代理、网关）而不破坏基本语义
- 兼容强：基础设施支持广泛

当然也有别的选择：

- WebSocket：适合长连接双向通信（聊天、实时推送）
- gRPC：更像“强类型 RPC”，内部服务调用很香
- MQTT：物联网场景常见

但“浏览器打开网页”这件事，HTTP 仍然是最通用、最兼容的选择。

### 3) HTTP 到底怎么工作？开发者怎么用？🔍

接下来进入主线剧情：一条请求从浏览器出发，到服务器返回，过程中每个关键点都拆开讲。

---

## 一张手绘图先把全景装进脑子 🧠

```text
你在地址栏敲下 URL
      |
      v
浏览器：组装 HTTP Request（方法/路径/headers/body）
      |
      v
（可能经过）CDN / 负载均衡 / 反向代理
      |
      v
应用服务器：解析 Request -> 业务处理 -> 组装 Response
      |
      v
浏览器：拿到 Response（status/headers/body）-> 渲染
```

---

## Part 1：HTTP 请求到底长什么样？🧾

### 请求 = 起始行 + Headers + Body

把一个 HTTP 请求想象成“三明治”：

- 第一层：请求行（方法、路径、协议版本）
- 第二层：Headers（面单信息）
- 第三层：Body（包裹内容，可有可无）

一个典型的 HTTP/1.1 请求长这样：

```http
GET /products/123?from=home HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml
Accept-Language: zh-CN,zh;q=0.9
Accept-Encoding: gzip, br
Connection: keep-alive
Cookie: session_id=abc123
```

这几行分别在说什么？

- `GET /products/123?from=home HTTP/1.1`：我要用 GET 方法拿这个路径的资源
- `Host`：我要访问哪个站点（HTTP/1.1 必备，因为同一个 IP 上可能挂很多站）
- `User-Agent`：我是谁（浏览器/版本）
- `Accept`：我希望拿到什么类型的内容
- `Accept-Encoding`：我支持哪些压缩格式（gzip/br）
- `Connection: keep-alive`：别聊一句就挂电话，咱保持连接（后面讲）
- `Cookie`：我带着“门禁卡”来了（后面讲）

如果你发的是 POST，并且带 JSON body，会像这样：

```http
POST /api/login HTTP/1.1
Host: example.com
Content-Type: application/json
Accept: application/json
Content-Length: 45

{"email":"a@b.com","password":"123456"}
```

你在 DevTools 里看到的 Request Headers / Request Payload，基本就是这些东西的“浏览器 UI 版本”。

---

## Part 2：HTTP 响应到底长什么样？📦

### 响应 = 状态行 + Headers + Body

服务器回你一句“结果如何”：

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
Content-Encoding: gzip
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax

{"id":123,"name":"Keyboard","price":199}
```

你可以把它理解成“快递回执”：

- 状态码：送到了？送丢了？地址不对？你没权限？
- 响应头：这包裹是什么、能缓存多久、压缩了吗、要不要给你发张新门禁卡（Set-Cookie）
- 响应体：真正的数据内容

---

## Part 3：方法（Method）不是随便写的 🧠

你会经常见到：

- `GET`：获取资源（通常不带 body）
- `POST`：创建资源/提交动作（常带 body）
- `PUT`：整体替换资源（幂等）
- `PATCH`：部分更新资源
- `DELETE`：删除资源（幂等）
- `HEAD`：只要 headers，不要 body
- `OPTIONS`：问问“你支不支持这些跨域规则”（CORS 预检常用）

开发者最重要的两个直觉：

1. GET 应该是安全的：不应该产生副作用（别用 GET 去扣款）
2. 幂等性：同样请求重复执行多次，结果是否一致

```text
GET /orders/123      重复 100 次，都只是查询 -> 结果一致 ✅
DELETE /orders/123   重复 100 次，删除第一次后后面都无事发生 -> 结果一致 ✅
POST /orders         重复 100 次，可能创建 100 个订单 -> 结果不一致 ❌
```

---

## Part 4：状态码（Status Code）是服务器的“表情包” 😄😡

你不需要背完，但需要记住常用的：

### 2xx：成功

- `200 OK`：成功且有响应
- `201 Created`：创建成功（常见于 POST）
- `204 No Content`：成功但没内容（比如删除成功）

### 3xx：跳转/缓存相关

- `301 Moved Permanently`：永久重定向
- `302 Found`：临时重定向
- `304 Not Modified`：没变，走缓存（后面详讲）

### 4xx：你（客户端）的问题

- `400 Bad Request`：参数不合法
- `401 Unauthorized`：你没带身份/身份无效
- `403 Forbidden`：你是谁我知道，但你没权限 😅
- `404 Not Found`：没这个资源
- `409 Conflict`：冲突（比如重复创建）
- `429 Too Many Requests`：别刷了，限流了

### 5xx：我（服务器）的问题

- `500 Internal Server Error`：服务器炸了
- `502 Bad Gateway`：网关/代理从上游拿不到好结果
- `503 Service Unavailable`：服务暂不可用（维护/过载）

---

## Part 5：Headers 是“魔法发生的地方” 🪄

Headers 是 HTTP 的扩展槽位：很多能力不是写死在协议里，而是靠 Header 逐渐长出来的。

下面用开发者最常见的一批 Header，讲清楚它们“到底对应什么”。

### 1) Content-Type：你发的 body 是什么？

常见值：

- `application/json`：JSON
- `application/x-www-form-urlencoded`：表单
- `multipart/form-data`：文件上传（带 boundary）
- `text/plain`：纯文本

### 2) Accept：你想要什么？

浏览器会告诉服务器：我希望得到 HTML 还是 JSON，还是图片？

```http
Accept: text/html
Accept: application/json
Accept: image/avif,image/webp,*/*
```

### 3) Authorization：你是谁？（常给 API 用）

最常见的 Bearer Token：

```http
Authorization: Bearer <token>
```

这个 token 可能是 JWT，也可能是别的系统签发的令牌。

### 4) Cookie / Set-Cookie：浏览器的“门禁卡系统”

服务器发：

```http
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
```

浏览器保存后，之后请求会自动带上：

```http
Cookie: session_id=abc123
```

几个关键属性一定要懂：

- `HttpOnly`：JS 读不到，降低 XSS 偷 cookie 风险
- `Secure`：只在 HTTPS 里发送
- `SameSite`：限制跨站发送 cookie（抗 CSRF 的重要手段）

### 5) Cache-Control / ETag：缓存的“控制面板”

- `Cache-Control`：能不能缓存？缓存多久？
- `ETag`：内容版本号，用来做协商缓存

这两套后面会单独用图讲。

### 6) Origin / Referer：我从哪来？（跨域相关）

- `Origin`：跨域时最重要的“来源站点”标记
- `Referer`：从哪个页面跳过来的（历史拼写原因少了个 r）

### 7) Connection: keep-alive：别聊一句就挂断

这个会直接影响性能（后面讲 Keep-Alive）。

---

## Part 6：用 Node 把 HTTP 这件事“掀开衣服看清楚” 🧑‍🔧

你说“最好结合开发”——那我们直接上手写一个最小可运行的 HTTP server，专门用来观察：

- 浏览器到底发了哪些 headers
- body 怎么读取
- 服务器怎么设置缓存、cookie、CORS

### 6.1 最小 HTTP 服务：打印请求头 + 读取 body

```js
import http from "node:http";

function readBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    req.on("data", (chunk) => chunks.push(chunk));
    req.on("end", () => resolve(Buffer.concat(chunks)));
    req.on("error", reject);
  });
}

const server = http.createServer(async (req, res) => {
  const bodyBuffer = await readBody(req);
  const bodyText = bodyBuffer.toString("utf8");

  const response = {
    method: req.method,
    url: req.url,
    headers: req.headers,
    body: bodyText,
  };

  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json; charset=utf-8");
  res.end(JSON.stringify(response, null, 2));
});

server.listen(3000);
```

现在你可以用 curl 直接打：

```bash
curl -i http://localhost:3000/hello \
  -H "X-Demo: 123" \
  -H "Content-Type: application/json" \
  -d '{"a":1,"b":2}'
```

你会在响应里看到服务器收到的 headers 和 body，这会非常“祛魅”。

### 6.2 让浏览器“记住你”：Set-Cookie

在上面的 server 里加一句：

```js
res.setHeader("Set-Cookie", "session_id=abc123; HttpOnly; SameSite=Lax");
```

然后用浏览器访问 `http://localhost:3000/hello`，再看下一次请求 headers，你会看到 `Cookie: session_id=abc123` 自动出现。

---

## Part 7：Keep-Alive：为什么不每次都重新连接？☎️

如果每次 HTTP 请求都要重新建立 TCP 连接：

- TCP 三次握手开销大
- TLS 握手更贵（HTTPS 的话）
- 频繁建立/关闭连接让服务器压力变大

所以 HTTP/1.1 倾向于保持连接：

```http
Connection: keep-alive
```

你可以把它理解成：

> 不是每句都“打电话->说一句->挂断”，而是“接通后多聊几句再挂”。📞

---

## Part 8：HTTP 缓存：你以为你在请求，其实你在复用 🧊

网页“变快”的大招之一就是缓存。

但缓存又容易让人抓狂：

> “我代码改了，为什么浏览器还是旧的？” 😭

### 8.1 强缓存：Cache-Control

服务器说：

```http
Cache-Control: max-age=60
```

意思是：这玩意 60 秒内你别来问我了，你自己用缓存。

强缓存的逻辑可以画成这样：

```text
浏览器要请求资源
      |
      v
本地有缓存吗？
  |        |
  | 没有   | 有
  v        v
去服务器   看看缓存是否还在有效期内
              |            |
              | 过期了      | 没过期
              v            v
          去服务器       直接用本地缓存（不发请求）
```

### 8.2 协商缓存：ETag + If-None-Match

如果你想更精细一点：浏览器可以带着“我上次看到的版本号”问服务器。

浏览器请求：

```http
GET /app.js HTTP/1.1
If-None-Match: "v1"
```

服务器对比后：

- 没变：返回 `304 Not Modified`（没有 body）
- 变了：返回 `200 OK` + 新内容 + 新 ETag

协商缓存“问一下再决定”的图：

```text
浏览器：我有缓存，但我不确定是不是最新
      |
      v
带上 If-None-Match 去问服务器
      |
      v
服务器：比对 ETag
   |               |
   | 没变           | 变了
   v               v
304（不用传内容）   200 + 新内容 + 新 ETag
```

### 8.3 开发实践：为什么你改了代码却没生效？

最常见的原因：

- 你给静态资源设置了很长的 `max-age`
- 但资源 URL 没变（比如一直叫 `/app.js`）

工程上一般的解法：

> 给静态资源加 hash 文件名，比如 `/app.8f3a2c.js`，内容变了文件名也变。  
> 然后对带 hash 的文件给超长缓存，对不带 hash 的入口文件给短缓存。

---

## Part 9：压缩：gzip / br 让你少跑几趟 🧳

浏览器会说：

```http
Accept-Encoding: gzip, br
```

服务器如果压缩了，会回：

```http
Content-Encoding: gzip
```

这就是“我把包裹压扁了再寄给你”，传输更快。

Node 里可以用 `node:zlib` 做 gzip：

```js
import http from "node:http";
import zlib from "node:zlib";

http
  .createServer((req, res) => {
    const payload = JSON.stringify({ ok: true, time: Date.now() });
    const acceptEncoding = String(req.headers["accept-encoding"] || "");
    if (acceptEncoding.includes("gzip")) {
      const gz = zlib.gzipSync(payload);
      res.statusCode = 200;
      res.setHeader("Content-Type", "application/json; charset=utf-8");
      res.setHeader("Content-Encoding", "gzip");
      res.end(gz);
      return;
    }
    res.statusCode = 200;
    res.setHeader("Content-Type", "application/json; charset=utf-8");
    res.end(payload);
  })
  .listen(3001);
```

---

## Part 10：跨域 CORS：为什么浏览器要管这么严？🧱

你写前端时最常遇到的红字之一：

> “Blocked by CORS policy …” 😡

先说一句大白话：

> CORS 不是服务器限制你，是浏览器限制你。

服务器其实很愿意给你数据，但浏览器说：

> “你这个网页来自 A 域名，凭什么偷摸去读 B 域名的响应？” 👮

### 10.1 简单请求 vs 预检请求（OPTIONS）

有些跨域请求，浏览器会先发一个 `OPTIONS` 过去问：

> “你允许我用 POST 吗？我能带这些 headers 吗？”

这就是预检（Preflight）。

手绘图：

```text
浏览器想跨域发请求
      |
      v
判断是否需要预检
  |              |
  | 不需要        | 需要
  v              v
直接发真实请求     先发 OPTIONS 预检
                    |
                    v
           服务器返回允许规则
                    |
                    v
           再发真实请求
```

### 10.2 服务器怎么正确响应 CORS？

最常见的允许规则：

```http
Access-Control-Allow-Origin: https://your-frontend.com
Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

如果你还要带 cookie（跨站登录常见），还需要：

```http
Access-Control-Allow-Credentials: true
```

并且 `Access-Control-Allow-Origin` 不能是 `*`，必须是明确域名。

### 10.3 Node 里最小 CORS 处理（包含预检）

```js
import http from "node:http";

function setCors(res, origin) {
  res.setHeader("Access-Control-Allow-Origin", origin);
  res.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
  res.setHeader("Access-Control-Allow-Credentials", "true");
}

http
  .createServer((req, res) => {
    const origin = String(req.headers.origin || "");
    if (origin) setCors(res, origin);

    if (req.method === "OPTIONS") {
      res.statusCode = 204;
      res.end();
      return;
    }

    res.statusCode = 200;
    res.setHeader("Content-Type", "application/json; charset=utf-8");
    res.end(JSON.stringify({ ok: true }));
  })
  .listen(3002);
```

---

## Part 11：登录那一坨：Cookie / Session / JWT / OAuth 🧟

这块最容易把人绕晕，我们先用一句话把它们分清：

- Cookie：浏览器保存并自动携带的小纸条
- Session：服务器保存的“你是谁”的档案
- JWT：把身份信息打包成 token（更偏无状态）
- OAuth：第三方登录/授权体系（你不用把密码交给第三方）

### 11.1 Cookie：浏览器保存的“小纸条”

Cookie 的本质就是：浏览器替你存一段数据，并在符合规则时自动带上。

### 11.2 Session：服务器保存的“你的档案”

典型模式：

- 浏览器保存 `session_id`（cookie）
- 服务器用 `session_id` 查到“你是谁”的会话信息（session）

手绘图：

```text
浏览器首次登录
   |
   v
服务器创建 session（存在内存/Redis/DB）
   |
   v
服务器通过 Set-Cookie 发 session_id 给浏览器
   |
   v
之后请求浏览器自动带 Cookie: session_id=...
   |
   v
服务器查 session_id -> 得到用户身份
```

优点：实现简单、可控  
缺点：服务端要存状态（扩容/多机要共享 session，常放 Redis）

### 11.3 JWT：把“身份信息”打包带走（无状态）

JWT 常见承载方式：

```http
Authorization: Bearer <jwt>
```

思路：

- 服务器签发 token（带用户信息 + 签名）
- 客户端之后每次请求都带上 token
- 服务器验证签名即可，不一定需要查 session 存储

优点：更“无状态”，扩容舒服  
缺点：撤销难、过期策略要设计、别把敏感数据塞 token 里

### 11.4 OAuth：我不想给你密码，我给你“授权”

你用“微信/Google 登录”时常见：

- 你不把密码给第三方网站
- 你去授权页确认
- 第三方拿到授权后再拿 token

这是一种“把登录交给更可信的身份提供方”的模式。

---

## Part 12：HTTP/1.1、HTTP/2、HTTP/3：它们在和谁打架？🥊

这部分按“问题 -> 方案”讲。

### 12.1 HTTP/1.1：最基础协议（够用但有痛点）

HTTP/1.1 非常成功，但它有性能痛点：

- 早期一个连接同一时间处理能力有限（队头阻塞更明显）
- 浏览器为了并发，只能对同域名开多个 TCP 连接
- header 冗余，每次都带重复字段

### 12.2 HTTP/2：多路复用 + 头部压缩

HTTP/2 的核心升级可以记成两句话：

- 一个 TCP 连接里同时跑很多条“流”（多路复用）
- header 不要每次都重复背诵（头部压缩）

手绘对比：

```text
HTTP/1.1：
连接 1：请求 A -> 响应 A -> 请求 B -> 响应 B ...

HTTP/2：
一个连接里：
流 1：A 请求/响应
流 3：B 请求/响应
流 5：C 请求/响应
它们可以交错传输
```

### 12.3 HTTP/3：QUIC，换底盘

HTTP/2 解决了应用层并发，但仍跑在 TCP 上：

- TCP 丢包可能影响整个连接体验

HTTP/3 跑在 QUIC（基于 UDP）上，用不同方式解决传输层问题：

- 更快的连接建立
- 更好的丢包恢复体验（按流隔离影响）

你可以先把它理解成：

> HTTP/3 不只是换版本号，是换“底盘”。🚗

---

## Part 13：HTTPS / TLS：为什么安全？SSL 为什么淘汰？🔒

### 13.1 HTTPS 是什么？

HTTPS = HTTP + TLS。

也就是：

> HTTP 的内容不再裸奔，而是先加密再传。🧥

### 13.2 TLS 解决什么问题？

主要三个：

- 机密性：别人抓包看不懂内容
- 完整性：别人篡改内容会被发现
- 身份认证：你连的是“真的服务器”，不是冒牌货

### 13.3 SSL 为什么淘汰？

简单说：SSL 的旧版本存在安全问题，现代都用 TLS。

你在工程里看到“SSL 证书”通常是历史叫法，真正协议是 TLS。

### 13.4 TLS 握手你先看懂这张图就够了

```text
浏览器 <------> 服务器
   |
   | 1) 浏览器：我支持这些加密套件
   |
   | 2) 服务器：我选这个 + 给你证书
   |
   | 3) 浏览器：我验证证书（CA 链）
   |
   | 4) 双方：协商出会话密钥
   |
   v
之后 HTTP 内容用会话密钥加密传输
```

工程实践里最常见的 TLS 问题：

- 证书过期
- 域名不匹配
- 中间证书链没配全

---

## 本章总结：HTTP 其实没那么神秘 ✅

用一句话总结 HTTP：

> HTTP 就是一套“请求/响应”的说话规则；Headers 是扩展槽位；  
> 缓存、压缩、认证、跨域、版本演进，都是围绕“更快、更稳、更安全、更好用”。

你现在应该已经能做到：

- 看懂一份 Request/Response 的 headers 大概在说什么
- 知道 cookie/session/jwt/oauth 在解决什么不同问题
- 遇到 CORS、缓存不生效、304/ETag、压缩这些坑能定位方向
- 理解 HTTP/2/3 和 HTTPS/TLS 的升级动机

---

## 下一章你可以继续追问的问题 🚀

1. 浏览器发请求前后，DNS/TCP/TLS 到底是怎么串起来的？
2. HTTP 的连接复用、连接池、代理层到底怎么影响性能？
3. 如果要做一个生产可用的 API 服务，哪些 headers 是必须设计好的？
