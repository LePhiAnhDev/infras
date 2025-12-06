# NGINX-PROXY TỰ ĐỘNG

Cấu hình nginx-proxy tự động cho nhiều website, services trên cùng VPS với Cloudflare SSL.

## TÍNH NĂNG

- Auto-discovery containers mới
- Cloudflare Real IP
- Tối ưu hiệu năng (gzip, keepalive)
- Security headers
- Không cần Let's Encrypt

## CÀI ĐẶT

```bash
cd infras
docker compose up -d
```

## SỬ DỤNG

### Cấu hình tối thiểu

Service cần 3 điều:
1. **Environment**: `VIRTUAL_HOST` (domain name) - **BẮT BUỘC**
2. **Network**: `proxy-network`
3. **Expose**: Port của service

### Ví dụ đơn giản

```yaml
services:
  my-website:
    image: nginx:alpine
    environment:
      - VIRTUAL_HOST=example.com,www.example.com
    networks:
      - proxy-network

networks:
  proxy-network:
    external: true
```

**Lưu ý**: Service chạy port khác 80 thì thêm `VIRTUAL_PORT`:

```yaml
environment:
  - VIRTUAL_HOST=example.com
  - VIRTUAL_PORT=3000
```

### Ví dụ backend + frontend

```yaml
services:
  backend:
    image: node:18-alpine
    environment:
      - VIRTUAL_HOST=api.example.com
      - VIRTUAL_PORT=3000
    networks:
      - proxy-network
    expose:
      - "3000"

  frontend:
    image: nginx:alpine
    environment:
      - VIRTUAL_HOST=example.com,www.example.com
    networks:
      - proxy-network

networks:
  proxy-network:
    external: true
```

## MONITORING

```bash
# Xem logs
docker logs nginx-proxy -f

# Test config
docker exec nginx-proxy nginx -t

# Reload
docker exec nginx-proxy nginx -s reload
```

## LƯU Ý

- Port 80 phải trống
- Domain phải trỏ về VPS
- Mở firewall: `sudo ufw allow 80/tcp`
- Dùng SSL của Cloudflare