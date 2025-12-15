# NGINX Reverse Proxy Infrastructure

Infrastructure cho NGINX reverse proxy với Docker, tự động phát hiện và route traffic đến containers thông qua `nginx-proxy`. Tối ưu cho Cloudflare CDN với bảo mật và hiệu năng cao.

## Kiến trúc

```
Internet → Cloudflare CDN → NGINX Proxy (Port 80) → Docker Containers
```

**Thành phần chính:**
- `nginx-proxy` container: Tự động phát hiện containers qua Docker socket và tạo cấu hình reverse proxy
- `proxy-network`: Bridge network kết nối các services
- Configuration files: Core config, Cloudflare IPs, security headers, upload limits

## Cấu trúc

```
lephianhdev-infras/
├── docker-compose.yml
├── nginx.conf
└── conf.d/
    ├── client_max_body_size.conf
    ├── cloudflare_real_ip.conf
    └── security_headers.conf
```

## Cấu hình chi tiết

### Docker Compose

**Container:**
- Image: `nginxproxy/nginx-proxy:alpine`
- Restart: `always`
- Port: `80:80` (HTTPS do Cloudflare xử lý)

**Environment:**
- `TRUST_DOWNSTREAM_PROXY=true`: Tin tưởng headers từ Cloudflare
- `ENABLE_IPV6=false`

**Volumes:**
- `/var/run/docker.sock:/tmp/docker.sock:ro`: Đọc Docker events (read-only)
- `./nginx.conf:/etc/nginx/nginx.conf:ro`
- `./conf.d:/etc/nginx/conf.d`

**Logging:** JSON, max 20MB/file, 5 files rotation (~100MB total)

### NGINX Core (`nginx.conf`)

**Performance:**
- `worker_processes auto`: Tự động detect CPU cores
- `worker_connections 1024`
- `sendfile on`, `tcp_nopush on`, `tcp_nodelay on`
- `keepalive_timeout 65`

**Timeouts:**
- Client: 30s
- Send: 30s  
- Proxy: 60s

**Features:**
- `server_tokens off`: Ẩn NGINX version
- Gzip compression cho text/css/js/json
- Custom log format với `$remote_addr` và `$http_x_forwarded_for`

### Client Max Body Size

```nginx
client_max_body_size 100m;
```
Upload tối đa 100MB.

### Cloudflare Real IP

Lấy IP thật của users từ Cloudflare:
- Khai báo 23 IPv4 và 7 IPv6 subnets của Cloudflare
- `real_ip_header CF-Connecting-IP`: Đọc IP từ header Cloudflare
- `real_ip_recursive on`: Xử lý multiple proxy layers

**Lợi ích:** Logs chính xác, rate limiting hiệu quả, geolocation blocking đúng.

### Security Headers

```nginx
X-Frame-Options: SAMEORIGIN           # Chống Clickjacking
X-Content-Type-Options: nosniff       # Ngăn MIME-sniffing
X-XSS-Protection: 1; mode=block       # Enable XSS filter
Referrer-Policy: strict-origin-when-cross-origin
```

## Sử dụng

### 1. Khởi động

```bash
git clone https://github.com/lephianhdev/infras.git
cd infras
docker-compose up -d
docker-compose logs -f
```

### 2. Deploy App

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - proxy-network
    environment:
      - VIRTUAL_HOST=myapp.example.com
      - VIRTUAL_PORT=3000

networks:
  proxy-network:
    external: true
```

**Environment variables:**
- `VIRTUAL_HOST`: Domain để route
- `VIRTUAL_PORT`: Port của app (nếu khác 80)
- `VIRTUAL_PROTO`: http/https (mặc định http)
- `VIRTUAL_PATH`: Path-based routing (optional)

### 3. Multiple Domains

```yaml
VIRTUAL_HOST=app.example.com,www.app.example.com
```

### 4. Path-based Routing

```yaml
VIRTUAL_HOST=example.com
VIRTUAL_PATH=/api/
```

## Luồng xử lý

```
User → Cloudflare (cache, SSL, CF-Connecting-IP) 
→ NGINX (extract real IP, security headers, gzip) 
→ Backend Container → Response 
→ NGINX (headers, compress) 
→ Cloudflare → User
```

## Best Practices

1. **Network:** Internal services dùng separate networks
2. **Variables:** Luôn set `VIRTUAL_HOST`, specify `VIRTUAL_PORT` nếu cần
3. **Logging:** Auto-rotate, monitor định kỳ với `docker-compose logs`
4. **Security:** Dùng Cloudflare SSL "Full" mode, update Cloudflare IPs định kỳ
5. **Performance:** Tăng `worker_connections` nếu traffic cao, enable Cloudflare cache

## Security Notes

- Docker socket mount read-only
- Rate limiting ở Cloudflare level
- SSL/TLS handled by Cloudflare
- Firewall: Chỉ allow Cloudflare IPs (recommended)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.