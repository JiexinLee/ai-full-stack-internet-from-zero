# Chapter 2: What Is HTTP Actually “Saying”? 📮

## Pull Your Brain Out of Buzzword Hell

When you open a web page, your browser and the server are doing something extremely ordinary:

> Browser: I want this resource  
> Server: sure, here you go (or… no)  

That “chat rule” is HTTP.

This chapter keeps the same style as Chapter 1: plain language, story-driven, developer-first. You will learn:

- what Request/Response headers in DevTools really mean
- who does what among Cookie / Session / JWT / OAuth
- why caching, compression, and CORS often “make your code look broken”
- what HTTP/1.1 vs HTTP/2 vs HTTP/3 vs HTTPS vs TLS are actually fighting in real life

***

## The Three Questions

### 1) Why does HTTP exist? 🤔

Machines can communicate, but they need an agreement:

- first, you establish a connection (TCP)
- then, you need a shared “language”

Otherwise one side says “give me the homepage” and the other side hears “drop the database.” Awkward. 😅

So we needed a universal, cross-language, cross-platform way to describe:

- what you want
- who you are
- what format you are sending
- how the other side should respond

HTTP is that rule: Request + Response.

Think of HTTP as a shipping label:

- you send a package with a label (method/path/headers/body)
- intermediaries route it (CDN/proxy/gateway)
- the receiver replies with a receipt (status/headers/body)

### 2) Why choose HTTP? Are there alternatives? 🧰

HTTP became the default language of the Web because it is:

- simple: request/response
- extensible: headers add capabilities over time
- layered: proxies/CDNs can sit in the middle without breaking the model
- widely supported: browsers, servers, infrastructure all speak it

Alternatives exist:

- WebSocket: long-lived, two-way communication (chat, realtime)
- gRPC: typed RPC that shines for internal services
- MQTT: common in IoT

But for “a browser opens a page,” HTTP remains the most universal choice.

### 3) How does it work in practice (for developers)? 🔍

We will follow one request end-to-end, then zoom into the parts that matter most in daily development: headers, auth, caching, CORS, versions, and HTTPS/TLS.

***

## One “Hand-Drawn” Big Picture 🧠

```text
You type a URL
    |
    v
Browser builds an HTTP Request (method/path/headers/body)
    |
    v
(maybe) CDN / Load Balancer / Reverse Proxy
    |
    v
App server parses Request -> runs business -> builds Response
    |
    v
Browser reads Response (status/headers/body) -> renders
```

***

## Part 1: What does an HTTP Request look like? 🧾

### Request = start line + headers + body

Think of an HTTP request as a sandwich:

- top: request line (method, path, version)
- middle: headers (shipping label)
- bottom: body (the package content, optional)

A typical HTTP/1.1 request:

```http
GET /products/123?from=home HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, br
Connection: keep-alive
Cookie: session_id=abc123
```

In DevTools, “Request Headers” and “Request Payload” are basically a UI version of this.

A POST with a JSON body:

```http
POST /api/login HTTP/1.1
Host: example.com
Content-Type: application/json
Accept: application/json
Content-Length: 45

{"email":"a@b.com","password":"123456"}
```

***

## Part 2: What does an HTTP Response look like? 📦

### Response = status line + headers + body

The server replies:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
Content-Encoding: gzip
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax

{"id":123,"name":"Keyboard","price":199}
```

Think of it as a delivery receipt:

- status code: success/failure/redirect/not modified
- response headers: metadata, caching rules, encoding, cookies
- body: the actual data

***

## Part 3: Methods are not “just strings” 🧠

Common ones:

- `GET`: fetch a resource
- `POST`: create/submit an action
- `PUT`: replace a resource (idempotent)
- `PATCH`: partially update
- `DELETE`: delete (idempotent)
- `HEAD`: headers only
- `OPTIONS`: “what do you support?” (often used for CORS preflight)

Two instincts that matter in real APIs:

1. GET should be safe (do not charge a credit card with GET)
2. idempotency: repeating the same request should not create extra side effects

```text
GET /orders/123      repeat 100 times -> still just reads ✅
DELETE /orders/123   repeat 100 times -> delete once, then no-op ✅
POST /orders         repeat 100 times -> maybe creates 100 orders ❌
```

***

## Part 4: Status codes are the server’s emojis 😄😡

You do not need all of them. You do need the common ones:

### 2xx (success)

- `200 OK`
- `201 Created`
- `204 No Content`

### 3xx (redirect / cache-related)

- `301 Moved Permanently`
- `302 Found`
- `304 Not Modified` (negotiated cache)

### 4xx (client-side problems)

- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`
- `429 Too Many Requests`

### 5xx (server-side problems)

- `500 Internal Server Error`
- `502 Bad Gateway`
- `503 Service Unavailable`

***

## Part 5: Headers are where the magic happens 🪄

