# 🔄 Crazyrouter Proxy Configuration Guide

> Set up Nginx reverse proxy or Cloudflare Workers for Crazyrouter API.

[Crazyrouter](https://crazyrouter.com?utm_source=github&utm_medium=github&utm_campaign=dev_community) — One API key, model-specific API access.

## Why Proxy?

- **Custom domain**: Use your own domain (e.g., `ai-api.yourdomain.com`)
- **Access control**: Add IP whitelisting, rate limiting
- **Logging**: Monitor API usage at the proxy level
- **Routing**: Control your ingress; requests still leave your infrastructure for Crazyrouter processing.

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
        proxy_ssl_server_name on;
        proxy_ssl_name crazyrouter.com;
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

Merge the location rule into the complete TLS server above; retain its certificate settings.
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
        proxy_ssl_server_name on;
        proxy_ssl_name crazyrouter.com;
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
        proxy_ssl_server_name on;
        proxy_ssl_name crazyrouter.com;
    proxy_buffering off;
}
```

## ☁️ Cloudflare Workers

### Basic Proxy
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.protocol = "https:";
    url.hostname = "crazyrouter.com";
    url.port = "";

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

Set `CRAZYROUTER_KEY` and a separate `PROXY_ACCESS_TOKEN` as Worker secrets. Clients must send the proxy token; never expose an unauthenticated proxy backed by your paid API key.
```javascript
export default {
  async fetch(request, env) {
    if (!env.PROXY_ACCESS_TOKEN || request.headers.get("Authorization") !== `Bearer ${env.PROXY_ACCESS_TOKEN}`) {
      return new Response("Unauthorized", { status: 401 });
    }
    const url = new URL(request.url);
    url.protocol = "https:";
    url.hostname = "crazyrouter.com";
    url.port = "";

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
      - "8443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/ssl/certs:ro
      - ./private:/etc/ssl/private:ro
    restart: unless-stopped
```

## ❓ FAQ

**Q: Do I need a proxy?**
A: No, you can use Crazyrouter directly. Proxy is optional for custom domains or access control.

**Q: Does streaming work through proxy?**
A: Yes, make sure `proxy_buffering off` is set in Nginx.

## 🔗 Links
- 🌐 [Crazyrouter](https://crazyrouter.com?utm_source=github&utm_medium=github&utm_campaign=dev_community)
- 💬 [Telegram](https://t.me/crzrouter)

## 📄 License
MIT
