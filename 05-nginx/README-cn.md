# 第五章：Nginx 到底在忙什么？🧑‍💼

## 先别被这个名字吓到

很多人第一次听到 Nginx，脑子里的反应通常是：

> “这玩意到底是服务器？代理？网关？静态文件工具？SSL 工具？负载均衡器？怎么什么都像它，什么都不是它？” 😵

这个困惑非常正常。

因为 Nginx 在真实工程里，确实经常同时扮演很多角色：

- Web 服务器
- 反向代理
- 负载均衡器
- HTTPS 终止层
- 静态资源服务器
- 缓存层
- 限流层

所以你会感觉它像一个“什么都掺一脚”的家伙。

但如果用一句大白话来总结 Nginx，它其实就是：

> **一个站在流量入口处的、特别擅长转发、分发、缓存、压缩和兜底的交通总指挥。**

这一章我们继续延续前两章的风格：

- 说人话
- 讲故事
- 带一点夸张但不胡扯
- 站在开发者视角
- 配手绘图和配置片段

而且这章会尽量照顾“完全没碰过 Nginx 的人”。

---

## 本章按三个问题来拆解

### 1. 为什么会出现 Nginx？🤔

先看一个最天真的服务：

```text
浏览器 ---> Node.js 应用
```

小项目、学习项目、个人 Demo，这么干完全没问题。

但只要事情稍微复杂一点，你就会马上撞上一堆现实问题：

- 静态资源每次都让应用自己返回，浪费性能
- HTTPS 证书、TLS 配置、重定向，全塞到应用里很麻烦
- 一个域名后面挂多台应用时，谁来分流？
- 某些请求根本不该进应用，比如静态文件、健康检查、恶意请求
- 应用挂了、慢了、重启了，谁来兜底？
- 你想做限流、缓存、gzip 压缩、IP 控制，总不能全写在业务代码里

这时候就很需要一个“站在应用前面”的专业角色。

Nginx 就是来干这个的。

它像什么？

像写字楼的一楼超级前台 + 门卫 + 分诊台 + 收发室：

- 访客先来前台
- 前台先判断你来干嘛
- 有的人直接被带去会议室
- 有的人只拿个文件就走
- 有的人被保安拦下
- 有的人被分流去不同部门

应用服务终于不用在门口一边站岗、一边接电话、一边送快递、一边管门禁了。

### 2. 为什么选择 Nginx？有没有别的选择？🧰

Nginx 火了这么多年，不是因为“大家都在用，所以我也用”。

而是因为它非常适合做入口层：

- 配置驱动，清晰直接
- 静态文件处理强
- 反向代理成熟
- 并发处理能力强
- 社区资料多
- 在 Linux 生产环境里极其常见

当然，Nginx 不是唯一选择：

- Apache：老牌 Web 服务器，生态深厚
- Caddy：自动 HTTPS 体验非常丝滑
- Envoy：更现代、更偏云原生和 service mesh
- Traefik：容器和动态服务发现体验友好

但如果你现在想真正搞懂“反向代理、静态资源、HTTPS、负载均衡”这些互联网入口能力，Nginx 依然是非常经典、非常值得学的一站。

### 3. Nginx 到底怎么工作？开发里怎么用？🔍

这就是这一章的主线。

我们会从这几个角度讲清楚：

- Nginx 到底站在整个请求链路里的哪一层
- 正向代理和反向代理到底差哪儿
- `server`、`location`、`upstream` 到底是什么
- 真实请求是怎么被转发到 Node / Python / Go 应用的
- HTTPS、gzip、缓存、限流怎么配
- 最后给你一些“刚上手最容易踩坑”的经验

---

## 一张图先把 Nginx 的位置记住 🧠

```text
浏览器
  |
  v
[ DNS ]
  |
  v
[ CDN ]   （有时有，有时没有）
  |
  v
[ Nginx ]
  |------> 直接返回静态资源
  |------> 反向代理到 App 1
  |------> 反向代理到 App 2
  |------> 做 HTTPS / gzip / 缓存 / 限流
  |
  v
[ 你的应用服务 ]
```

一句话记忆：

> **Nginx 通常不是“业务主角”，但它常常是“所有流量先见到的第一张脸”。**

---

## Part 1：Nginx 到底是什么？🧾

如果你非要给 Nginx 一个职业身份，我更建议你这样记：