Headers are HTTP’s extension points. A lot of the “modern Web” is basically “more headers.”

Here are the ones you will constantly see in development:

### 1) Content-Type: what is the body format?

Common values:

- `application/json`
- `application/x-www-form-urlencoded`
- `multipart/form-data`
- `text/plain`

### 2) Accept: what do you want back?

```http
Accept: text/html
Accept: application/json
Accept: image/avif,image/webp,*/*
```

### 3) Authorization: who are you (for APIs)?

```http
Authorization: Bearer <token>
```

That token might be a JWT, or another signed token format.

### 4) Cookie / Set-Cookie: the browser’s badge system

Server sends:

```http
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
```

Browser later attaches:

```http
Cookie: session_id=abc123
```

Key attributes:

- `HttpOnly`: JS cannot read it (helps reduce XSS cookie theft)
- `Secure`: only sent over HTTPS
- `SameSite`: limits cross-site cookie sending (important against CSRF)

### 5) Cache-Control / ETag: caching controls

- `Cache-Control`: can this be cached, and for how long?
- `ETag`: content version id used for negotiated caching

### 6) Origin / Referer: where did you come from (CORS)?

- `Origin`: the most important one for CORS
- `Referer`: navigation source (historical spelling)

### 7) Connection: keep-alive

Keeping connections alive is a big performance lever (we will cover it).

***

## Part 6: Let’s peel HTTP open with Node 🧑‍🔧

Here is a minimal Node HTTP server that lets you observe:

- which headers the browser actually sends
- how request bodies arrive

### 6.1 Minimal server: print headers + body

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

Test with curl:

```bash
curl -i http://localhost:3000/hello \
  -H "X-Demo: 123" \
  -H "Content-Type: application/json" \
  -d '{"a":1,"b":2}'
```

### 6.2 Make the browser “remember you”: Set-Cookie

Add:

```js
res.setHeader("Set-Cookie", "session_id=abc123; HttpOnly; SameSite=Lax");
```

Visit the endpoint in a browser, then check the next request: you should see `Cookie: session_id=abc123`.

***

## Part 7: Keep-Alive: why not reconnect every time? ☎️

If every HTTP request opens a brand-new TCP connection:

- TCP handshake cost adds up
- TLS handshake cost is even bigger (for HTTPS)
- servers burn resources creating/closing sockets

So HTTP/1.1 tends to keep connections alive:

```http
Connection: keep-alive
```

Think of it as:

> not “call -> say one sentence -> hang up”  
> but “call once -> talk multiple times -> hang up later.” 📞

***

## Part 8: HTTP caching: you think you are requesting, but you are reusing 🧊

Caching is one of the biggest speed wins.

It is also a top reason why developers scream:

> “I changed the code, why is the browser still showing the old thing?” 😭

### 8.1 Strong cache: Cache-Control

Server says:

```http
Cache-Control: max-age=60
```

Meaning: for 60 seconds, do not ask me again—use your local cache.

Rough decision tree:

```text
Browser wants a resource
      |
      v
Do I have a cached copy?
  |        |
  | no     | yes
  v        v
Go server  Is it still fresh?
             |          |
             | expired  | fresh
             v          v
         Go server   use cache (no request)
```

### 8.2 Negotiated cache: ETag + If-None-Match

Browser can ask: “I have version v1, is it still current?”

```http
GET /app.js HTTP/1.1
If-None-Match: "v1"
```

Server replies:

- unchanged: `304 Not Modified` (no body)
- changed: `200 OK` + new body + new `ETag`

```text
Browser: I have cache, not sure if fresh
      |
      v
Ask server with If-None-Match
      |
      v
Server compares ETag
   |               |
   | unchanged      | changed
   v               v
304 no body         200 + new body + new ETag
```

### 8.3 Dev practice: why changes “don’t apply”

Common reason:

- your static file has a long `max-age`
- but the URL never changes (always `/app.js`)

Common production pattern:

> hashed filenames like `/app.8f3a2c.js`  
> long cache for hashed files, short cache for the HTML entry

***

## Part 9: Compression: gzip/br makes payloads smaller 🧳

Browser says:

```http
Accept-Encoding: gzip, br
```

Server replies:

```http
Content-Encoding: gzip
```

Meaning: “I compressed it before sending.”

Minimal gzip in Node:

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

***

## Part 10: CORS: why is the browser so strict? 🧱

That classic error:

> “Blocked by CORS policy …” 😡

Plain English:

> CORS is mainly enforced by the browser, not by your server.

The server might be happy to respond, but the browser says:

> “This page came from origin A. Why are you reading responses from origin B?” 👮

### 10.1 Simple request vs preflight (OPTIONS)

Some cross-origin requests trigger a preflight:

> “Do you allow POST? Do you allow these headers?”

