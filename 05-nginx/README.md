# Chapter 5: What Is Nginx Actually Busy Doing? 🧑‍💼

## First: Do Not Let the Name Intimidate You

The first time many people hear “Nginx”, their brain goes:

> “Wait... is this a web server? A proxy? A gateway? A load balancer? A static file tool? An SSL thing? Why does it seem to be everything at once?” 😵

That confusion is completely normal.

Because in real projects, Nginx often really does play many roles at the same time:

- web server
- reverse proxy
- load balancer
- HTTPS termination layer
- static file server
- cache layer
- rate limiter

So yes, it can feel like “that one coworker who somehow shows up in every meeting.”

But if we summarize Nginx in one plain sentence:

> **It is a traffic coordinator sitting at the system entrance, and it is very good at forwarding, routing, caching, compressing, and protecting requests before they hit your application.**

This chapter keeps the same style as Chapters 1 and 2:

- plain language
- story-driven explanation
- developer-oriented examples
- hand-drawn markdown diagrams
- practical config snippets

And this one is especially written for people who have barely touched Nginx before.

***

## The Three Questions

### 1. Why does Nginx exist? 🤔

Let’s start with the most naive possible setup:

```text
Browser ---> Node.js app
```

For a toy project, a personal demo, or a learning app, this is totally fine.

But the moment the project gets even slightly real, a bunch of problems show up:

- static files are being served by the application itself
- HTTPS certificates and redirects become annoying
- if you have multiple app instances, who distributes traffic?
- some requests should never hit the app at all
- if the app restarts or slows down, who protects the outside world from the mess?
- if you want gzip, caching, IP controls, or rate limiting, do you really want to code all of that into the business app?

That is when you want a professional component standing in front of the app.

That component is often Nginx.

Think of it like a super-powered building front desk + security guard + traffic controller + package room:

- everyone arrives at the front desk first
- the front desk decides where they should go
- some visitors just pick up a file and leave
- some are redirected
- some are blocked
- some are distributed to different departments

Your application no longer has to be the receptionist, security guard, courier, and business department all at once.

### 2. Why choose Nginx? Are there alternatives? 🧰

Nginx became popular for a reason, not just because “everyone uses it.”

It is very well suited for the entry layer of web systems:

- config-driven and predictable
- strong static file handling
- mature reverse proxy behavior
- great concurrency characteristics
- massive amount of community knowledge
- extremely common in Linux production environments

Of course, Nginx is not the only option:

- Apache: classic and battle-tested
- Caddy: very smooth HTTPS experience
- Envoy: more modern, more cloud-native, more mesh-oriented
- Traefik: very friendly for containers and dynamic service discovery

But if your goal is to truly understand reverse proxying, static serving, HTTPS, gzip, caching, and load balancing, Nginx is still one of the best classic starting points.

### 3. How does Nginx actually work, and how do developers use it? 🔍

That is the main story of this chapter.

We will cover:

- where Nginx sits in the request path
- the difference between forward proxy and reverse proxy
- what `server`, `location`, and `upstream` actually mean
- how requests are forwarded to Node / Python / Go apps
- how HTTPS, gzip, caching, and rate limiting are configured
- the most common beginner mistakes

***

## One Diagram to Lock the Big Picture Into Your Head 🧠

```text
Browser
  |
  v
[ DNS ]
  |
  v
[ CDN ]   (sometimes present, sometimes not)
  |
  v
[ Nginx ]
  |------> directly serves static files
  |------> reverse proxies to App 1
  |------> reverse proxies to App 2
  |------> handles HTTPS / gzip / cache / rate limiting
  |
  v
[ Your application service ]
```

One-sentence memory trick:

> **Nginx is usually not the business star of the show, but it is often the first face all traffic sees.**

***

## Part 1: What exactly is Nginx? 🧾

If you really want to give Nginx a job title, here is a practical way to think about it:

- from the “website files” angle: it is a web server
- from the “request forwarding” angle: it is a reverse proxy
- from the “spread requests across machines” angle: it is a load balancer
- from the “stand at the HTTPS gate” angle: it is a TLS termination layer
- from the “limit/cache/control traffic” angle: it is an entry-layer traffic controller

That is why beginners often feel like it has an identity crisis.

It does not.

It is just extremely capable.

***

## Part 2: Forward Proxy vs Reverse Proxy 🪞