- 从“网页文件角度”看：它是 Web 服务器
- 从“转发请求角度”看：它是反向代理
- 从“把请求分到多台机器”角度看：它是负载均衡器
- 从“站在 HTTPS 门口”角度看：它是 TLS 终止层
- 从“限制流量和缓存”角度看：它是入口治理层

这也是为什么初学者经常觉得它“身份不明”。

其实不是身份不明，是它太能干。

---

## Part 2：正向代理 vs 反向代理，到底差哪儿？🪞

这是学 Nginx 时第一个非常容易混淆的点。

### 2.1 正向代理：代理“客户端”

你可以把正向代理理解成：

> 客户端找了个中间人替自己去访问外部世界。

手绘图：

```text
客户端 ---> 正向代理 ---> 目标网站
```

常见场景：

- 科学上网
- 公司内网统一出网
- 爬虫统一走代理 IP

目标网站通常知道：

> “我看到的是一个代理服务器，不一定是真正的客户端。”

### 2.2 反向代理：代理“服务器”

反向代理则相反：

> 用户以为自己在直接访问网站，实际上先到达了一个站在服务器前面的中间层。

手绘图：

```text
客户端 ---> 反向代理 ---> 应用服务器
```

用户通常并不关心后面到底有几台服务器、什么语言、什么框架。

这就是 Nginx 最经典的角色。

### 2.3 一句话强行记忆

- 正向代理：替“我”去访问别人
- 反向代理：替“别人”来接待我

如果你每次都记混，就反复念三遍：

> 正向代理靠近客户端，反向代理靠近服务端。

---

## Part 3：为什么业务服务前面常常要放一个 Nginx？🚪

假设你直接用 Node.js 启一个服务：

```text
用户 ---> Node.js:3000
```

开始看起来很美好。

但很快你会想做这些事：

- 访问 `example.com` 自动跳到 `https://example.com`
- `/api` 走后端服务
- `/` 和 `/assets` 直接返回前端静态文件
- 图片、CSS、JS 加缓存头
- 大文件开启 gzip
- 某个 IP 别刷太猛
- 应用服务重启时，对外最好不要太丑

这些能力如果都塞到业务代码里，当然也不是不行。

但那会让你的应用变成：

> 一边写业务，一边兼职保安、快递员、门卫、前台和交通警察。😵‍💫

所以工程上通常会变成：

```text
用户 ---> Nginx ---> Node / Python / Go 应用
```

这样分工就清晰很多：

- Nginx 负责入口治理
- 应用负责业务逻辑

---

## Part 4：Nginx 的配置文件到底在说什么？📄

很多人第一次打开 Nginx 配置，眼睛会这样：

> `events {}`  
> `http {}`  
> `server {}`  
> `location / {}`  
> “……我是谁，我在哪，这些大括号到底谁管谁？” 😵

别慌，我们拆开。

### 4.1 先看一个最小结构

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name example.com;

        location / {
            return 200 "hello nginx\n";
        }
    }
}
```

你可以先这样理解层级：

```text
全局配置
  |
  +-- events   -> 跟连接处理相关
  |
  +-- http     -> HTTP 世界的大容器
        |
        +-- server   -> 一个网站/一个虚拟主机
              |
              +-- location -> 某类路径匹配规则
