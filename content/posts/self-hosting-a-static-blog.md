---
title: "Self-Hosting a Static Blog with Hugo + Git + CI/CD on Proxmox"
date: 2025-08-20
tags: ["self-hosting", "hugo", "proxmox", "docker", "blog"]
draft: false
---

## Introduction

I always wanted to document and showcase my findings without depending on third-party blogging platforms.  
The solution? **Self-hosting a static blog**.

In this post, I’ll share how I set up my personal blog using:

- **Hugo** → Static site generator  
- **Git** → Version control for my posts  
- **CI/CD** → Auto-deploy on changes  
- **Proxmox** → My homelab server as the host  
- **Cloudflare Tunnel** → To expose my blog to the internet securely  

This setup is simple, fast, and gives me full control over my content.

---

## Why Hugo?

- **Speed**: Hugo generates static HTML in milliseconds.  
- **Markdown**: Posts are just Markdown files.  
- **Themes**: Huge library of ready-to-use themes.  
- **Low Maintenance**: No heavy database or backend.

Perfect for self-hosting with minimal resources.

---

## Step 1: Install Hugo Locally

On your development machine:

```bash
sudo apt install hugo
````

Create a new site:

```bash
hugo new site myblog
cd myblog
```

Initialize Git and add a theme:

```bash
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke themes/ananke
echo 'theme = "ananke"' >> hugo.toml
```

Create your first post:

```bash
hugo new posts/hello-world.md
```

Run locally:

```bash
hugo server -D
```

Visit [http://localhost:1313](http://localhost:1313) to see your site 🎉

---

## Step 2: Dockerize the Blog

We’ll use a **multi-stage Dockerfile** with Nginx:

```dockerfile
# Stage 1: Install Hugo in Alpine
FROM alpine:latest AS builder
RUN apk add --no-cache hugo
WORKDIR /src
COPY . .
RUN hugo --minify

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=builder /src/public /usr/share/nginx/html
```

Build & run locally:

```bash
docker build -t myblog .
docker run -d -p 8080:80 myblog
```

Now your Hugo blog is served by **Nginx** at `http://localhost:8080`.

---

## Step 3: Deploy on Proxmox

1. Create a VM or LXC container with **Docker installed**.
2. Clone your blog repo:

   ```bash
   git clone https://github.com/<your-username>/myblog.git
   cd myblog
   ```
3. Build & run the container:

   ```bash
   docker build -t myblog .
   docker run -d --name myblog -p 8080:80 myblog
   ```

---

## Step 4: Expose with Cloudflare Tunnel

Instead of opening ports, I use **Cloudflare Tunnel**.

Install `cloudflared` and authenticate:

```bash
cloudflared tunnel login
```

Create a tunnel:

```bash
cloudflared tunnel create myblog
```

Map it to your container:

```bash
cloudflared tunnel route dns myblog blog.mydomain.com
```

Config file (`~/.cloudflared/config.yml`):

```yaml
tunnel: myblog
credentials-file: /root/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: blog.mydomain.com
    service: http://localhost:8080
  - service: http_status:404
```

Run the tunnel as a service:

```bash
cloudflared tunnel run myblog
```

Now your blog is live at `https://blog.mydomain.com` 🎉

---

## Step 5: Automate with GitHub Actions (CI/CD)

Whenever I push a new post, I want my server to rebuild & redeploy automatically.

Here’s a simple GitHub Actions workflow:

```yaml
name: Deploy Hugo Blog

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t myblog .
      - name: Stop Old Container
        run: docker rm -f myblog || true
      - name: Run New Container
        run: docker run -d --name myblog -p 8080:80 myblog
```

This uses a **self-hosted runner** on my Proxmox server.
Whenever I push to `main`, the site rebuilds and restarts automatically.

---

## Future Plans

I plan to extend this setup with **n8n**:

* Automating post publishing (e.g., write in Notion → auto-sync to Hugo).
* Auto-summaries and social media sharing.
* Notifications when a new post goes live.

---

## Conclusion

Self-hosting with Hugo + Git + CI/CD on Proxmox is a **lightweight, secure, and fun way** to start blogging.
Using **Cloudflare Tunnel** makes it easy to expose the site without opening firewall ports.

If you want to take control of your blogging journey — give this setup a try.

👉 Next step for me: Automating blog updates with **n8n**.

---

*Thanks for reading! If you’re self-hosting your blog too, I’d love to hear your setup.*
