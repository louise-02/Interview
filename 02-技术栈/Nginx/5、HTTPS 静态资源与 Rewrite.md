# 5、HTTPS 静态资源与 Rewrite

## 1、HTTPS 配置

### 1.1 HTTP 强制跳转 HTTPS

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

### 1.2 HTTPS 完整示例

证书目录：`/data/nginx/current/conf/ssl/example.com/`

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # 证书链 + 私钥
    ssl_certificate     /data/nginx/current/conf/ssl/example.com/fullchain.pem;
    ssl_certificate_key /data/nginx/current/conf/ssl/example.com/privkey.pem;

    ssl_session_timeout 1d;
    ssl_session_cache   shared:SSL:50m;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # 确认 HTTPS 稳定后再开 HSTS
    # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

| 参数 | 说明 |
|------|------|
| `ssl_certificate` | 服务端证书（含中间证书链） |
| `ssl_certificate_key` | 私钥 |
| `ssl_session_cache` | 会话复用，减少握手 |
| `ssl_protocols` | 禁用 TLS 1.0/1.1 |
| `http2` | 启用 HTTP/2（需 ssl） |

### 1.3 Let's Encrypt

```bash
certbot certonly --nginx -d example.com -d www.example.com
# 证书通常在 /etc/letsencrypt/live/example.com/
nginx -t && systemctl reload nginx
```

---

## 2、location 匹配规则

| 修饰符 | 示例 | 匹配方式 |
|--------|------|----------|
| 无 | `location /api/` | 前缀匹配 |
| `=` | `location = /` | 精确匹配（最高优先级之一） |
| `^~` | `location ^~ /static/` | 前缀匹配，**匹配后不再走正则** |
| `~` | `location ~ \.php$` | 正则，区分大小写 |
| `~*` | `location ~* \.(jpg\|png)$` | 正则，不区分大小写 |
| `@` | `location @fallback` | 命名 location，供 try_files / error_page 跳转 |

**优先级（简化）**

```text
= 精确 > ^~ 前缀 > ~ / ~* 正则（按配置顺序） > 普通前缀
```

同一前缀越长越优先（如 `/api/v1` 优于 `/api`）。

---

## 3、root 与 alias

```nginx
# root：完整路径 = root + URI
location /images/ {
    root /data/www;          # /images/a.jpg → /data/www/images/a.jpg
}

# alias：完整路径 = alias + URI 去掉 location 部分
location /images/ {
    alias /data/www/img/;    # /images/a.jpg → /data/www/img/a.jpg
}
```

**注意**：`alias` 末尾通常加 `/`；`root` 不在 location 路径末尾重复目录名。

---

## 4、try_files 与 SPA 前端

Vue / React 单页应用：刷新子路由不能 404。

```nginx
location / {
    root   /data/www/dist;
    index  index.html;
    try_files $uri $uri/ /index.html;
}
```

**执行顺序**：先找文件 `$uri` → 再找目录 `$uri/` → 都没有则内部重写到 `/index.html`。

**另一种写法（history 模式）**

```nginx
location / {
    root html;
    index index.html;
    if (!-e $request_filename) {
        rewrite ^/(.*) /index.html last;
    }
}
```

> 推荐 `try_files`，比 `if` 更清晰；`if` 在 Nginx 中有坑。

---

## 5、Rewrite 规则

```nginx
# 永久重定向
rewrite ^/old-path/(.*)$ /new-path/$1 permanent;

# 内部重写（浏览器 URL 不变）
rewrite ^/api/v1/(.*)$ /v1/$1 last;

# 停止当前 rewrite 集，重新匹配 location
# last — 重新走 location
# break — 当前 location 内继续
# redirect / permanent — 302 / 301 给客户端
```

**常用场景**

| 场景 | 示例 |
|------|------|
| 域名统一 | `rewrite ^(.*)$ https://www.example.com$1 permanent;` |
| 去掉 www | `if ($host = 'www.example.com') { return 301 https://example.com$request_uri; }` |
| 旧 URL 迁移 | `rewrite ^/product/(\d+)$ /item/$1 permanent;` |

---

## 6、静态资源与缓存

```nginx
# 带 hash 的构建产物：长期缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
    root /data/www/dist;
    expires 30d;
    add_header Cache-Control "public, immutable";
}

# HTML 不缓存（避免发版后用户看到旧入口）
location ~* \.(html|htm)$ {
    root /data/www/dist;
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

| 策略 | 适用 |
|------|------|
| `expires 30d` | 带 contenthash 的 js/css |
| `no-cache` | index.html、接口文档 |
| `immutable` | 文件名含 hash，永不变 |

---

## 7、安全相关 location

```nginx
# 禁止访问隐藏文件（.git、.env 等）
location ~ /\.(?!well-known) {
    deny all;
    access_log off;
    log_not_found off;
}

# 禁止直接访问敏感后缀
location ~* \.(sql|bak|conf|log)$ {
    deny all;
}
```

**安全响应头（可选）**

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
```

---

## 8、跨域（简单场景）

```nginx
location /api/ {
    proxy_pass http://backend;

    add_header Access-Control-Allow-Origin  $http_origin always;
    add_header Access-Control-Allow-Methods  'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header Access-Control-Allow-Headers  'Authorization, Content-Type' always;
    add_header Access-Control-Allow-Credentials true always;

    if ($request_method = OPTIONS) {
        return 204;
    }
}
```

> 复杂 CORS 建议在后端或 Gateway 统一处理；Nginx 仅适合简单静态/API 入口。

---

## 9、前后端分离完整示例

```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;

    # 前端静态
    location / {
        root   /data/www/dist;
        index  index.html;
        try_files $uri $uri/ /index.html;
    }

    # 后端 API
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection        "";
    }

    location ~ /\.(?!well-known) {
        deny all;
    }
}
```

下一篇：[6、性能优化与常见问题.md](./6、性能优化与常见问题.md)
