# 📚 Hướng Dẫn Chi Tiết Cấu Hình Nginx Virtual Host An Toàn

> **Mục đích:** Tài liệu training cho System Engineer về cấu hình Nginx production-ready với HTTPS và security headers

---

## 📑 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Rate Limiting Zones](#1-rate-limiting-zones)
3. [Upstream Backend](#2-upstream-backend)
4. [HTTP Server Block](#3-http-server-block-redirect)
5. [HTTPS Server Block](#4-https-server-block)
6. [SSL/TLS Configuration](#5-ssltls-configuration)
7. [Security Headers](#6-security-headers)
8. [ModSecurity WAF](#7-modsecurity-waf)
9. [Rate Limiting & DDoS Protection](#8-rate-limiting--ddos-protection)
10. [Logging](#9-logging)
11. [Location Blocks](#10-location-blocks)
12. [Checklist Triển Khai](#checklist-triển-khai)

---

## Tổng Quan

File template này cung cấp cấu hình Nginx **production-ready** với:
- ✅ HTTPS bắt buộc (HTTP auto-redirect)
- ✅ Security headers đầy đủ (A+ SSL Labs)
- ✅ ModSecurity WAF protection
- ✅ Rate limiting & DDoS protection
- ✅ Tối ưu performance

---

## 1. Rate Limiting Zones

### 📋 Mô tả
Định nghĩa các zone để giới hạn số request và connection từ client, chống DDoS.

```nginx
limit_req_zone $binary_remote_addr zone=sample_ratelimit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=sample_conn:10m;
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị mẫu | Bắt buộc/Option | Khuyến nghị |
|---------|---------|-------------|-----------------|-------------|
| `$binary_remote_addr` | Biến lưu IP client (dạng binary, tiết kiệm RAM) | - | ✅ Bắt buộc | Dùng binary thay vì text |
| `zone=` | Tên zone và dung lượng bộ nhớ | `sample_ratelimit:10m` | ✅ Bắt buộc | 10m = ~160k IP addresses |
| `rate=` | Số request tối đa mỗi giây | `10r/s` | ✅ Bắt buộc | Điều chỉnh theo nhu cầu |

### 💡 Giải thích
- **limit_req_zone**: Giới hạn số request/giây từ 1 IP
- **limit_conn_zone**: Giới hạn số kết nối đồng thời từ 1 IP
- **10m**: 10MB RAM, lưu được ~160,000 địa chỉ IP
- **10r/s**: Cho phép 10 requests/giây/IP

### 🎯 Khi nào điều chỉnh?
- **API server**: Giảm xuống `5r/s` hoặc `3r/s`
- **Website thông thường**: `10r/s` - `20r/s`
- **High traffic site**: `50r/s` - `100r/s`

---

## 2. Upstream Backend

### 📋 Mô tả
Định nghĩa các backend server cho reverse proxy (load balancing).

```nginx
upstream sample_backend {
    least_conn;
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
    # server 127.0.0.1:8081 max_fails=3 fail_timeout=30s backup;
    keepalive 32;
}
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị mẫu | Bắt buộc/Option | Khuyến nghị |
|---------|---------|-------------|-----------------|-------------|
| `upstream` name | Tên backend group | `sample_backend` | ✅ Bắt buộc | Đặt tên có ý nghĩa |
| `least_conn` | Thuật toán load balancing | - | ⚙️ Option | Alternatives: `ip_hash`, `round_robin` |
| `server` | Địa chỉ backend server | `127.0.0.1:8080` | ✅ Bắt buộc | IP:Port hoặc unix socket |
| `max_fails` | Số lần fail trước khi đánh dấu down | `3` | ⚙️ Option | Default: 1 |
| `fail_timeout` | Thời gian đánh dấu server down | `30s` | ⚙️ Option | Default: 10s |
| `backup` | Server dự phòng | - | ⚙️ Option | Chỉ dùng khi primary fail |
| `keepalive` | Số kết nối persistent tới backend | `32` | ⚙️ Option | Tăng performance |

### 💡 Load Balancing Methods

| Method | Mô tả | Khi nào dùng |
|--------|-------|--------------|
| `round_robin` | Phân phối tuần tự (default) | Các server cấu hình giống nhau |
| `least_conn` | Gửi đến server có ít connection nhất | Request xử lý lâu, không đồng đều |
| `ip_hash` | Cùng IP luôn về cùng server | Cần session persistence |
| `hash $request_uri` | Hash theo URL | Cache optimization |

### 🎯 Ví dụ cấu hình thực tế

#### Multiple backends (High Availability):
```nginx
upstream sample_backend {
    least_conn;
    server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;
    keepalive 64;
}
```

#### Unix socket (PHP-FPM style):
```nginx
upstream php_backend {
    server unix:/var/run/php-fpm.sock;
    keepalive 16;
}
```

---

## 3. HTTP Server Block (Redirect)

### 📋 Mô tả
Server block lắng nghe port 80 (HTTP) và redirect toàn bộ traffic sang HTTPS.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name sample.com.vn www.sample.com.vn;
    
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/certbot;
        allow all;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị mẫu | Bắt buộc/Option | Khuyến nghị |
|---------|---------|-------------|-----------------|-------------|
| `listen 80` | Lắng nghe IPv4 port 80 | `80` | ✅ Bắt buộc | Standard HTTP port |
| `listen [::]:80` | Lắng nghe IPv6 port 80 | `[::]:80` | ⚙️ Option | Khuyến nghị có |
| `server_name` | Domain names | `sample.com.vn www.sample.com.vn` | ✅ Bắt buộc | Thay bằng domain thực |
| `return 301` | HTTP redirect code | `301` | ✅ Bắt buộc | 301 = permanent |

### 💡 Giải thích ACME Challenge

```nginx
location ^~ /.well-known/acme-challenge/ {
    default_type "text/plain";
    root /var/www/certbot;
    allow all;
}
```

- **Mục đích**: Cho phép Let's Encrypt verify domain ownership
- **Bắt buộc**: ✅ Nếu dùng Let's Encrypt/Certbot
- **`^~`**: Prefix match, ưu tiên cao
- **root**: Thư mục chứa challenge files

### 🎯 Các loại redirect codes

| Code | Tên | Khi nào dùng |
|------|-----|--------------|
| `301` | Moved Permanently | **Khuyến nghị** - SEO friendly |
| `302` | Found (Temporary) | Testing, tạm thời |
| `307` | Temporary Redirect | Giữ nguyên HTTP method |
| `308` | Permanent Redirect | Giữ nguyên HTTP method |

---

## 4. HTTPS Server Block

### 📋 Mô tả
Server block chính, lắng nghe port 443 (HTTPS) và xử lý tất cả requests.

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name sample.com.vn www.sample.com.vn;
    
    root /var/www/sample.com.vn/public;
    index index.html index.htm index.php;
    
    charset utf-8;
    # ... SSL config và locations
}
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị mẫu | Bắt buộc/Option | Khuyến nghị |
|---------|---------|-------------|-----------------|-------------|
| `listen 443 ssl http2` | HTTPS port + HTTP/2 | `443 ssl http2` | ✅ Bắt buộc | Bật HTTP/2 cho performance |
| `listen [::]:443 ssl http2` | IPv6 HTTPS | `[::]:443 ssl http2` | ⚙️ Option | Khuyến nghị có |
| `server_name` | Domain names | `sample.com.vn www.sample.com.vn` | ✅ Bắt buộc | Phải trùng với SSL cert |
| `root` | Document root | `/var/www/sample.com.vn/public` | ✅ Bắt buộc | Thư mục chứa files |
| `index` | Default index files | `index.html index.htm index.php` | ⚙️ Option | Thứ tự ưu tiên |
| `charset` | Character encoding | `utf-8` | ⚙️ Option | Khuyến nghị utf-8 |

### 💡 HTTP/2 Benefits
- ✅ Multiplexing: Nhiều request/response đồng thời
- ✅ Header compression
- ✅ Server push
- ✅ Performance tăng 20-50%

---

## 5. SSL/TLS Configuration

### 📋 Mô tả
Cấu hình SSL/TLS certificates và security settings.

```nginx
# Certificate paths
ssl_certificate /etc/letsencrypt/live/sample.com.vn/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/sample.com.vn/privkey.pem;
ssl_trusted_certificate /etc/letsencrypt/live/sample.com.vn/chain.pem;

# SSL protocols and ciphers
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...';
ssl_prefer_server_ciphers on;

# Session cache
ssl_session_cache shared:SSL:50m;
ssl_session_timeout 1d;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

# DH Parameters
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `ssl_certificate` | File certificate (public key) | Path to fullchain.pem | ✅ Bắt buộc | Phải có certificate + intermediate |
| `ssl_certificate_key` | File private key | Path to privkey.pem | ✅ Bắt buộc | Bảo mật file này (chmod 600) |
| `ssl_trusted_certificate` | Chain certificates | Path to chain.pem | ⚙️ Option | Cần cho OCSP stapling |
| `ssl_protocols` | TLS versions cho phép | `TLSv1.2 TLSv1.3` | ✅ Bắt buộc | **Không dùng** TLSv1.0, TLSv1.1 |
| `ssl_ciphers` | Cipher suites | Strong ciphers only | ✅ Bắt buộc | Ưu tiên ECDHE, AEAD |
| `ssl_prefer_server_ciphers` | Server chọn cipher | `on` | ⚙️ Option | `on` cho TLS 1.2, `off` cho 1.3 |

### 📊 SSL Session Parameters

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `ssl_session_cache` | Cache SSL sessions | `shared:SSL:50m` | ⚙️ Option | 50m = ~200k sessions |
| `ssl_session_timeout` | Thời gian cache session | `1d` (1 day) | ⚙️ Option | Balance security/performance |
| `ssl_session_tickets` | TLS session tickets | `off` | ⚙️ Option | `off` tốt hơn cho security |

### 📊 OCSP Stapling

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `ssl_stapling` | Enable OCSP stapling | `on` | ⚙️ Option | ✅ Khuyến nghị bật |
| `ssl_stapling_verify` | Verify OCSP response | `on` | ⚙️ Option | Bật nếu có trusted cert |
| `resolver` | DNS servers | `8.8.8.8 8.8.4.4` | ⚙️ Option (nếu bật stapling) | Dùng DNS nhanh |
| `resolver_timeout` | DNS timeout | `5s` | ⚙️ Option | Default là đủ |

### 💡 Cipher Suite Explained

```nginx
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...';
```

**Cấu trúc cipher:** `KeyExchange-Auth-Encryption-Hash`

| Component | Giải thích | Ví dụ |
|-----------|------------|-------|
| **ECDHE** | Elliptic Curve Diffie-Hellman Ephemeral (Perfect Forward Secrecy) | ✅ Khuyến nghị |
| **RSA/ECDSA** | Authentication method | Tùy loại certificate |
| **AES128/AES256** | Encryption algorithm | AES-GCM ưu tiên (AEAD) |
| **GCM** | Galois/Counter Mode | AEAD cipher mode |
| **SHA256** | Hash algorithm | SHA256/SHA384 |

### 🎯 Tạo DH Parameters

```bash
# Strong DH parameters (4096-bit, mất ~5-10 phút)
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096

# Nhanh hơn (2048-bit, vẫn an toàn)
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

| Tham số | Ý nghĩa | Bắt buộc/Option | Khuyến nghị |
|---------|---------|-----------------|-------------|
| `ssl_dhparam` | DH parameters file | ⚙️ Option | ✅ Cần cho cipher DHE |

---

## 6. Security Headers

### 📋 Mô tả
HTTP headers để bảo vệ chống các attack phổ biến.

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Content-Security-Policy "..." always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "..." always;
```

### 📊 Chi tiết từng header

#### 🔒 Strict-Transport-Security (HSTS)

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

| Directive | Ý nghĩa | Giá trị | Bắt buộc/Option |
|-----------|---------|---------|-----------------|
| `max-age` | Thời gian cache HTTPS (giây) | `63072000` (2 năm) | ✅ Bắt buộc |
| `includeSubDomains` | Áp dụng cho subdomains | - | ⚙️ Option |
| `preload` | Đưa vào HSTS preload list | - | ⚙️ Option |
| `always` | Thêm header cho mọi response | - | ✅ Khuyến nghị |

**⚠️ Cảnh báo:**
- Phải test kỹ trước khi bật `includeSubDomains`
- Không thể rollback dễ dàng sau khi `preload`
- Bắt đầu với `max-age=300` (5 phút) để test

**Chống attack:** SSL Stripping

---

#### 🔒 X-Content-Type-Options

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

| Giá trị | Ý nghĩa | Bắt buộc/Option |
|---------|---------|-----------------|
| `nosniff` | Chặn MIME-type sniffing | ✅ Bắt buộc |

**Chống attack:** MIME Confusion Attacks

---

#### 🔒 X-Frame-Options

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

| Giá trị | Ý nghĩa | Khi nào dùng |
|---------|---------|--------------|
| `DENY` | Không cho phép iframe | Site không cần embed |
| `SAMEORIGIN` | Chỉ cho phép same origin | **Khuyến nghị** |
| `ALLOW-FROM uri` | Cho phép domain cụ thể | ⚠️ Deprecated |

**Chống attack:** Clickjacking

---

#### 🔒 X-XSS-Protection (Legacy)

```nginx
add_header X-XSS-Protection "1; mode=block" always;
```

| Giá trị | Ý nghĩa |
|---------|---------|
| `0` | Tắt XSS filter |
| `1` | Bật filter (sanitize) |
| `1; mode=block` | Bật filter (block page) |

**⚠️ Note:** Legacy header, modern browsers dùng CSP

---

#### 🔒 Content-Security-Policy (CSP) ⭐ Quan trọng

```nginx
add_header Content-Security-Policy 
    "default-src 'self'; 
     script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net; 
     style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
     font-src 'self' https://fonts.gstatic.com data:;
     img-src 'self' data: https:;
     connect-src 'self';
     frame-ancestors 'self';
     base-uri 'self';
     form-action 'self';" always;
```

| Directive | Ý nghĩa | Giá trị mẫu | Bắt buộc/Option |
|-----------|---------|-------------|-----------------|
| `default-src` | Default policy cho tất cả | `'self'` | ✅ Bắt buộc |
| `script-src` | JavaScript sources | `'self' https://cdn.com` | ⚙️ Override default |
| `style-src` | CSS sources | `'self' 'unsafe-inline'` | ⚙️ Override default |
| `img-src` | Image sources | `'self' data: https:` | ⚙️ Override default |
| `font-src` | Font sources | `'self' https://fonts.com` | ⚙️ Override default |
| `connect-src` | AJAX, WebSocket sources | `'self'` | ⚙️ Override default |
| `frame-ancestors` | Ai được phép embed site | `'self'` | ⚙️ Option |
| `base-uri` | Giới hạn `<base>` tag | `'self'` | ⚙️ Option |
| `form-action` | Form submit destinations | `'self'` | ⚙️ Option |

**CSP Keywords:**

| Keyword | Ý nghĩa | Khuyến nghị |
|---------|---------|-------------|
| `'self'` | Same origin | ✅ An toàn |
| `'none'` | Chặn tất cả | ✅ An toàn nhất |
| `'unsafe-inline'` | Cho phép inline scripts/styles | ⚠️ Không an toàn |
| `'unsafe-eval'` | Cho phép eval() | ⚠️ Tránh nếu có thể |
| `https:` | Bất kỳ HTTPS source | ⚙️ OK cho images |
| `data:` | Data URIs | ⚙️ OK cho fonts/images |

**🎯 Ví dụ CSP strict:**

```nginx
# CSP rất strict - chặn hầu hết XSS
add_header Content-Security-Policy 
    "default-src 'none'; 
     script-src 'self'; 
     style-src 'self'; 
     img-src 'self'; 
     font-src 'self'; 
     connect-src 'self'; 
     frame-ancestors 'none';" always;
```

**Chống attack:** XSS (Cross-Site Scripting), Data Injection

---

#### 🔒 Referrer-Policy

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

| Policy | Ý nghĩa | Khi nào dùng |
|--------|---------|--------------|
| `no-referrer` | Không gửi referrer | Rất private |
| `same-origin` | Chỉ gửi cho same-origin | Cân bằng |
| `strict-origin` | Chỉ gửi origin cho HTTPS | ✅ Khuyến nghị |
| `strict-origin-when-cross-origin` | Full URL same-origin, origin cross-origin | **Default tốt** |

**Chống:** Information Leakage

---

#### 🔒 Permissions-Policy (Feature Policy)

```nginx
add_header Permissions-Policy 
    "geolocation=(self), 
     microphone=(), 
     camera=(), 
     payment=(), 
     usb=()" always;
```

| Feature | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `geolocation=(self)` | Same origin | Chỉ site mình dùng GPS |
| `camera=()` | None | Tắt camera |
| `microphone=()` | None | Tắt mic |
| `payment=(self)` | Same origin | Payment API |
| `usb=()` | None | Tắt USB API |

**Các feature khác:** `accelerometer`, `gyroscope`, `magnetometer`, `fullscreen`, `picture-in-picture`

---

#### 🔒 Hide Server Info

```nginx
server_tokens off;
more_clear_headers 'Server';
more_clear_headers 'X-Powered-By';
```

| Directive | Ý nghĩa | Bắt buộc/Option | Module |
|-----------|---------|-----------------|--------|
| `server_tokens off` | Ẩn Nginx version | ✅ Khuyến nghị | Core |
| `more_clear_headers` | Xóa header | ⚙️ Option | headers-more-nginx-module |

**Chống:** Information Disclosure

---

## 7. ModSecurity WAF

### 📋 Mô tả
Web Application Firewall bảo vệ chống SQL Injection, XSS, LFI/RFI...

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `modsecurity` | Bật/tắt WAF | `on` / `off` | ✅ Bắt buộc (nếu cài) | ✅ Luôn bật production |
| `modsecurity_rules_file` | File cấu hình rules | Path | ✅ Bắt buộc | `/etc/nginx/modsec/main.conf` |

### 💡 ModSecurity Rules

**OWASP Core Rule Set (CRS):**
```bash
# Clone CRS
cd /etc/nginx/modsec
git clone https://github.com/coreruleset/coreruleset.git
cd coreruleset
mv crs-setup.conf.example crs-setup.conf
```

**Main config (`/etc/nginx/modsec/main.conf`):**
```nginx
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset/crs-setup.conf
Include /etc/nginx/modsec/coreruleset/rules/*.conf
```

### 🎯 Điều chỉnh ModSecurity

| Mode | Mô tả | Khi nào dùng |
|------|-------|--------------|
| `SecRuleEngine On` | Block threats | ✅ Production |
| `SecRuleEngine DetectionOnly` | Log only, không block | Testing/tuning |
| `SecRuleEngine Off` | Tắt hoàn toàn | Troubleshooting |

**⚠️ False Positives:**
- Test kỹ ở mode `DetectionOnly` trước
- Tạo whitelist rules cho các false positives
- Monitor logs: `/var/log/modsec/audit.log`

---

## 8. Rate Limiting & DDoS Protection

### 📋 Mô tả
Áp dụng rate limits đã định nghĩa, giới hạn request size và timeouts.

```nginx
limit_req zone=sample_ratelimit burst=20 nodelay;
limit_conn sample_conn 10;

client_body_buffer_size 1M;
client_max_body_size 10M;
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;

client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 15;
send_timeout 10;
```

### 📊 Chi tiết Rate Limiting

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `limit_req` | Áp dụng request limit | `zone=name burst=N` | ⚙️ Option | ✅ Khuyến nghị |
| `burst` | Cho phép burst requests | `20` | ⚙️ Option | Tùy use case |
| `nodelay` | Không delay requests trong burst | - | ⚙️ Option | Tốt cho UX |
| `limit_conn` | Giới hạn connections | `10` | ⚙️ Option | ✅ Khuyến nghị |

**💡 Burst explained:**
```
Rate: 10r/s, Burst: 20
→ Cho phép 30 requests tức thời (10 + 20 burst)
→ Sau đó chỉ 10r/s

nodelay: Xử lý burst ngay lập tức
delay: Trải đều burst theo rate
```

### 📊 Request Size Limits

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Khuyến nghị |
|---------|---------|---------|-----------------|-------------|
| `client_body_buffer_size` | Buffer size cho request body | `1M` | ⚙️ Option | Default: 8k/16k |
| `client_max_body_size` | Max upload size | `10M` | ⚙️ Option | Điều chỉnh theo nhu cầu |
| `client_header_buffer_size` | Buffer cho headers | `1k` | ⚙️ Option | Default: 1k |
| `large_client_header_buffers` | Buffer cho large headers | `4 8k` | ⚙️ Option | 4 buffers x 8KB |

**🎯 Khi nào điều chỉnh:**
- **File upload**: Tăng `client_max_body_size` lên `100M` hoặc `500M`
- **API với large payload**: Tăng `client_body_buffer_size`
- **Cookies nhiều**: Tăng `large_client_header_buffers`

### 📊 Timeouts

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option | Impact |
|---------|---------|---------|-----------------|--------|
| `client_body_timeout` | Timeout đọc request body | `12s` | ⚙️ Option | Chống slowloris |
| `client_header_timeout` | Timeout đọc headers | `12s` | ⚙️ Option | Chống slowloris |
| `keepalive_timeout` | Keepalive connection | `15s` | ⚙️ Option | Balance performance/resources |
| `send_timeout` | Timeout gửi response | `10s` | ⚙️ Option | Nếu client chậm |

**⚠️ Slowloris Attack:**
- Attacker gửi headers/body rất chậm
- Giữ connections open, exhaust server
- Fix: Giảm timeouts (10-15s)

---

## 9. Logging

### 📋 Mô tả
Cấu hình access log và error log.

```nginx
access_log /var/log/nginx/sample.com.vn-access.log combined;
error_log /var/log/nginx/sample.com.vn-error.log warn;

log_format security_log '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       '$request_time $upstream_response_time '
                       '$pipe $connection_requests';
```

### 📊 Chi tiết tham số

| Tham số | Ý nghĩa | Giá trị | Bắt buộc/Option |
|---------|---------|---------|-----------------|
| `access_log` | File log access | Path + format | ⚙️ Option |
| `error_log` | File log errors | Path + level | ⚙️ Option |
| `log_format` | Định nghĩa format | Name + pattern | ⚙️ Option |

### 📊 Log Levels

| Level | Mô tả | Khi nào dùng |
|-------|-------|--------------|
| `debug` | Debug information | Development |
| `info` | Informational | Verbose logging |
| `notice` | Notice | Normal events |
| `warn` | Warnings | **Khuyến nghị production** |
| `error` | Errors only | Production minimal |
| `crit` | Critical | Production critical only |

### 💡 Built-in Log Formats

| Format | Mô tả |
|--------|-------|
| `combined` | Apache combined log format (standard) |
| `main` | Similar to combined |
| Custom | Tự định nghĩa |

### 🎯 Custom Log Variables

| Variable | Ý nghĩa |
|----------|---------|
| `$remote_addr` | Client IP |
| `$remote_user` | HTTP auth username |
| `$time_local` | Timestamp |
| `$request` | Full request line |
| `$status` | Response status code |
| `$body_bytes_sent` | Response size (bytes) |
| `$http_referer` | Referer header |
| `$http_user_agent` | User agent |
| `$request_time` | Request processing time |
| `$upstream_response_time` | Backend response time |

### 🎯 Disable Logging (Performance)

```nginx
# Tắt log cho static files
location ~* \.(jpg|jpeg|png|gif|css|js)$ {
    access_log off;
    log_not_found off;
}
```

---

## 10. Location Blocks

### 📋 Mô tả
Các location blocks xử lý different types of requests.

### 🔒 A. Deny Sensitive Files

```nginx
# Hidden files
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# Backup and source files
location ~* \.(bak|config|sql|fla|psd|ini|log|sh|inc|swp|dist|md|yml|yaml|env)$ {
    deny all;
}

# Specific directories
location ~ ^/(vendor|storage|database|tests|node_modules|\.git)/ {
    deny all;
}
```

| Pattern | Mô tả | Bắt buộc/Option |
|---------|-------|-----------------|
| `~ /\.` | Regex match hidden files | ✅ Bắt buộc |
| `~*` | Case-insensitive regex | ✅ Bắt buộc |
| `^/` | Start with | ✅ Bắt buộc |
| `deny all` | Chặn tất cả | ✅ Bắt buộc |

**⚠️ Quan trọng:** Luôn chặn `.git`, `.env`, `vendor/`, `node_modules/`

---

### 📦 B. Static Files Optimization

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot|webp|avif)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    add_header X-Content-Type-Options "nosniff" always;
    access_log off;
    log_not_found off;
}
```

| Directive | Ý nghĩa | Giá trị | Khuyến nghị |
|-----------|---------|---------|-------------|
| `expires` | Cache duration | `1y`, `30d`, `max` | `1y` cho assets versioned |
| `Cache-Control` | Cache policy | `public`, `private`, `immutable` | `public, immutable` tốt nhất |

**Cache-Control values:**
- `public`: Có thể cache bởi CDN/proxy
- `private`: Chỉ cache ở browser
- `immutable`: File không đổi (cho fingerprinted assets)
- `max-age=31536000`: Cache 1 năm (seconds)

---

### 🌐 C. Main Application Routing

```nginx
location / {
    try_files $uri $uri/ @proxy;
}

location @proxy {
    proxy_pass http://sample_backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # ... more proxy settings
}
```

| Directive | Ý nghĩa | Bắt buộc/Option |
|-----------|---------|-----------------|
| `try_files` | Thử serve file, không có thì fallback | ⚙️ Option |
| `@proxy` | Named location | ⚙️ Option |
| `proxy_pass` | Backend URL | ✅ Bắt buộc (nếu proxy) |

**📊 Proxy Headers (Bắt buộc cho reverse proxy):**

| Header | Ý nghĩa | Bắt buộc/Option |
|--------|---------|-----------------|
| `Host` | Original hostname | ✅ Bắt buộc |
| `X-Real-IP` | Client real IP | ✅ Khuyến nghị |
| `X-Forwarded-For` | Proxy chain IPs | ✅ Khuyến nghị |
| `X-Forwarded-Proto` | HTTP/HTTPS | ✅ Bắt buộc |
| `X-Forwarded-Host` | Original host | ⚙️ Option |
| `X-Forwarded-Port` | Original port | ⚙️ Option |

---

### 🐘 D. PHP-FPM Configuration

```nginx
location ~ \.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```

| Directive | Ý nghĩa | Bắt buộc/Option |
|-----------|---------|-----------------|
| `try_files $uri =404` | Security: Prevent script injection | ✅ Bắt buộc |
| `fastcgi_pass` | PHP-FPM socket/port | ✅ Bắt buộc |
| `SCRIPT_FILENAME` | File path to execute | ✅ Bắt buộc |
| `include fastcgi_params` | Standard FastCGI params | ✅ Bắt buộc |

**⚠️ Security:** `try_files $uri =404` prevents PHP from executing uploaded files as PHP!

---

### 🔌 E. API Endpoint

```nginx
location /api/ {
    limit_req zone=sample_ratelimit burst=10 nodelay;
    
    # CORS headers
    add_header Access-Control-Allow-Origin "https://sample.com.vn" always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
    
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    
    proxy_pass http://sample_backend;
}
```

**📊 CORS Headers:**

| Header | Ý nghĩa | Bắt buộc/Option |
|--------|---------|-----------------|
| `Access-Control-Allow-Origin` | Allowed origins | ✅ Bắt buộc (CORS) |
| `Access-Control-Allow-Methods` | Allowed HTTP methods | ✅ Bắt buộc (CORS) |
| `Access-Control-Allow-Headers` | Allowed request headers | ✅ Bắt buộc (CORS) |
| `Access-Control-Max-Age` | Preflight cache time | ⚙️ Option |

**⚠️ CORS Security:**
- **Never use** `*` in production: `Access-Control-Allow-Origin "*"`
- Use specific domain: `"https://yourdomain.com"`
- Or check Origin dynamically

---

### ❤️ F. Health Check

```nginx
location /health {
    access_log off;
    return 200 "healthy\n";
    add_header Content-Type text/plain;
}
```

**Mục đích:** Load balancer/monitoring checks

---

### 🚫 G. Error Pages

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

location = /404.html {
    internal;
    root /var/www/errors;
}
```

| Directive | Ý nghĩa | Bắt buộc/Option |
|-----------|---------|-----------------|
| `error_page` | Map error code to page | ⚙️ Option |
| `internal` | Chỉ internal redirects | ✅ Khuyến nghị |

---

## 📋 Checklist Triển Khai

### ✅ Trước khi deploy

- [ ] **Thay domain:** `sample.com.vn` → domain thực
- [ ] **Thay paths:** Document root, log paths
- [ ] **Backend upstream:** Cấu hình đúng IP:port
- [ ] **SSL certificates:** Tạo và kiểm tra paths
- [ ] **DH parameters:** `openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096`
- [ ] **DNS records:** A/AAAA records trỏ đúng
- [ ] **Firewall:** Mở port 80, 443

### ✅ Cài đặt dependencies

```bash
# Ubuntu/Debian
apt-get update
apt-get install -y nginx-full nginx-extras certbot

# ModSecurity
apt-get install -y libnginx-mod-security2

# Headers-more module (optional)
apt-get install -y libnginx-mod-http-headers-more-filter
```

### ✅ Tạo SSL certificate (Let's Encrypt)

```bash
# Webroot method
certbot certonly --webroot -w /var/www/certbot \
  -d sample.com.vn -d www.sample.com.vn \
  --email admin@sample.com.vn --agree-tos

# Standalone method (cần stop nginx trước)
certbot certonly --standalone \
  -d sample.com.vn -d www.sample.com.vn \
  --email admin@sample.com.vn --agree-tos

# Auto-renewal
systemctl enable certbot.timer
```

### ✅ Test cấu hình

```bash
# Syntax check
nginx -t

# Nếu OK, reload
systemctl reload nginx

# Hoặc restart
systemctl restart nginx
```

### ✅ Monitoring & Testing

```bash
# Check logs real-time
tail -f /var/log/nginx/sample.com.vn-access.log
tail -f /var/log/nginx/sample.com.vn-error.log
tail -f /var/log/modsec/audit.log

# Test SSL
curl -I https://sample.com.vn

# Test headers
curl -I -k https://sample.com.vn | grep -i "strict-transport-security\|x-frame-options"
```

### ✅ Security Testing Tools

| Tool | Purpose | URL |
|------|---------|-----|
| SSL Labs | SSL/TLS grade | https://www.ssllabs.com/ssltest/ |
| Security Headers | Check security headers | https://securityheaders.com/ |
| Mozilla Observatory | Overall security | https://observatory.mozilla.org/ |
| CSP Evaluator | Validate CSP | https://csp-evaluator.withgoogle.com/ |

**Target Scores:**
- SSL Labs: **A+**
- Security Headers: **A+**
- Mozilla Observatory: **A+**

---

## 🎯 Fine-tuning Recommendations

### 🔧 Theo loại ứng dụng

#### Static Website
```nginx
# Không cần upstream
# Không cần PHP-FPM
# Tăng cache time
expires 1y;
```

#### WordPress/PHP App
```nginx
# Cần PHP-FPM
# Giữ nguyên cache cho static
# Có thể cần tăng client_max_body_size cho media upload
client_max_body_size 100M;
```

#### API Server
```nginx
# Rate limit nghiêm hơn
limit_req zone=sample_ratelimit burst=5 nodelay;

# CORS cụ thể
add_header Access-Control-Allow-Origin "https://app.yourdomain.com";

# Không cần static file caching
```

#### High Traffic Site
```nginx
# Tăng keepalive
keepalive_timeout 65;

# Tăng worker connections (nginx.conf)
worker_connections 4096;

# Tăng rate limits
rate=100r/s;
```

---

## 🔐 Security Levels

### Level 1: Basic (Minimum)
- ✅ HTTPS redirect
- ✅ SSL certificate valid
- ✅ Basic headers (HSTS, X-Frame-Options)

### Level 2: Standard (Recommended)
- ✅ All security headers
- ✅ TLS 1.2+ only
- ✅ Rate limiting
- ✅ Hide sensitive files

### Level 3: Advanced (Production)
- ✅ ModSecurity WAF
- ✅ Strict CSP
- ✅ OCSP Stapling
- ✅ DH Parameters

### Level 4: Paranoid (Critical Systems)
- ✅ TLS 1.3 only
- ✅ Strict CSP (no unsafe-inline)
- ✅ Multiple WAF layers
- ✅ DDoS mitigation (Cloudflare/AWS Shield)
- ✅ Geo-blocking
- ✅ Client certificate authentication

---

## 📚 Tài liệu tham khảo

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [ModSecurity Handbook](https://www.feistyduck.com/books/modsecurity-handbook/)
- [Content Security Policy Reference](https://content-security-policy.com/)

---

## 🆘 Troubleshooting Common Issues

### ❌ "Too many redirects"
**Nguyên nhân:** HTTP → HTTPS redirect loop

**Fix:**
```nginx
# Kiểm tra X-Forwarded-Proto nếu behind load balancer
if ($http_x_forwarded_proto = "http") {
    return 301 https://$server_name$request_uri;
}
```

### ❌ "502 Bad Gateway"
**Nguyên nhân:** Backend không chạy hoặc sai port

**Fix:**
```bash
# Kiểm tra backend
netstat -tlnp | grep 8080
systemctl status your-app

# Check nginx error log
tail -f /var/log/nginx/error.log
```

### ❌ "SSL certificate problem"
**Nguyên nhân:** Cert không match domain, hoặc expired

**Fix:**
```bash
# Kiểm tra cert
openssl x509 -in /etc/letsencrypt/live/sample.com.vn/cert.pem -text -noout

# Renew cert
certbot renew --dry-run
certbot renew
```

### ❌ "403 Forbidden"
**Nguyên nhân:** Permissions, hoặc ModSecurity block

**Fix:**
```bash
# Check file permissions
ls -la /var/www/sample.com.vn/

# Check ModSecurity log
tail -f /var/log/modsec/audit.log

# Temporary disable ModSec
modsecurity off;
```

### ❌ "Client request body too large"
**Nguyên nhân:** File upload lớn hơn `client_max_body_size`

**Fix:**
```nginx
client_max_body_size 100M;  # Tăng lên
```

---

## 🎓 Training Tips

### Cho người mới:
1. **Bắt đầu đơn giản:** Tắt ModSecurity, dùng HTTP trước
2. **Từng bước:** Thêm HTTPS → Headers → WAF
3. **Test từng thay đổi:** `nginx -t` sau mỗi edit
4. **Đọc logs:** Logs là bạn tốt nhất
5. **Backup config:** Luôn backup trước khi sửa

### Lab Exercise:
1. Deploy basic HTTP vhost
2. Add Let's Encrypt SSL
3. Add security headers từng cái
4. Test với SSL Labs
5. Add rate limiting
6. Add ModSecurity
7. Tune false positives

---

**📝 Document Version:** 1.0  
**👤 Author:** Expert System Engineer  
**📅 Last Updated:** October 2025  
**🏷️ Tags:** nginx, security, ssl, https, waf, modsecurity, production

---

