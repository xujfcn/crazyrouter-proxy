# 🔄 Crazyrouter Proxy Configuration

> Nginx reverse proxy and Cloudflare Workers setup for Crazyrouter.

## Nginx Reverse Proxy
```nginx
server {
    listen 443 ssl;
    server_name ai-api.yourdomain.com;

    location / {
        proxy_pass https://crazyrouter.com;
        proxy_set_header Host crazyrouter.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
    }
}
```

## Cloudflare Worker
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = 'crazyrouter.com';
    return fetch(new Request(url, request));
  }
};
```

## 🔗 [Crazyrouter](https://crazyrouter.com?ref=github)
## 📄 License: MIT