This is one of the first concepts beginners mix up.

### 2.1 Forward proxy: proxying the client

Plain version:

> The client uses a middleman to access the outside world.

```text
Client ---> Forward Proxy ---> Target Website
```

Common scenarios:

- internal company outbound proxy
- crawlers using proxy IPs
- controlled outbound network access

The target site usually knows:

> “I am seeing a proxy, not necessarily the real client.”

### 2.2 Reverse proxy: proxying the server

Reverse proxy is the opposite:

> The user thinks they are talking to the website directly, but they hit a middle layer in front of the servers first.

```text
Client ---> Reverse Proxy ---> Application Server
```

The user usually does not care how many servers are behind it or what language they run.

This is Nginx’s most classic role.

### 2.3 One brutal memory trick

- forward proxy: stands in for “me”
- reverse proxy: stands in for “them”

Or even shorter:

> forward proxy is close to the client  
> reverse proxy is close to the server  

***

## Part 3: Why put Nginx in front of the app? 🚪

Suppose your Node app is running directly on port 3000:

```text
User ---> Node.js:3000
```

Soon you want to do things like:

- redirect `http://example.com` to `https://example.com`
- send `/api` to the backend
- serve `/` and `/assets` as static files
- add cache headers to images, CSS, and JS
- enable gzip
- rate limit abusive clients
- make the app restarts less ugly from the outside

Yes, technically you *can* cram all of that into your application.

But then your app becomes:

> business logic + receptionist + security + courier + traffic police all at once. 😵‍💫

So real systems often become:

```text
User ---> Nginx ---> Node / Python / Go app
```

Now the job split is much cleaner:

- Nginx handles entry-layer concerns
- the app handles business logic

***

## Part 4: What is the Nginx config file even saying? 📄

A lot of people open their first Nginx config and immediately feel like:

> `events {}`  
> `http {}`  
> `server {}`  
> `location / {}`  
> “Who owns what here? Why is everything inside everything else?” 😵

Let’s decode it.

### 4.1 A minimal shape

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

A practical way to read the hierarchy:

```text
global config
  |
  +-- events   -> connection/event handling
  |
  +-- http     -> the big HTTP container
        |
        +-- server   -> one virtual host / site
              |
              +-- location -> one path-matching rule
```

### 4.2 What do these blocks do?

#### Global (main context)

Things like:

- `worker_processes`
- `error_log`
- `pid`

These affect the whole Nginx process.

#### `events`

This mainly controls connection/event behavior.

At beginner level, just think:

> “This is where Nginx’s low-level connection handling settings live.”

#### `http`

This is the home base for HTTP-related configuration.

You often see things like:

- `include mime.types`
- `sendfile on`
- `gzip on`
- `upstream`
- `server`

#### `server`

A `server` block is roughly:

> “one set of rules for a domain + port combination.”

Examples:

- who listens on port 80
- who handles `example.com`
- who handles `api.example.com`

#### `location`

This is one of the most important and most-used blocks.

It decides:

> “How should requests for this path pattern be handled?”

For example:

- `/api/` goes to backend
- `/assets/` serves static files
- `/` returns the homepage

***

## Part 5: Start with the easiest case: a static site 🍞

Suppose your front-end build output is in:

```text
/var/www/myapp
```

and contains:

- `index.html`
- `assets/app.js`
- `assets/app.css`

A classic Nginx config looks like this:

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

### 5.1 What does each line mean?

- `listen 80;`: listen on port 80
- `server_name example.com;`: handle this hostname
- `root /var/www/myapp;`: static file root directory
- `index index.html;`: default entry file
- `try_files $uri $uri/ /index.html;`: try the real file, then fallback to `index.html`

### 5.2 Why do SPA frontends often need `try_files`?

Because client-side routes often look like:

```text
/dashboard
/profile
/orders/123
```

Those look like real paths to the browser, but they are not necessarily real files on disk.

Without `try_files`, Nginx says:

> “There is no file called `/orders/123`. Here is your 404.” 😑

But what you actually want is:

> “Even though that file does not exist, send back `index.html` and let the frontend router take over.”

That is why `try_files` is so important for SPAs.

```text
Browser requests /orders/123
      |
      v
Nginx checks whether /orders/123 exists on disk
      |
      | no
      v
Falls back to /index.html
      |
      v
Frontend router takes control
```