```

### 4.2 这些块分别在干嘛？

#### 全局（main context）

像：

- `worker_processes`
- `error_log`
- `pid`

这类属于 Nginx 整体运行参数。

#### `events`

主要跟连接事件处理有关。

初学阶段你不用把它想得太玄学，先把它理解成：

> “Nginx 怎么管理很多连接”的底层设置区。

#### `http`

这是 HTTP 相关配置的大本营。

里面通常会放：

- `include mime.types`
- `sendfile on`
- `gzip on`
- `upstream`
- `server`

#### `server`

一个 `server` 基本可以理解成：

> “我对一个域名/端口组合的一套接待规则。”

比如：

- 谁监听 80 端口
- 谁处理 `example.com`
- 谁处理 `api.example.com`

#### `location`

这是最常用的匹配规则块。

它决定：

> 某个 URL 路径，最终该怎么处理。

比如：

- `/api/` 转发给后端
- `/assets/` 返回静态文件
- `/` 返回首页

---

## Part 5：先从最简单的静态网站开始 🍞

如果你的前端构建产物在：

```text
/var/www/myapp
```

里面有：

- `index.html`
- `assets/app.js`
- `assets/app.css`

那一个非常经典的 Nginx 配置是：

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/myapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 5.1 这里每一行在干嘛？

- `listen 80;`：监听 80 端口（HTTP）
- `server_name example.com;`：处理这个域名的请求
- `root /var/www/myapp;`：静态文件根目录
- `index index.html;`：默认首页文件
- `try_files $uri $uri/ /index.html;`：先找真实文件，找不到就回退到 `index.html`

### 5.2 为什么 SPA 前端常常要配 `try_files`？

因为 Vue / React 这种前端路由经常长这样：

```text
/dashboard
/profile
/orders/123
```

这些路径在浏览器看来像真实页面路径，但在服务器磁盘上未必真的有这些文件。

如果你不做 `try_files` 回退，Nginx 会直接说：

> “没有 `/orders/123` 这个文件，404 再见。” 😑

而你真正想要的是：

> “虽然这个文件不存在，但把 `index.html` 给浏览器，让前端路由自己接管。”

这就是 `try_files` 在 SPA 里的巨大价值。

手绘图：

```text
浏览器请求 /orders/123
      |
      v
Nginx 先看磁盘上有没有 /orders/123
      |
      | 没有
      v
回退返回 /index.html
      |
      v
前端路由接管页面渲染
```

---

## Part 6：反向代理，Nginx 最经典的主业 🧭

如果你的应用跑在本机 3000 端口：

```text
127.0.0.1:3000
```

你可以让 Nginx 站在前面接待所有用户请求：

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 6.1 这一坨在说什么？

最关键的是：

```nginx
proxy_pass http://127.0.0.1:3000;
```

意思就是：

> 这个请求别在我这里处理了，转发给后面的应用服务。

而下面这些 `proxy_set_header` 非常重要，因为它们是在告诉后端：

- 用户原来访问的 Host 是什么
- 用户真实 IP 是什么
- 请求经过了哪些代理
- 原始协议是 HTTP 还是 HTTPS

这些信息在后端框架里非常常用，比如：

- 记录真实用户 IP
- 生成回调 URL
- 判断当前是否 HTTPS

### 6.2 手绘图理解反向代理

```text
浏览器
  |
  v
[ Nginx ]
  |
  | proxy_pass
  v
[ Node.js App :3000 ]
```

浏览器以为自己是在和 `api.example.com` 说话。

实际上是 Nginx 先接单，再把请求转交给 Node 应用。

---

## Part 7：`location` 到底怎么匹配？这是新手重灾区 🚨

Nginx 最容易把人搞晕的地方之一，就是 `location` 匹配。

你可能写了三个 `location`，结果发现：

> “为什么走的不是我想的那个块？？？” 😵

### 7.1 最常见的几种写法

```nginx
location = / {
    return 200 "exact match\n";
}

location /api/ {
    proxy_pass http://127.0.0.1:3000;
}

location / {
    root /var/www/site;
}

location ~ \.php$ {
    return 403;
}
```

### 7.2 先记住最实用的一版规则

你先不用背完整官方优先级，先记住这版够用的：

1. `location = /xxx`：精确匹配，优先级很高
2. 前缀匹配里，谁更长谁更具体，通常优先
3. `~` / `~*`：正则匹配
4. 最后兜底通常是 `location /`

### 7.3 大白话理解

```text
location = /login      像“只接待这个具体门牌号”
location /api/         像“凡是 api 这条街的都来这边”
location /             像“剩下的人都来总台”
location ~ \.jpg$      像“名字符合某种模式的人单独处理”
```

### 7.4 为什么这很重要？

因为你的这些能力都依赖正确匹配：

- `/api/` 转发给后端
- `/assets/` 返回静态资源
- `/admin/` 加 IP 白名单
- `/healthz` 直接返回健康检查

如果匹配错了，你会出现非常离谱的现象：

- API 被当成静态文件
- 静态文件被转发到后端
- SPA 路由和真实接口互相打架

---

## Part 8：`proxy_pass` 后面的斜杠，真的是经典大坑 🔪

这个点必须单独讲，因为太多人在这儿摔过。

看两个配置：

### 写法 A

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

### 写法 B

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

看起来就差一个 `/`，但结果可能完全不同。

### 大白话理解

- **没有尾斜杠**：更像把原始路径整体拼过去
- **有尾斜杠**：更像把匹配掉的前缀替换掉

举个例子，请求：

```text
/api/users
```

对于不同写法，转发结果可能变成：

```text
A: http://127.0.0.1:3000/api/users
B: http://127.0.0.1:3000/users
```

所以如果你发现后端路径“莫名其妙少了一截或多了一截”，第一反应就去看：

> `location` 和 `proxy_pass` 后面那个斜杠是不是配错了。

---

## Part 9：负载均衡，让 Nginx 不止会转发，还会分流 🏥

当你后面不止一台应用机器时，就可以用 `upstream`：

```nginx
upstream myapp {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://myapp;
    }
}
```

### 9.1 这是什么意思？

`upstream myapp` 就是：

> 我定义了一组后端机器，名字叫 `myapp`。

然后：

```nginx
proxy_pass http://myapp;
```

就表示：

> 请求别发给某一台固定机器了，发给 `myapp` 这组后端吧。

### 9.2 手绘图理解

```text
浏览器
  |
  v
