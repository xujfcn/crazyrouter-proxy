# 🔄 Crazyrouter Proxy Configuration Guide

> Set up Nginx reverse proxy or Cloudflare Workers for Crazyrouter API.

[Crazyrouter](https://crazyrouter.com?ref=github) — One API key, 300+ AI models, 45% cheaper.

## Why Proxy?

- **Custom domain**: Use your own domain (e.g., `ai-api.yourdomain.com`)
- **Access control**: Add IP whitelisting, rate limiting
- **Logging**: Monitor API usage at the proxy level
- **Compliance**: Keep API traffic within your infrastructure

## 🔧 Nginx Reverse Proxy

### Basic Setup
```nginx
server {
    listen 443 ssl http2;
    server_name ai-api.yourdomain.com;

    ssl_certificate /etc/ssl/certs/yourdomain.pem;
    ssl_certificate_key /etc/ssl/private/yourdomain.key;

    location / {
        proxy_pass https://crazyrouter.com;
        proxy_set_header Host crazyrouter.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # SSE streaming support
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
        chunked_transfer_encoding on;
    }
}
```

### With Rate Limiting
```nginx
# Define rate limit zone
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;

server {
    listen 443 ssl http2;
    server_name ai-api.yourdomain.com;

    location / {
        limit_req zone=api burst=10 nodelay;
        proxy_pass https://crazyrouter.com;
        proxy_set_header Host crazyrouter.com;
        proxy_buffering off;
    }
}
```

### With IP Whitelist
```nginx
location / {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass https://crazyrouter.com;
    proxy_set_header Host crazyrouter.com;
    proxy_buffering off;
}
```

## ☁️ Cloudflare Workers

### Basic Proxy
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "crazyrouter.com";

    const newRequest = new Request(url, {
      method: request.method,
      headers: request.headers,
      body: request.body,
    });

    return fetch(newRequest);
  }
};
```

### With API Key Injection
```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    url.hostname = "crazyrouter.com";

    const headers = new Headers(request.headers);
    headers.set("Authorization", `Bearer ${env.CRAZYROUTER_KEY}`);

    return fetch(new Request(url, {
      method: request.method,
      headers,
      body: request.body,
    }));
  }
};
```

## 🐳 Docker (Nginx)

```yaml
version: "3.8"
services:
  proxy:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    restart: unless-stopped
```

## ❓ FAQ

**Q: Do I need a proxy?**
A: No, you can use Crazyrouter directly. Proxy is optional for custom domains or access control.

**Q: Does streaming work through proxy?**
A: Yes, make sure `proxy_buffering off` is set in Nginx.

## 🔗 Links
- 🌐 [Crazyrouter](https://crazyrouter.com?ref=github)
- 💬 [Telegram](https://t.me/crzrouter)

## 📄 License
MIT