***

## Part 6: Reverse proxying, Nginx’s most classic job 🧭

If your application runs on:

```text
127.0.0.1:3000
```

you can place Nginx in front:

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

### 6.1 What is this actually saying?

The key line is:

```nginx
proxy_pass http://127.0.0.1:3000;
```

Which means:

> “Do not process this request here. Forward it to the upstream application.”

The `proxy_set_header` lines are also very important. They tell the backend:

- what host the user originally requested
- what the real client IP was
- what proxy chain the request passed through
- whether the original scheme was HTTP or HTTPS

Backends use this information all the time for:

- logging real client IPs
- generating callback URLs
- detecting HTTPS

### 6.2 Tiny mental diagram

```text
Browser
  |
  v
[ Nginx ]
  |
  | proxy_pass
  v
[ Node.js App :3000 ]
```

The browser thinks it is talking to `api.example.com`.

In reality, Nginx receives the request first and hands it to the app behind the scenes.

***

## Part 7: How does `location` matching work? Beginner danger zone 🚨

This is one of the biggest sources of confusion in Nginx.

You write three `location` blocks, then suddenly think:

> “Why is Nginx not using the one I expected?” 😵

### 7.1 Common forms

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

### 7.2 A practical simplified rule set

You do not need the full official priority table on day one. Start with this:

1. `location = /xxx` -> exact match, very high priority
2. among prefix matches, the more specific / longer one usually wins
3. `~` / `~*` -> regex matching
4. `location /` is often your fallback

### 7.3 Plain-language version

```text
location = /login      = “only this exact address”
location /api/         = “everything on the /api/ street”
location /             = “everyone else goes to the main desk”
location ~ \.jpg$      = “anyone whose name matches this pattern”
```

### 7.4 Why does this matter so much?

Because many core behaviors depend on correct matching:

- `/api/` goes to backend
- `/assets/` serves static content
- `/admin/` has IP restrictions
- `/healthz` returns a health check directly

If matching goes wrong, extremely weird things happen:

- APIs get treated like files
- static files get forwarded to the backend
- SPA routes fight with real server endpoints

***

## Part 8: The trailing slash in `proxy_pass` is a legendary trap 🔪

This one deserves its own section because so many people trip over it.

Look at these two configs:

### Version A

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

### Version B

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

It looks like the same config with one tiny slash difference.

It is not.

### Plain-language mental model

- **without a trailing slash**: more like forwarding the original path as-is
- **with a trailing slash**: more like replacing the matched prefix

Suppose the incoming request is:

```text
/api/users
```

The forwarded path may become:

```text
A: http://127.0.0.1:3000/api/users
B: http://127.0.0.1:3000/users
```

So if your backend receives the wrong path, one of the first things to inspect is:

> the slash relationship between `location` and `proxy_pass`

***

## Part 9: Load balancing: Nginx can distribute, not just forward 🏥

Once you have more than one app instance, `upstream` comes in:

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

### 9.1 What does this mean?

`upstream myapp` means:

> “I am defining a group of backend servers named `myapp`.”

Then:

```nginx
proxy_pass http://myapp;
```

means:

> “Do not send requests to a single machine. Send them to this backend group.”

### 9.2 Mental picture

```text
Browser
  |
  v
[ Nginx ]
  |----> App 1 :3001
  |----> App 2 :3002
  |----> App 3 :3003
```

### 9.3 Common strategies

Default is often round-robin.

You may also see:

- round robin
- weights
- `ip_hash`

Example:

```nginx
upstream myapp {
    server 10.0.0.1:3000 weight=3;
    server 10.0.0.2:3000 weight=1;
}
```

Meaning the first server gets a bigger share of traffic.

***

## Part 10: Why are static files such a good fit for Nginx? 📦

You *can* have Node.js or Python serve every image, every JS bundle, and every CSS file.

But usually that is not elegant.

Because static resources are:

- simple
- high-volume
- cache-friendly
- not business logic

So they are a perfect fit for Nginx.

### 10.1 A common static config

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

### 10.2 What happens here?

- Nginx reads files straight from disk
- returns them directly to the browser
- attaches cache headers

That means the application does less work and browsers may avoid hitting the server at all next time.

***

## Part 11: HTTPS / SSL / TLS: why is this so often done at the Nginx layer? 🔒