[ Nginx ]
  |----> App 1 :3001
  |----> App 2 :3002
  |----> App 3 :3003
```

### 9.3 常见策略

默认常常是轮询。

你还会见到一些思路：

- 轮询：一个一个分
- 权重：强机器多干点
- ip_hash：同一 IP 尽量打到同一台

例如：

```nginx
upstream myapp {
    server 10.0.0.1:3000 weight=3;
    server 10.0.0.2:3000 weight=1;
}
```

意思是第一台大概吃更多流量。

---

## Part 10：静态资源为什么适合交给 Nginx？📦

如果你让 Node.js / Python 应用去返回每一张图片、每一个 JS 文件，也不是不行。

但通常不优雅。

因为静态资源的特点是：

- 逻辑简单
- 访问量大
- 特别适合缓存
- 不需要进业务代码

所以非常适合交给 Nginx。

### 10.1 一个典型配置

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    location /assets/ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }
}
```

### 10.2 这里发生了什么？

- Nginx 直接从磁盘读文件
- 直接返回给浏览器
- 同时附带缓存头

浏览器下次看到同样资源时，很可能直接用缓存，不再打到服务端。

这就是为什么它能大幅减轻应用压力。

---

## Part 11：HTTPS / SSL / TLS：为什么经常放在 Nginx 这一层？🔒

你在生产环境里很常见这种架构：

```text
浏览器 -- HTTPS --> Nginx -- HTTP --> 应用服务
```

很多人第一次看到会惊讶：

> “等等，为什么浏览器到 Nginx 是 HTTPS，Nginx 到后端反而可能是 HTTP？” 🤔

因为 Nginx 常常承担“TLS 终止”这个角色。

### 11.1 什么叫 TLS 终止？

就是：

> 浏览器和 Nginx 之间负责加密通信。  
> 到了 Nginx 这里，先把加密解开，再转发给内网应用。

如果后端服务在内网里、机器间链路可信，这样做很常见。

### 11.2 一个最小 HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 11.3 HTTP 自动跳 HTTPS

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

这就是你经常看到“输入 http 自动跳 https”的经典写法。

### 11.4 手绘图理解

```text
浏览器 --(TLS 加密)--> Nginx --(内网转发)--> App
```

一句话理解：

> Nginx 常常是网站门口负责“穿防弹衣”的那个人。 🛡️

---

## Part 12：gzip：让数据瘦身后再上路 🎈

很多文本类资源其实很适合压缩：

- HTML
- CSS
- JS
- JSON

Nginx 可以直接帮你压缩再发给浏览器。

### 12.1 一个常见配置

```nginx
http {
    gzip on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_vary on;
}
```

### 12.2 每个配置在说什么？

- `gzip on;`：开启压缩
- `gzip_min_length 1024;`：太小的内容就别压了，不划算
- `gzip_types ...`：哪些 MIME 类型参与压缩
- `gzip_vary on;`：让缓存代理知道“压缩版本和非压缩版本”不同

### 12.3 为什么不是所有内容都适合 gzip？

因为像图片、视频、zip 这类文件本身已经压缩过了。

你再压一遍，通常：

- 收益不大
- 还浪费 CPU

所以 gzip 主要利好文本资源。

---

## Part 13：缓存：Nginx 不只是“转发”，它还能“记住” 🧠

缓存分两种你先分清：

