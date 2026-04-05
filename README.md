# Hardening a Containerized Static Site (Flaude) with nginx

---

## Learning Objectives

By the end of this lab, students will be able to:

1. Serve a static site from a custom nginx Docker container
2. Incrementally apply and verify server-side configurations
3. Use `curl` and browser DevTools to observe and validate HTTP behavior
4. Explain the purpose and tradeoff of each configuration decision
5. Write a minimal but production-realistic `nginx.conf`

---

## Lab Setup

### Project Structure

```
CIS467-SP26-docker-flaude/
├── Dockerfile
├── README.md
├── nginx.conf
├── index.html
├── src/
│   ├── app.js      
│   └── style.css
```

### Base Dockerfile

Students start with this and do not modify it — all changes happen in `nginx.conf`:

```dockerfile
FROM nginx:alpine
COPY site/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

### Rebuild Helper

You can use this command throughout the lab to build and run your image:

```bash
docker build -t lab-nginx . && docker run --rm -p 8080:80 lab-nginx
```

---

## Checkpoint 0 — Baseline (Just Serve Files)

### Goal
Confirm the site loads before any custom configuration is applied.

### nginx.conf

```nginx
events {}

http {
    include /etc/nginx/mime.types;

    server {
        listen 80;
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

### Verification

```bash
curl -I http://localhost:8080/
```

**Expected:** `200 OK`, no special headers, no compression.

### 0.1 - Reflection Question
> What headers does nginx send by default? Are any of them surprising?

--- Server: 
nginx/1.x.x — identifies the server software and version
Date — timestamp of the response
Content-Type — the MIME type of the response
Content-Length — size of the response body in bytes
Last-Modified — when the file was last changed
ETag — a unique identifier for the version of the resource
Accept-Ranges: bytes — tells the client it can request partial content
Connection: keep-alive — indicates the connection will stay open for more requests

## Checkpoint 1 — Compression

### Goal
Reduce asset transfer size for text-based files using gzip.

### Changes to `nginx.conf`

Add inside the `http` block:

```nginx
gzip on;
gzip_types text/plain text/css application/javascript application/json;
gzip_min_length 1024;
```

### Verification

```bash
curl -I -H "Accept-Encoding: gzip" http://localhost:8080/index.js
```

Look for: `Content-Encoding: gzip`

Also verify in browser DevTools → Network tab → select a JS or CSS file →
check the **Response Headers** panel.

### 1.1 Reflection Question
> Why does `gzip_min_length` exist? What's the cost of compressing a 200-byte file?

--- gzip_min_length exists because below a certain file size the overhead of compression costs more than it saves. The default in nginx is 20 bytes which is honestly pretty low. Most people set it somewhere between 256 and 1000 bytes in practice. It is one of those settings that exists to keep you from doing work that makes things slower rather than faster.

## Checkpoint 2 — Cache Control

### Goal
Apply appropriate caching strategies: aggressive caching for fingerprinted assets,
no caching for HTML entry points.

### Changes to `nginx.conf`

Add inside the `server` block:

```nginx
# HTML — always revalidate
location ~* \.html$ {
    add_header Cache-Control "no-cache, must-revalidate";
}

# Fingerprinted assets — cache for 1 year
location ~* \.(js|css|png|jpg|woff2|mp4)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

### Verification

```bash
curl -I http://localhost:8080/index.html
curl -I http://localhost:8080/My_Differential_Equation.mp4
```

Confirm different `Cache-Control` values on each response.

### 2.1 - Reflection Question
> Why would caching `index.html` aggressively be dangerous for a single-page app?
> What would happen if a user's browser cached a stale `index.html` pointing to
> old JS bundles?

--- index.html in a single page app is the entry point for everything. It is the file that tells the browser which JavaScript bundles to go load. And this is where the problem lives. 

 if the browser has aggressively cached the old index.html it is still pointing to main.a3f9c2.js. Which no longer exists on your server.

## Checkpoint 3 — Security Headers

### Goal
Protect users from common browser-level attacks by adding standard security headers.

### Changes to `nginx.conf`

Add inside the `server` block (or a dedicated location):

```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Permissions-Policy "geolocation=(), camera=(), microphone=()";
add_header Content-Security-Policy
    "default-src 'self'; script-src 'self'; style-src 'self';";
```

### Verification

```bash
curl -I http://localhost:8080/
```

All five headers should appear in the response.

Also check: https://securityheaders.com (enter `http://localhost:8080` if using
a tunneling tool, or deploy to a VPS for full scoring).

### 3.1 - Reflection Questions
> Break the CSP intentionally — add an inline `<script>` tag to `index.html`
> and observe the browser console error. What does this teach you about
> how CSP is enforced?

--- CSP is a second line of defense. It is assuming that despite your best efforts something malicious might make it into your HTML and it is the browser's job to refuse to run it. That is a fundamentally different security model than just trying to prevent bad input from getting in at all. You are building a fallback that assumes you might lose the first fight.

## Checkpoint 4 — SPA Routing Fallback

### Goal
Ensure that client-side routes (e.g., `/dashboard`, `/profile/42`) return
`index.html` instead of a 404, allowing JavaScript frameworks to handle routing.

### Setup

Add a link in `index.html` to a route that has no corresponding HTML file:

```html
<a href="/dashboard">Go to Dashboard</a>
```

Without the fallback, clicking this returns a 404.

### Changes to `nginx.conf`

Replace or update the default `location` block:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

### Verification

```bash
curl -I http://localhost:8080/dashboard
```

**Expected:** `200 OK` with the content of `index.html` — not a 404.

Also add a custom 404 page to handle truly missing assets:

```nginx
error_page 404 /404.html;
```

### 4.1 - Reflection Questions
> If every route returns `index.html` with a 200, what are the SEO implications?
> How do SSR frameworks like Next.js solve this problem?

--- The root of the problem is that a standard SPA treats the browser as the place where the app gets built. SSR frameworks move that work to the server so that by the time anything reaches a browser or a crawler the content is already there. The JavaScript still runs on the client afterward to hydrate the page and make it interactive but the critical content does not depend on it. That distinction matters a whole lot for SEO and honestly for performance too since users on slow connections see real content faster as well.

## Checkpoint 5 — Rate Limiting

### Goal
Protect the server from abusive request patterns using nginx's built-in
rate limiting directives.

### Changes to `nginx.conf`

Add to the `http` block:

```nginx
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
```

Apply it in the `server` block:

```nginx
limit_req zone=general burst=20 nodelay;
limit_req_status 429;
```

### Verification

Use a loop to fire rapid requests:

```bash
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" \
  http://localhost:8080/; done
```

Some responses should return `429 Too Many Requests` once the burst is exhausted.

### 5.1 - Reflection Question
> Rate limiting on a static site might seem overkill — when would it actually
> matter in production?

--- For a personal project or a low traffic site rate limiting on purely static content is probably not worth the configuration complexity. But the moment you are in production with real traffic and real users it stops being overkill and starts being basic hygiene. The cost of setting it up is low. The cost of not having it when you need it can be really high. It is one of those things like backing up your database where it feels unnecessary right up until the moment it is the only thing that saves you.

## Checkpoint 6 — Block Sensitive Paths

### Goal
Prevent accidental exposure of configuration files, version control artifacts,
or environment files that might exist in the container.

### Changes to `nginx.conf`

```nginx
location ~ /\. {
    deny all;
    return 404;
}

location ~* \.(env|git|yml|yaml|config)$ {
    deny all;
    return 404;
}
```

### Verification

```bash
# Create a test file to block
echo "SECRET=abc123" > site/.env

# Rebuild and test
curl -I http://localhost:8080/.env
```

**Expected:** `404` — not the file contents.

### 6.1 - Reflection Question
> Why return `404` instead of `403 Forbidden`? What information does each
> status code leak to an attacker?

--- Return the minimum amount of information necessary to communicate what the client needs to know. Error messages, status codes, and even response timing can all leak information to someone who is paying attention. A well designed system thinks about what each response communicates not just to a normal user but to someone who is actively looking for weaknesses.

## Final nginx.conf

You should have a complete, working config. Review it as a whole and identify any ordering issues or redundancies.

---

## Deliverable: Written Reflection (Individual)

Submit a short written response (200-500 words) answering the following:

1. Which configuration had the most visible impact when you verified it? Why?
2. Choose one header or directive you added. Research what a real-world attack
   looks like that it mitigates, and describe it briefly.
3. What does this lab reveal about what managed hosting platforms like Netlify
   are silently doing on your behalf?

--- Question 1 — Most Visible Impact
The configuration that had the most visible impact was gzip compression. And the reason is that it is the one where you can actually see the difference right in front of you. When you pull up the network tab before and after enabling it the file sizes drop and the change is immediate and obvious. Everything else we configured was important but a lot of it is invisible by nature. Security headers do not show you anything dramatic. But gzip shows you the numbers and the numbers are different and that makes the concept stick in a way the others did not.

Question 2 — Real World Attack
The header I want to talk about is X-Frame-Options. It prevents your page from being loaded inside an iframe on another site and the attack it mitigates is clickjacking. An attacker builds a page that looks completely normal but invisibly layered on top of it inside a transparent iframe is your real site. The user thinks they are clicking the attacker's button but they are actually clicking a real button on your site underneath it. Confirming a transfer. Changing an email address. Deleting an account. And they have no idea. Setting X-Frame-Options: DENY means your page refuses to render inside an iframe at all and that attack falls apart immediately.

Question 3 — What Managed Platforms Are Doing Silently
This is the most eye opening part of the lab honestly. When you use Netlify or Vercel you drop your files in and it just works. What this lab reveals is that there is a whole layer of configuration underneath that somebody had to think through. Gzip. Security headers. Caching rules. Rate limiting. All of it is being handled silently on your behalf. And that is great until you need to step outside what the platform provides and you realize you have no idea what is actually running underneath you.

## Grading Rubric

| Component | Points |
|---|---|
| All 6 checkpoints complete with working config | 40 |
| Verification commands run and output documented (screenshots or paste) | 20 |
| Written reflection — depth and specificity | 30 |
| Config is clean, commented, and well-organized | 10 |
| **Total** | **100** |