In production, you often see:

```text
Browser -- HTTPS --> Nginx -- HTTP --> App
```

The first time people see this, they often ask:

> “Wait, why is browser -> Nginx HTTPS, but Nginx -> backend sometimes only HTTP?” 🤔

Because Nginx often does TLS termination.

### 11.1 What is TLS termination?

It means:

> The browser talks securely to Nginx.  
> Nginx decrypts the traffic and forwards it internally.  

If the backend runs inside a trusted internal network, this is very common.

### 11.2 Minimal HTTPS config

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

### 11.3 Redirect HTTP to HTTPS

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### 11.4 Tiny diagram

```text
Browser --(TLS encrypted)--> Nginx --(internal forward)--> App
```

One-line memory:

> Nginx is often the person standing at the front door handing out the bulletproof vest. 🛡️

***

## Part 12: gzip: make the payload slimmer before sending 🎈

Many text-based resources compress very well:

- HTML
- CSS
- JS
- JSON

Nginx can compress them before sending.

### 12.1 Common config

```nginx
http {
    gzip on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_vary on;
}
```

### 12.2 What do these lines mean?

- `gzip on;` -> enable gzip
- `gzip_min_length 1024;` -> do not bother compressing tiny payloads
- `gzip_types ...` -> which MIME types should be compressed
- `gzip_vary on;` -> help caches distinguish compressed vs non-compressed responses

### 12.3 Why not gzip everything?

Because files like images, videos, and zip archives are often already compressed.

Trying to gzip them again usually:

- saves almost nothing
- wastes CPU

So gzip is mostly useful for text content.

***

## Part 13: Caching: Nginx does not only forward, it can also remember 🧠

First separate two ideas:

### 13.1 Browser cache

Nginx tells the browser how to cache:

```nginx
location /assets/ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

Meaning:

> “Browser, do not keep asking me for this file every five seconds.”

### 13.2 Proxy cache (Nginx caching upstream responses)

This is more like:

> “The backend already returned this. I will store it here and maybe answer the next similar request myself.”

A simplified example:

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

### 13.3 Tiny proxy cache diagram

```text
Request /news
   |
   v
Nginx checks local cache
   |             |
   | hit         | miss
   v             v
return directly  ask backend
                  |
                  v
             store the result
                  |
                  v
              return to user