### 13.1 浏览器缓存

Nginx 给浏览器返回缓存头，比如：

```nginx
location /assets/ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

意思是：

> 浏览器，这些静态资源 30 天内别老来烦我。

### 13.2 代理缓存（Nginx 自己缓存上游响应）

这更像：

> 后端应用返回了一个结果，我 Nginx 先记下来，下次类似请求我直接回，不一定再打后端。

一个简化示意：

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m max_size=1g inactive=60m use_temp_path=off;

server {
    listen 80;
    server_name api.example.com;

    location /news/ {
        proxy_cache mycache;
        proxy_cache_valid 200 1m;
        proxy_pass http://127.0.0.1:3000;
    }
}
```

### 13.3 手绘图理解代理缓存

```text
请求 /news
   |
   v
Nginx 先看本地缓存
   |             |
   | 命中         | 未命中
   v             v
直接返回        去请求后端
                  |
                  v
             把结果存起来
                  |
                  v
               返回用户
```

### 13.4 什么适合缓存？

适合：

- 新闻列表
- 公共内容页
- 某些热点接口

不适合乱缓存：

- 用户私有数据
- 强实时状态
- 购物车、支付、库存这类敏感链路

---

## Part 14：限流：别让别人把你服务器当水龙头拧爆 🚰

限流不是“高冷大厂专属能力”。

它很现实：

- 防止恶意刷接口
- 防止用户误操作狂点
- 防止爬虫把站点打崩
- 防止某些热点接口被瞬间冲爆

### 14.1 一个常见限流配置

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

    server {
        listen 80;
        server_name api.example.com;

        location /api/ {
            limit_req zone=api_limit burst=10 nodelay;
            proxy_pass http://127.0.0.1:3000;
        }
    }
}
```

### 14.2 大白话解释

- `rate=5r/s`：每个 IP 平均每秒 5 个请求
- `burst=10`：允许短时间多冲一点
- `nodelay`：突发请求不排队，能过就过，超过就拒

你可以把它理解成商场安检口：

- 正常速度一个一个进
- 短时间允许多来几个人
- 但你不能一口气带 300 人冲门

### 14.3 限流不是万能药

它只能挡一部分“入口层洪水”。

如果你要做更完整的防护，还会结合：

- CDN/WAF
- 应用层验证码
- 登录态控制
- 风控策略

---

## Part 15：日志：Nginx 到底看到了什么？📜

Nginx 非常适合做入口日志记录。

常见有两个：

- `access_log`：访问日志
- `error_log`：错误日志

### 15.1 一个简单例子

```nginx
server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

### 15.2 为什么它很重要？

因为很多问题并不是后端代码逻辑错，而是：

- 路由没匹配到
- 上游连不上
- 请求太大
- 静态文件路径错了
- 某个代理头没带对

这时候先看 Nginx 日志，往往比你先猜代码更有效。

---

## Part 16：一份更接近真实项目的 Nginx 配置 🧩

下面这份配置把常见能力拼在一起，让你更有“生产味儿”：

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    gzip on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript application/xml;

    upstream backend {
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
    }

    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        listen 80;
        server_name example.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        root /var/www/frontend;
        index index.html;

        location /assets/ {
            expires 30d;
            add_header Cache-Control "public, max-age=2592000";
        }

        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

### 16.1 这一份配置实现了什么？

- 80 自动跳 443
- 443 提供 HTTPS
- `/assets/` 走静态资源缓存
- `/api/` 走后端集群
- API 有简单限流
- 前端路由走 SPA 回退
- 开启了 gzip

如果你能把这份配置吃透，Nginx 就已经不是“陌生名词”了，而是一个开始能上手用的工具了。

---

## Part 17：开发时最常见的 Nginx 命令 🛠️

这里给你一些非常高频的命令感知。

### 17.1 检查配置是否有语法错误

```bash
nginx -t
```

这几乎是每次改配置之后都应该敲的命令。

### 17.2 重载配置

```bash
nginx -s reload
```

意思是：

> 我配置改好了，请你平滑重载一下。

不是粗暴杀掉重启，而是尽量平滑切换。

### 17.3 停止 Nginx

```bash
nginx -s stop
```

### 17.4 查看最终完整配置

```bash
nginx -T
```

这个特别适合排查：

- 你 `include` 了哪些文件
- 实际加载的是哪份配置

---