```text
Browser wants cross-origin request
      |
      v
Do we need preflight?
  |              |
  | no           | yes
  v              v
Send request      Send OPTIONS first
                    |
                    v
           Server returns CORS rules
                    |
                    v
             Send the real request
```

### 10.2 Typical CORS headers

```http
Access-Control-Allow-Origin: https://your-frontend.com
Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

If you need cookies:

```http
Access-Control-Allow-Credentials: true
```

And `Allow-Origin` cannot be `*`; it must be a specific origin.

### 10.3 Minimal CORS handling in Node (with preflight)

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

***

## Part 11: The login cluster: Cookie / Session / JWT / OAuth 🧟

One-sentence separation:

- Cookie: small data the browser stores and automatically sends
- Session: server-side record of “who you are”
- JWT: a signed token carrying identity info (more stateless)
- OAuth: third-party authorization (you do not give your password to random apps)

### 11.1 Cookie

Cookies are just browser-managed key/value pairs with sending rules.

### 11.2 Session

Typical flow:

- browser stores `session_id` in a cookie
- server uses that id to fetch session data (in memory/Redis/DB)

```text
First login
   |
   v
Server creates session (memory/Redis/DB)
   |
   v
Server sends session_id via Set-Cookie
   |
   v
Browser later sends Cookie: session_id=...
   |
   v
Server reads session_id -> identifies user
```

Pros: simple, controllable  
Cons: server holds state (multi-instance needs shared store, often Redis)

### 11.3 JWT

Often carried via:

```http
Authorization: Bearer <jwt>
```

Idea:

- server signs a token
- client sends it every request
- server verifies signature (often without session storage)

Pros: stateless scaling  
Cons: revocation is harder, expiration strategy matters, do not store sensitive data inside the token

### 11.4 OAuth

Common “Login with Google/WeChat” pattern:

- user approves on the identity provider
- the app receives authorization and tokens
- user never hands the password to the third-party site

***

## Part 12: HTTP/1.1 vs HTTP/2 vs HTTP/3: what are they fighting? 🥊

### 12.1 HTTP/1.1: the base (works, but has pain)

Pain points:

- limited concurrency per connection (and head-of-line issues in practice)
- browsers open multiple TCP connections to parallelize
- headers repeat a lot

### 12.2 HTTP/2: multiplexing + header compression

Two key upgrades:

- multiple streams over one TCP connection (multiplexing)
- header compression so we do not repeat the same text constantly

```text
HTTP/1.1:
Conn 1: request A -> response A -> request B -> response B ...

HTTP/2:
One conn:
Stream 1: A
Stream 3: B
Stream 5: C
Packets can interleave
```

### 12.3 HTTP/3: QUIC (new chassis)

HTTP/2 still rides TCP:

- TCP loss can affect the entire connection experience

HTTP/3 runs on QUIC (over UDP) and handles transport differently:

- faster connection setup
- better loss recovery patterns (more isolation per stream)

Simple mental model:

> HTTP/3 is not just a new version number. It changes the transport “chassis.” 🚗

***

## Part 13: HTTPS / TLS: why is it secure? why is SSL “gone”? 🔒

### 13.1 What is HTTPS?

HTTPS = HTTP + TLS.

Meaning:

> HTTP no longer runs naked. It encrypts first, then sends. 🧥

### 13.2 What does TLS give you?

- confidentiality: packet sniffers cannot read the content
- integrity: tampering is detected
- authentication: you are talking to the real server, not an impostor

### 13.3 Why is SSL deprecated?

Old SSL versions had serious security problems. Modern systems use TLS.

People still say “SSL certificate” historically, but the protocol today is TLS.

### 13.4 TLS handshake: the only diagram you need at first

```text
Browser <------> Server
   |
   | 1) Browser: supported cipher suites
   |
   | 2) Server: chooses one + sends certificate
   |
   | 3) Browser: verifies certificate chain (CA)
   |
   | 4) Both: agree on a session key
   |
   v
HTTP data is encrypted with the session key
```

Common production TLS issues:

- expired certificate
- domain mismatch
- incomplete certificate chain

***

## Summary: HTTP is not magic ✅

One-sentence summary:

> HTTP is just a request/response rule; headers are extension slots;  
> caching, compression, auth, CORS, and protocol versions exist to make the Web faster, safer, and more reliable.  

At this point you should be able to:

- look at headers and understand what they mean in human terms
- separate cookie/session/jwt/oauth by problem they solve
- debug common “dev pain” like CORS, 304/ETag, caching, compression
- explain the motivation for HTTP/2/3 and HTTPS/TLS

***

## Good Next Questions 🚀

1. How do DNS/TCP/TLS connect to the HTTP request you see in DevTools?
2. How do connection reuse, pools, and proxies impact latency in real systems?
3. If you design a production API, which headers and status codes must be intentionally designed?