```

### 13.4 What is safe to cache?

Good candidates:

- public news lists
- public content pages
- some hot but non-personalized APIs

Bad candidates for careless caching:

- user-private data
- highly real-time state
- carts, payments, inventory-critical responses

***

## Part 14: Rate limiting: do not let people use your server like a fire hose 🚰

Rate limiting is not some fancy “big tech only” feature.

It solves very practical problems:

- malicious abuse
- accidental rapid clicking
- crawlers going wild
- hot APIs getting flooded

### 14.1 A common config

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

### 14.2 Plain-language explanation

- `rate=5r/s` -> average 5 requests per second per IP
- `burst=10` -> allow short bursts above the base rate
- `nodelay` -> do not queue burst requests; let them through immediately until the burst budget is exhausted

Think of it like a security gate at a venue:

- normal visitors pass steadily
- a short burst is tolerated
- but 300 people cannot all rush the door at once

### 14.3 Rate limiting is not magic armor

It helps at the entry layer.

But real protection often also includes:

- CDN / WAF
- captchas
- auth requirements
- business-level anti-abuse rules

***

## Part 15: Logs: what did Nginx actually see? 📜

Nginx is a great place to record entry-layer logs.

Two very common ones:

- `access_log`
- `error_log`

### 15.1 Example

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

### 15.2 Why do they matter?

Because many issues are not actually “the backend code is wrong.”

Sometimes the real problem is:

- route mismatch
- upstream unreachable
- request body too large
- wrong static file path
- missing proxy headers

In those cases, checking Nginx logs first is often smarter than guessing in the app code.

***

## Part 16: A more realistic Nginx config for a real-ish project 🧩

Here is a config that combines many common features:

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

### 16.1 What does this config achieve?

- port 80 redirects to 443
- port 443 serves HTTPS
- `/assets/` gets static caching
- `/api/` goes to a backend pool
- API gets simple rate limiting
- SPA routes fall back to `index.html`
- gzip is enabled

If you can read this config with confidence, Nginx has already stopped being a mysterious buzzword and started becoming a practical tool.

***

## Part 17: Common Nginx commands developers actually use 🛠️

### 17.1 Test config syntax

```bash
nginx -t
```

This is the command you should almost always run after changing config.

### 17.2 Reload config

```bash
nginx -s reload
```

Meaning:

> “I updated the config. Please reload it gracefully.”

### 17.3 Stop Nginx

```bash
nginx -s stop
```

### 17.4 Print the full effective config

```bash
nginx -T
```

This is especially useful when:

- you use `include`
- you are not sure which file is actually being loaded

***

## Part 18: The beginner trap collection 🕳️

This section matters a lot, because these are more real than abstract theory.

### 18.1 `location` does not match the way you expected

Symptom:

- you think `/api/` should proxy
- it ends up handled by `location /`

First check:

- matching rules
- path shape
- missing or extra `/`

### 18.2 The `proxy_pass` trailing slash trap

Symptom:

- backend receives the wrong path

First check:

- relationship between `location` and `proxy_pass`

### 18.3 Static files return 404

Symptom:

- page opens
- JS/CSS files do not

First check:

- `root` path correctness
- whether files actually exist
- path vs disk directory mapping

### 18.4 SPA refresh gives 404

Symptom:

- homepage works
- refreshing `/dashboard` returns 404

First check:

- did you forget `try_files $uri $uri/ /index.html;`?

### 18.5 HTTPS still does not work

First check:

- certificate path
- hostname matching
- ports 80/443 open

### 18.6 Backend cannot see the real client IP

First check:

- did you pass `X-Real-IP` / `X-Forwarded-For`?

### 18.7 Config changes do not apply

First check:

- `nginx -t`
- `nginx -s reload`
- are you editing the config file that Nginx is actually loading?

***

## Part 19: If Nginx were a real company role, what would it be? 🏢

Think of your system as a company:

- users: visitors
- app services: the employees doing real business work
- database: the archive room
- Redis: sticky notes on desks
- Nginx: the super-capable front desk + security at the entrance

Nginx’s job is:

- receive every visitor first
- decide where they should go
- hand out static files directly from the front desk cabinet
- route API requests to the right department
- rate limit people who rush the door
- handle HTTPS for secure entry
- cache some public content so it can be served faster next time

So the important intuition is:

> Nginx rarely owns business rules, but it heavily influences whether users can reach the business smoothly in the first place.

It is the keeper of order at the entrance layer.

***

## The Most Important Instincts From This Chapter ✋

If you only remember a few things, remember these:

### 1. Nginx usually sits at the entry layer, not deep in the business layer

It is strongest at receiving, forwarding, distributing, compressing, caching, limiting, and terminating HTTPS.

### 2. It does not replace your app server; it helps your app server

Node / Python / Go still handle business logic. Nginx handles traffic governance.

### 3. Reverse proxying is its most classic role

In most web systems, Nginx’s most important job is:

> take requests in cleanly and deliver them safely to the real application behind it.

### 4. `location` and `proxy_pass` are the two bones beginners must chew through

Once those click, Nginx becomes much less scary.

### 5. Nginx’s value is not “being fancy”; it is professionalizing entry-layer problems

Static files, TLS, gzip, caching, rate limiting: these are all better handled by the layer that specializes in them.

***

## One final recap diagram 🧵

```text
User request arrives
   |
   v
[ Nginx ]
   |
   |-- if HTTP -> redirect to HTTPS
   |
   |-- if static file -> return directly from disk
   |
   |-- if API -> reverse proxy to backend
   |
   |-- if multiple backends -> load balance
   |
   |-- if compressible -> gzip
   |
   |-- if cacheable -> add cache headers / proxy cache
   |
   |-- if traffic too aggressive -> rate limit
   |
   v
[ Application Service ]
```

At this point, you should be able to answer pretty clearly:

- why Nginx exists
- where its boundary is relative to the app
- what its most common real-world roles are
- what its typical config blocks are doing

***

## Good Next Questions 🚀

1. Why can Nginx handle so many connections? What is the event-driven model really doing?
2. How does Nginx usually cooperate with Node.js / Express / Next.js in production?
3. How do HTTPS certificates and automatic renewal actually work in practice?
4. How should rate limiting and caching strategies be designed more carefully in production?