## Part 18：新手最容易踩的坑合集 🕳️

这部分非常重要，因为这些坑比概念更真实。

### 18.1 `location` 匹配没按你想的走

现象：

- 你以为 `/api/` 会进代理
- 实际走了 `location /`

第一反应：

- 检查匹配规则
- 检查路径有没有多一个 `/`

### 18.2 `proxy_pass` 的尾斜杠坑

现象：

- 后端收到的路径不对

第一反应：

- 看 `location` 和 `proxy_pass` 的组合

### 18.3 静态文件 404

现象：

- 页面打开了，但 JS/CSS 404

第一反应：

- 看 `root` 路径是不是对
- 看实际磁盘文件在不在
- 看 URL 路径和文件目录对应关系

### 18.4 SPA 刷新页面 404

现象：

- 首页正常
- 刷新 `/dashboard` 就 404

第一反应：

- 你是不是忘了 `try_files $uri $uri/ /index.html;`

### 18.5 HTTPS 配完还是打不开

第一反应：

- 证书路径对不对
- 域名匹配不匹配
- 80/443 有没有放通

### 18.6 后端拿不到真实 IP

第一反应：

- 你是不是没传 `X-Real-IP` / `X-Forwarded-For`

### 18.7 改完配置不生效

第一反应：

- `nginx -t`
- `nginx -s reload`
- 看你到底改的是不是被加载的那份配置

---

## Part 19：如果把 Nginx 想成一个真实公司，它在干嘛？🏢

这一段是给你建立直觉的。

把整套系统想成一家公司：

- 用户：访客
- 应用服务：真正处理业务的员工
- 数据库：档案室
- Redis：桌面便签
- Nginx：公司一楼超级前台和门卫

Nginx 的工作就是：

- 先接待所有访客
- 判断你来找谁
- 静态文件直接从前台柜子拿给你
- API 请求送去正确部门
- 某些人来得太猛先限流
- 需要加密通道的走 HTTPS
- 某些公共资料可以缓存起来下次直接给

所以你会发现：

> Nginx 很少碰业务规则，但它极大影响用户能不能顺利到达业务。

它像“入口层的秩序维护者”。

---

## 本章最重要的几个直觉 ✋

如果这一章读完你只记住几个点，我希望你记住这些：

### 1. Nginx 通常站在流量入口，不站在业务深处

它最擅长的是接待、转发、分流、压缩、缓存、限流、HTTPS。

### 2. 它不是业务服务器的替代品，而是业务服务器的前置助手

Node / Python / Go 继续负责业务逻辑，Nginx 负责入口治理。

### 3. 反向代理是它最经典的角色

大多数 Web 项目里，Nginx 最重要的身份就是：

> 把请求稳稳地送到后面的应用服务。

### 4. `location` 和 `proxy_pass` 是新手必须啃下来的两块骨头

这两块一旦搞懂，Nginx 的恐惧感会下降一大半。

### 5. Nginx 的价值不在“炫”，在“把入口层问题专业化”

静态资源、TLS、gzip、缓存、限流这些事情，本来就该交给更擅长的人。

---

## 一张总复盘图：把这一章串起来 🧵

```text
用户请求到来
   |
   v
[ Nginx ]
   |
   |-- 如果是 HTTP -> 跳 HTTPS
   |
   |-- 如果是静态文件 -> 直接从磁盘返回
   |
   |-- 如果是 API -> 反向代理到后端
   |
   |-- 如果后端有多台 -> 负载均衡分流
   |
   |-- 如果资源适合压缩 -> gzip
   |
   |-- 如果资源适合缓存 -> 加缓存头 / 代理缓存
   |
   |-- 如果请求太猛 -> 限流
   |
   v
[ 应用服务 ]
```

现在你应该能比较清楚地回答：

- Nginx 为什么存在
- 它和应用服务的边界在哪
- 它在真实项目里最常见的角色是什么
- 它的几种典型配置大概在干嘛

---

## 下一章你可以继续追问的问题 🚀

1. Nginx 为什么能扛住很多连接？它的事件驱动模型到底是什么？
2. Nginx 和 Node.js / Express / Next.js / Taro 项目在生产部署里通常怎么配合？
3. HTTPS 证书申请和自动续期（比如 Let's Encrypt）到底怎么落地？
4. 生产环境里如何设计更靠谱的缓存策略和限流策略？
