# 🚀 Hướng Dẫn Triển Khai Nginx Reverse Proxy - antm.lab.vn

> **Domain:** antm.lab.vn  
> **Architecture:** Reverse Proxy  
> **Date:** October 23, 2025  
> **Security Level:** Production-ready A+ SSL Grade

---

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc Reverse Proxy](#kiến-trúc-reverse-proxy)
3. [Giải Thích Rate Limiting](#giải-thích-rate-limiting)
4. [Yêu Cầu Hệ Thống](#yêu-cầu-hệ-thống)
5. [Chuẩn Bị](#chuẩn-bị)
6. [Cài Đặt Dependencies](#cài-đặt-dependencies)
7. [Cấu Hình DNS](#cấu-hình-dns)
8. [Tạo SSL Certificate](#tạo-ssl-certificate)
9. [Triển Khai Nginx Config](#triển-khai-nginx-config)
10. [Kiểm Tra & Testing](#kiểm-tra--testing)
11. [Monitoring & Maintenance](#monitoring--maintenance)
12. [Troubleshooting](#troubleshooting)

---

## Tổng Quan

File cấu hình `antm.lab.vn.conf` là **Reverse Proxy** production-ready với:

✅ **Reverse Proxy** - Forward tất cả requests tới backend  
✅ **HTTPS bắt buộc** - Auto-redirect từ HTTP  
✅ **TLS 1.2/1.3** - Strong cipher suites  
✅ **Security Headers** - HSTS, CSP, X-Frame-Options, etc.  
✅ **ModSecurity WAF** - OWASP CRS protection  
✅ **Rate Limiting** - 2-tier protection (request + connection)  
✅ **Health Checks** - `/health` và `/status` endpoints  
✅ **Custom Error Pages** - Branded error handling

---

## Kiến Trúc Reverse Proxy

### 🏗️ **Luồng Request Flow:**

```
Internet → Nginx (antm.lab.vn:443) → Backend (127.0.0.1:8080)
            │                            │
            ├─ SSL Termination           ├─ Application Logic
            ├─ Security Headers          ├─ Business Logic
            ├─ WAF Filtering             ├─ Database Access
            ├─ Rate Limiting             └─ Response Generation
            └─ Proxy Headers
```

### 📊 **Trách nhiệm từng layer:**

| Layer | Nginx (Reverse Proxy) | Backend Application |
|-------|----------------------|---------------------|
| **SSL/TLS** | ✅ Terminate SSL | ❌ HTTP only |
| **Security Headers** | ✅ Add headers | ❌ Không cần |
| **WAF** | ✅ ModSecurity | ❌ Protected |
| **Rate Limiting** | ✅ Control traffic | ❌ Không lo |
| **Static Files** | ❌ Backend xử lý | ✅ Serve files |
| **Business Logic** | ❌ N/A | ✅ Core logic |
| **Database** | ❌ N/A | ✅ Data access |

### 🎯 **Lợi ích Reverse Proxy:**

1. **Centralized Security** - Nginx làm security gateway
2. **SSL Offloading** - Backend không cần handle SSL
3. **Load Balancing** - Dễ dàng scale thêm backends
4. **Caching** (optional) - Cache ở Nginx layer
5. **Hide Backend** - Client không biết backend structure

---

## Giải Thích Rate Limiting

### 🔄 **Tại sao cần 2 bước?**

Rate limiting trong Nginx có **2 bước bắt buộc**:

#### **BƯỚC 1: Định nghĩa Zones (Global Level)**

```nginx
# Đặt NGOÀI server block (http context)
limit_req_zone $binary_remote_addr zone=antm_ratelimit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=antm_conn:10m;
```

**Nhiệm vụ:**
- Tạo **shared memory** (vùng nhớ dùng chung)
- Đặt **tên zone** để reference sau
- Cấp phát **RAM** (10MB = ~160,000 IPs)
- Định nghĩa **tracking key** (`$binary_remote_addr` = IP client)
- Set **rate cơ bản** (10 requests/second)

**Tại sao ở ngoài?**
- Shared memory phải khai báo ở **global level**
- Nhiều server blocks có thể **dùng chung** 1 zone
- Nginx cần biết **trước** khi start workers

#### **BƯỚC 2: Áp dụng Rate Limiting (Server Level)**

```nginx
# Đặt TRONG server block
server {
    limit_req zone=antm_ratelimit burst=20 nodelay;
    limit_conn antm_conn 10;
}
```

**Nhiệm vụ:**
- **Reference zone** đã tạo (theo tên)
- Set **tham số bổ sung** (burst, nodelay)
- **Áp dụng rules** cho location cụ thể
- **Enforce limits** lên traffic thực tế

**Tại sao trong server?**
- Mỗi server/location có thể **customize** khác nhau
- Linh hoạt theo **endpoint** (web vs API)

### 📊 **So sánh 2 loại Rate Limiting:**

| Aspect | Request Limiting | Connection Limiting |
|--------|------------------|---------------------|
| **Directive** | `limit_req_zone` / `limit_req` | `limit_conn_zone` / `limit_conn` |
| **Giới hạn** | Requests/giây | Connections đồng thời |
| **Ví dụ** | 10 req/s | 10 concurrent connections |
| **Use case** | Chống request flooding | Chống connection exhaustion |
| **Burst** | Có (flexible) | Không (strict) |

### 🎯 **Ví dụ thực tế:**

```nginx
# Client IP: 192.168.1.100

# REQUEST LIMITING:
# - Cho phép 10 requests/giây
# - Burst 20 requests
# → Request 1-30: OK (10 normal + 20 burst)
# → Request 31+: 503 Service Unavailable

# CONNECTION LIMITING:
# - Cho phép 10 connections đồng thời
# → Connection 1-10: OK
# → Connection 11: Reject ngay lập tức
```

### ⚙️ **Tùy chỉnh Rate Limiting:**

| Scenario | Rate | Burst | Connections |
|----------|------|-------|-------------|
| **Public website** | 10r/s | 20 | 10 |
| **API endpoint** | 5r/s | 10 | 5 |
| **Admin panel** | 3r/s | 5 | 3 |
| **High traffic** | 50r/s | 100 | 50 |

### 📈 **Memory Calculator:**

```
Zone Size → Max IPs tracked
1 MB  → ~16,000 IPs
5 MB  → ~80,000 IPs
10 MB → ~160,000 IPs
20 MB → ~320,000 IPs
50 MB → ~800,000 IPs
```

**Entry size:** ~64 bytes (IP + count + timestamp + metadata)

---

## Yêu Cầu Hệ Thống

### Minimum Requirements

| Component | Version | Notes |
|-----------|---------|-------|
| **OS** | Ubuntu 20.04+ / Debian 11+ | CentOS/RHEL cũng OK |
| **Nginx** | 1.18+ | Với module ModSecurity |
| **RAM** | 2GB+ | 4GB khuyến nghị |
| **Disk** | 20GB+ | Cho logs và certificates |
| **CPU** | 2 cores+ | Tùy traffic |

### Network Requirements

- ✅ Port 80 (HTTP) open
- ✅ Port 443 (HTTPS) open
- ✅ DNS A record trỏ về server IP
- ✅ Firewall cho phép inbound traffic

---

## Chuẩn Bị

### 1. Thông Tin Cần Có

Trước khi bắt đầu, chuẩn bị:

```bash
# Domain information
DOMAIN="antm.lab.vn"
SUBDOMAIN="www.antm.lab.vn"  # Optional

# Server IP
SERVER_IP="YOUR_SERVER_IP"

# Email for Let's Encrypt
ADMIN_EMAIL="admin@antm.lab.vn"

# Backend application
BACKEND_HOST="127.0.0.1"
BACKEND_PORT="8080"
```

### 2. Cấu Trúc Thư Mục

```bash
# Tạo cấu trúc thư mục
sudo mkdir -p /var/www/antm.lab.vn/public
sudo mkdir -p /var/www/certbot
sudo mkdir -p /var/www/errors
sudo mkdir -p /var/log/nginx
sudo mkdir -p /etc/nginx/ssl
sudo mkdir -p /etc/nginx/modsec
```

### 3. Permissions

```bash
# Set ownership
sudo chown -R www-data:www-data /var/www/antm.lab.vn
sudo chown -R www-data:www-data /var/www/errors

# Set permissions
sudo chmod -R 755 /var/www/antm.lab.vn
sudo chmod -R 755 /var/www/errors
```

---

## Cài Đặt Dependencies

### Ubuntu/Debian

```bash
# Update package list
sudo apt-get update

# Install Nginx with modules
sudo apt-get install -y nginx-full nginx-extras

# Install ModSecurity
sudo apt-get install -y libnginx-mod-security2

# Install headers-more module (for hiding server info)
sudo apt-get install -y libnginx-mod-http-headers-more-filter

# Install Certbot for Let's Encrypt
sudo apt-get install -y certbot python3-certbot-nginx

# Install useful tools
sudo apt-get install -y curl wget openssl
```

### CentOS/RHEL

```bash
# Enable EPEL repository
sudo yum install -y epel-release

# Install Nginx
sudo yum install -y nginx

# Install ModSecurity
sudo yum install -y mod_security mod_security_crs

# Install Certbot
sudo yum install -y certbot python3-certbot-nginx
```

### Verify Installation

```bash
# Check Nginx version
nginx -v

# Check modules loaded
nginx -V 2>&1 | grep -o with-http_[a-z_]*_module

# Verify ModSecurity
ls -la /etc/nginx/modsec/
```

---

## Cấu Hình DNS

### 1. Tạo DNS Records

Tại nhà cung cấp DNS (Cloudflare, Route53, etc.):

```dns
# A record cho root domain
antm.lab.vn.        IN  A   YOUR_SERVER_IP

# A record cho www subdomain
www.antm.lab.vn.    IN  A   YOUR_SERVER_IP

# AAAA record (nếu có IPv6)
antm.lab.vn.        IN  AAAA   YOUR_IPv6_ADDRESS
```

### 2. Kiểm Tra DNS Propagation

```bash
# Check DNS resolution
dig antm.lab.vn +short
dig www.antm.lab.vn +short

# Or using nslookup
nslookup antm.lab.vn
nslookup www.antm.lab.vn

# Wait for DNS propagation (có thể mất 5-60 phút)
```

### 3. Verify từ Server

```bash
# Ping từ server
ping -c 3 antm.lab.vn

# Curl test
curl -I http://antm.lab.vn
```

---

## Tạo SSL Certificate

### Option 1: Let's Encrypt (Khuyến nghị - FREE)

#### Method 1: Webroot (Nginx đang chạy)

```bash
# Tạo webroot directory
sudo mkdir -p /var/www/certbot

# Request certificate
sudo certbot certonly --webroot \
  -w /var/www/certbot \
  -d antm.lab.vn \
  -d www.antm.lab.vn \
  --email admin@antm.lab.vn \
  --agree-tos \
  --no-eff-email

# Certificates sẽ được lưu tại:
# /etc/letsencrypt/live/antm.lab.vn/fullchain.pem
# /etc/letsencrypt/live/antm.lab.vn/privkey.pem
# /etc/letsencrypt/live/antm.lab.vn/chain.pem
```

#### Method 2: Standalone (Nginx tắt)

```bash
# Stop Nginx
sudo systemctl stop nginx

# Request certificate
sudo certbot certonly --standalone \
  -d antm.lab.vn \
  -d www.antm.lab.vn \
  --email admin@antm.lab.vn \
  --agree-tos \
  --no-eff-email

# Start Nginx lại
sudo systemctl start nginx
```

#### Auto-renewal Setup

```bash
# Enable auto-renewal timer
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# Test renewal (dry-run)
sudo certbot renew --dry-run

# Check timer status
sudo systemctl status certbot.timer

# Manual renewal (if needed)
sudo certbot renew
```

### Option 2: Self-Signed Certificate (Testing Only)

```bash
# Tạo self-signed certificate (valid 365 days)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/antm.lab.vn.key \
  -out /etc/nginx/ssl/antm.lab.vn.crt \
  -subj "/C=VN/ST=HCM/L=HCM/O=ANTM/CN=antm.lab.vn"

# Update config to use self-signed cert
# ssl_certificate /etc/nginx/ssl/antm.lab.vn.crt;
# ssl_certificate_key /etc/nginx/ssl/antm.lab.vn.key;
```

### Tạo DH Parameters (Bắt buộc)

```bash
# Generate strong DH parameters (mất 5-10 phút)
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096

# Nếu muốn nhanh hơn (2048-bit)
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048

# Set permissions
sudo chmod 644 /etc/nginx/ssl/dhparam.pem
```

---

## Triển Khai Nginx Config

### 1. Cấu Hình ModSecurity

```bash
# Copy ModSecurity config mẫu
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf

# Chỉnh sửa config
sudo nano /etc/nginx/modsec/modsecurity.conf

# Thay đổi dòng này:
SecRuleEngine DetectionOnly
# Thành:
SecRuleEngine On

# Tải OWASP Core Rule Set
cd /etc/nginx/modsec
sudo git clone https://github.com/coreruleset/coreruleset.git
cd coreruleset
sudo mv crs-setup.conf.example crs-setup.conf

# Tạo main config file
sudo nano /etc/nginx/modsec/main.conf
```

**Nội dung `/etc/nginx/modsec/main.conf`:**

```nginx
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset/crs-setup.conf
Include /etc/nginx/modsec/coreruleset/rules/*.conf
```

### 2. Tạo Error Pages

```bash
# Tạo custom error page
sudo nano /var/www/errors/404.html
```

**404.html:**
```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Không Tìm Thấy Trang</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; }
        h1 { font-size: 50px; color: #e74c3c; }
        p { font-size: 20px; color: #555; }
        a { color: #3498db; text-decoration: none; }
    </style>
</head>
<body>
    <h1>404</h1>
    <p>Trang bạn tìm kiếm không tồn tại.</p>
    <p><a href="/">← Quay về trang chủ</a></p>
</body>
</html>
```

**Tương tự cho 403.html và 50x.html**

### 3. Copy Nginx Config

```bash
# Copy file cấu hình
sudo cp antm.lab.vn.conf /etc/nginx/sites-available/antm.lab.vn.conf

# Tạo symbolic link
sudo ln -s /etc/nginx/sites-available/antm.lab.vn.conf /etc/nginx/sites-enabled/

# Xóa default config (optional)
sudo rm /etc/nginx/sites-enabled/default
```

### 4. Điều Chỉnh Config

**Các giá trị CẦN THAY ĐỔI trong `antm.lab.vn.conf`:**

```nginx
# 1. Document root - thay đổi theo project
root /var/www/antm.lab.vn/public;

# 2. Backend upstream - IP và port thực tế
upstream antm_backend {
    least_conn;
    server 127.0.0.1:8080;  # ← THAY ĐỔI NÀY
}

# 3. Client max body size (nếu cần upload file lớn)
client_max_body_size 100M;  # Mặc định: 10M

# 4. PHP-FPM socket (nếu dùng PHP)
# Uncomment section PHP-FPM và điều chỉnh version
fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;

# 5. Internal network cho /status endpoint
allow 192.168.1.0/24;  # ← THAY ĐỔI theo mạng LAN
```

### 5. Test Config

```bash
# Kiểm tra syntax
sudo nginx -t

# Nếu có lỗi, check error message
# Thường gặp:
# - SSL certificate không tồn tại
# - dhparam.pem chưa tạo
# - ModSecurity config sai
```

### 6. Apply Config

```bash
# Reload Nginx (zero downtime)
sudo systemctl reload nginx

# Hoặc restart (nếu cần)
sudo systemctl restart nginx

# Enable Nginx auto-start
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

---

## Kiểm Tra & Testing

### 1. Basic Connectivity

```bash
# Test HTTP redirect
curl -I http://antm.lab.vn
# Expected: 301 Moved Permanently → https://antm.lab.vn

# Test HTTPS
curl -I https://antm.lab.vn
# Expected: 200 OK (hoặc status từ backend)
```

### 2. Security Headers Check

```bash
# Check all security headers
curl -I https://antm.lab.vn | grep -i "strict-transport-security\|x-frame-options\|x-content-type\|content-security"

# Expected output:
# strict-transport-security: max-age=63072000; includeSubDomains; preload
# x-frame-options: SAMEORIGIN
# x-content-type-options: nosniff
# content-security-policy: ...
```

### 3. SSL/TLS Testing

```bash
# Test SSL connection
openssl s_client -connect antm.lab.vn:443 -servername antm.lab.vn

# Check certificate info
echo | openssl s_client -connect antm.lab.vn:443 -servername antm.lab.vn 2>/dev/null | openssl x509 -noout -dates -subject

# Test specific TLS version
openssl s_client -connect antm.lab.vn:443 -tls1_2
openssl s_client -connect antm.lab.vn:443 -tls1_3
```

### 4. Online Security Scanners

#### SSL Labs Test
```
URL: https://www.ssllabs.com/ssltest/analyze.html?d=antm.lab.vn
Target Score: A+
```

#### Security Headers Test
```
URL: https://securityheaders.com/?q=https://antm.lab.vn
Target Score: A+
```

#### Mozilla Observatory
```
URL: https://observatory.mozilla.org/analyze/antm.lab.vn
Target Score: A+
```

### 5. Rate Limiting Test

```bash
# Test rate limiting (should see 503 after 30 requests)
for i in {1..35}; do curl -I https://antm.lab.vn 2>&1 | grep HTTP; done

# Expected: First 30 succeed, then 503 Service Unavailable
```

### 6. ModSecurity WAF Test

```bash
# Test SQL Injection detection
curl "https://antm.lab.vn/?id=1' OR '1'='1"
# Expected: 403 Forbidden (blocked by ModSecurity)

# Test XSS detection
curl "https://antm.lab.vn/?search=<script>alert('xss')</script>"
# Expected: 403 Forbidden

# Check ModSecurity log
sudo tail -f /var/log/modsec/audit.log
```

### 7. Health Check Endpoints

```bash
# Public health check
curl https://antm.lab.vn/health
# Expected: healthy

# Nginx status (chỉ từ internal network)
curl http://127.0.0.1/status
# Expected: Active connections, requests info
```

### 8. Error Pages

```bash
# Test 404 page
curl -I https://antm.lab.vn/nonexistent-page
# Expected: 404 Not Found + custom page

# Test backend down (stop backend service)
# Expected: 502/503 + custom error page
```

---

## Monitoring & Maintenance

### 1. Log Monitoring

```bash
# Watch access log real-time
sudo tail -f /var/log/nginx/antm.lab.vn-access.log

# Watch error log
sudo tail -f /var/log/nginx/antm.lab.vn-error.log

# ModSecurity audit log
sudo tail -f /var/log/modsec/audit.log

# Filter for errors only
sudo tail -f /var/log/nginx/antm.lab.vn-error.log | grep error
```

### 2. Log Rotation

```bash
# Nginx log rotation config
sudo nano /etc/logrotate.d/nginx
```

**Nội dung:**
```
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 3. Performance Monitoring

```bash
# Check Nginx processes
ps aux | grep nginx

# Check connections
sudo netstat -tlnp | grep nginx
# Or
sudo ss -tlnp | grep nginx

# Check resource usage
sudo systemctl status nginx

# Top processes
top -p $(pgrep -d',' nginx)
```

### 4. SSL Certificate Monitoring

```bash
# Check certificate expiry
echo | openssl s_client -servername antm.lab.vn -connect antm.lab.vn:443 2>/dev/null | openssl x509 -noout -dates

# Days until expiration
echo | openssl s_client -servername antm.lab.vn -connect antm.lab.vn:443 2>/dev/null | openssl x509 -noout -enddate

# Auto-renewal check
sudo certbot renew --dry-run
```

### 5. Backup Configuration

```bash
# Backup Nginx config
sudo tar -czf nginx-backup-$(date +%Y%m%d).tar.gz \
  /etc/nginx/sites-available/ \
  /etc/nginx/sites-enabled/ \
  /etc/nginx/nginx.conf \
  /etc/nginx/ssl/

# Backup certificates
sudo tar -czf letsencrypt-backup-$(date +%Y%m%d).tar.gz \
  /etc/letsencrypt/

# Backup to remote server (optional)
scp nginx-backup-*.tar.gz user@backup-server:/backups/
```

---

## Troubleshooting

### ❌ Problem: "502 Bad Gateway"

**Nguyên nhân:** Backend không chạy hoặc sai cấu hình upstream

**Giải pháp:**
```bash
# 1. Kiểm tra backend đang chạy
sudo netstat -tlnp | grep 8080
# Hoặc
sudo ss -tlnp | grep 8080

# 2. Start backend nếu chưa chạy
sudo systemctl start your-backend-service

# 3. Check Nginx error log
sudo tail -50 /var/log/nginx/antm.lab.vn-error.log

# 4. Test backend directly
curl http://127.0.0.1:8080

# 5. Check firewall
sudo ufw status
sudo iptables -L
```

---

### ❌ Problem: "Too many redirects"

**Nguyên nhân:** Redirect loop HTTP → HTTPS

**Giải pháp:**
```bash
# Nếu behind load balancer/proxy, thêm vào HTTPS server block:
if ($http_x_forwarded_proto = "http") {
    return 301 https://$server_name$request_uri;
}

# Reload Nginx
sudo systemctl reload nginx
```

---

### ❌ Problem: "SSL certificate problem"

**Nguyên nhân:** Certificate không match domain hoặc expired

**Giải pháp:**
```bash
# 1. Check certificate
sudo openssl x509 -in /etc/letsencrypt/live/antm.lab.vn/cert.pem -text -noout

# 2. Renew certificate
sudo certbot renew

# 3. Force renew
sudo certbot renew --force-renewal

# 4. Check certificate files exist
ls -la /etc/letsencrypt/live/antm.lab.vn/
```

---

### ❌ Problem: "403 Forbidden" (Không phải WAF)

**Nguyên nhân:** File permissions hoặc index file không tồn tại

**Giải pháp:**
```bash
# 1. Check file permissions
ls -la /var/www/antm.lab.vn/public/

# 2. Fix ownership
sudo chown -R www-data:www-data /var/www/antm.lab.vn/

# 3. Fix permissions
sudo chmod -R 755 /var/www/antm.lab.vn/

# 4. Tạo index file test
echo "Hello antm.lab.vn" | sudo tee /var/www/antm.lab.vn/public/index.html

# 5. Check Nginx user
ps aux | grep nginx
```

---

### ❌ Problem: "ModSecurity blocking legitimate requests"

**Nguyên nhân:** False positives

**Giải pháp:**
```bash
# 1. Check ModSecurity log
sudo tail -100 /var/log/modsec/audit.log

# 2. Tìm rule ID gây false positive (ví dụ: 942100)
# Trong log sẽ thấy: [id "942100"]

# 3. Disable rule cụ thể
sudo nano /etc/nginx/modsec/custom-rules.conf

# Thêm dòng:
SecRuleRemoveById 942100

# 4. Include custom rules vào main.conf
sudo nano /etc/nginx/modsec/main.conf

# Thêm dòng:
Include /etc/nginx/modsec/custom-rules.conf

# 5. Reload Nginx
sudo systemctl reload nginx

# 6. Hoặc tạm thời chuyển sang DetectionOnly mode
SecRuleEngine DetectionOnly  # Chỉ log, không block
```

---

### ❌ Problem: "Rate limiting blocking normal users"

**Nguyên nhân:** Rate limit quá thấp

**Giải pháp:**
```bash
# Điều chỉnh trong config
limit_req_zone $binary_remote_addr zone=antm_ratelimit:10m rate=20r/s;  # Tăng lên 20r/s
limit_req zone=antm_ratelimit burst=50 nodelay;  # Tăng burst lên 50

# Reload
sudo systemctl reload nginx
```

---

### ❌ Problem: "Nginx won't start"

**Nguyên nhân:** Syntax error hoặc port conflict

**Giải pháp:**
```bash
# 1. Test config
sudo nginx -t

# 2. Check detailed error
sudo journalctl -xeu nginx

# 3. Check port 80/443 đã bị chiếm chưa
sudo netstat -tlnp | grep :80
sudo netstat -tlnp | grep :443

# 4. Kill process nếu cần
sudo kill -9 <PID>

# 5. Start lại
sudo systemctl start nginx
```

---

### ❌ Problem: "Client max body size error"

**Nguyên nhân:** Upload file lớn hơn giới hạn

**Giải pháp:**
```bash
# Tăng limit trong config
client_max_body_size 100M;  # Hoặc 500M

# Reload
sudo systemctl reload nginx
```

---

## 📊 Performance Tuning (Optional)

### Nginx Worker Processes

```bash
sudo nano /etc/nginx/nginx.conf
```

```nginx
# Số worker processes = số CPU cores
worker_processes auto;

# Số connections mỗi worker
events {
    worker_connections 4096;  # Tăng từ 1024
    use epoll;  # Linux-specific
}

# File descriptors
worker_rlimit_nofile 8192;
```

### Enable Gzip Compression

```nginx
# Trong http block
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml text/javascript 
           application/json application/javascript application/xml+rss 
           application/rss+xml font/truetype font/opentype 
           application/vnd.ms-fontobject image/svg+xml;
gzip_disable "msie6";
```

### Enable HTTP/2

```nginx
# Already enabled trong config
listen 443 ssl http2;
```

---

## 📝 Checklist Sau Khi Deploy

- [ ] ✅ DNS records đã trỏ đúng IP
- [ ] ✅ SSL certificate đã tạo và valid
- [ ] ✅ DH parameters đã generate
- [ ] ✅ ModSecurity đã cài và config
- [ ] ✅ Nginx config syntax OK (`nginx -t`)
- [ ] ✅ Nginx đã restart/reload
- [ ] ✅ HTTP → HTTPS redirect hoạt động
- [ ] ✅ Security headers đầy đủ
- [ ] ✅ SSL Labs test: **A+**
- [ ] ✅ Security Headers test: **A+**
- [ ] ✅ Rate limiting hoạt động
- [ ] ✅ Health check endpoint accessible
- [ ] ✅ Error pages hiển thị đúng
- [ ] ✅ Backend proxy hoạt động
- [ ] ✅ Logs đang ghi vào đúng file
- [ ] ✅ Auto-renewal SSL đã enable
- [ ] ✅ Backup config đã thực hiện
- [ ] ✅ Monitoring setup (optional)

---

## 🔗 Quick Reference Commands

```bash
# Test config
sudo nginx -t

# Reload (zero downtime)
sudo systemctl reload nginx

# Restart
sudo systemctl restart nginx

# Check status
sudo systemctl status nginx

# View logs
sudo tail -f /var/log/nginx/antm.lab.vn-access.log
sudo tail -f /var/log/nginx/antm.lab.vn-error.log

# Renew SSL
sudo certbot renew

# Test SSL
curl -I https://antm.lab.vn

# Check certificate expiry
sudo certbot certificates
```

---

## 📞 Support & Resources

- **Nginx Documentation:** https://nginx.org/en/docs/
- **ModSecurity:** https://github.com/SpiderLabs/ModSecurity
- **Let's Encrypt:** https://letsencrypt.org/
- **SSL Labs:** https://www.ssllabs.com/ssltest/
- **Security Headers:** https://securityheaders.com/

---

## 📜 Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2025-10-23 | 1.0 | Initial deployment for antm.lab.vn |

---

**🎉 Chúc bạn triển khai thành công!**

Nếu gặp vấn đề, tham khảo section [Troubleshooting](#troubleshooting) hoặc check logs.
