# 🛡️ WAF (Web Application Firewall) Solutions

> Repository chứa các giải pháp WAF opensource, templates cấu hình và hướng dẫn triển khai cho production environment

---

## 📋 Mục Lục

- [Tổng Quan](#-tổng-quan)
- [Các Giải Pháp WAF](#-các-giải-pháp-waf)
  - [BunkerWeb](#1-bunkerweb)
  - [Nginx + ModSecurity](#2-nginx--modsecurity)
- [Templates](#-templates)
- [Upload Rules](#-upload-rules)
- [Cấu Trúc Thư Mục](#-cấu-trúc-thư-mục)
- [So Sánh Giải Pháp](#-so-sánh-giải-pháp)
- [Khuyến Nghị Triển Khai](#-khuyến-nghị-triển-khai)

---

## 🎯 Tổng Quan

Repository này cung cấp:

✅ **Nhiều giải pháp WAF opensource** để lựa chọn theo nhu cầu  
✅ **Templates production-ready** với security best practices  
✅ **Hướng dẫn chi tiết** từ cài đặt đến vận hành  
✅ **Custom rules** cho các use-case cụ thể (file upload, API protection...)  
✅ **Docker-compose configurations** để triển khai nhanh  

---

## 🛡️ Các Giải Pháp WAF

### 1. BunkerWeb

**📍 Thư mục:** `bunkerweb/`

#### 🔍 Mô Tả
BunkerWeb là WAF hiện đại, all-in-one solution được xây dựng trên Nginx với:
- 🐳 **Container-first design** - Tối ưu cho Docker/Kubernetes
- 🎨 **Web UI** quản lý trực quan
- 🔧 **Auto-configuration** với Docker labels
- 📊 **Database backend** (MariaDB) để lưu config
- 🔄 **Auto-renewal SSL** tích hợp Let's Encrypt
- 🚀 **Reverse proxy** với load balancing

#### 📦 Components

| Service | Mô tả | Port |
|---------|-------|------|
| `bunkerweb` | Main WAF proxy | 80, 443 |
| `bw-autoconf` | Auto-configuration engine | - |
| `bw-scheduler` | Task scheduler | - |
| `bw-ui` | Web management UI | 7000 |
| `bw-db` | MariaDB database | 3306 |
| `bw-docker` | Docker socket proxy | 2375 |

#### 🚀 Quick Start

```bash
cd bunkerweb/
docker-compose up -d
```

Truy cập UI: `http://<server-ip>:7000`

#### ⚙️ Configuration

**File:** `docker-compose.yaml`

```yaml
environment:
  - SERVER_NAME=cybersys.blog
  - MULTISITE=yes
  - USE_REVERSE_PROXY=yes
  - REVERSE_PROXY_URL=/
  - REVERSE_PROXY_HOST=https://cybersys.blog
```

**Tùy chỉnh:**
- Thay `SERVER_NAME` bằng domain của bạn
- Cấu hình `REVERSE_PROXY_HOST` trỏ đến backend
- Điều chỉnh `DATABASE_URI` nếu cần

#### 📚 Tài Liệu

- Official Docs: https://docs.bunkerweb.io/
- GitHub: https://github.com/bunkerity/bunkerweb

#### ✅ Ưu Điểm
- ✅ Dễ triển khai với Docker
- ✅ Web UI quản lý trực quan
- ✅ Auto-config cho multi-site
- ✅ Built-in best practices
- ✅ Active development & support

#### ❌ Nhược Điểm
- ❌ Tốn tài nguyên hơn (nhiều containers)
- ❌ Phụ thuộc Docker ecosystem
- ❌ Ít tùy biến hơn ModSecurity thuần

#### 🎯 Khi Nào Dùng
- ✅ Triển khai mới, ưu tiên Docker
- ✅ Cần UI quản lý dễ dùng
- ✅ Multi-site với auto-configuration
- ✅ Team ít kinh nghiệm về WAF

---

### 2. Nginx + ModSecurity

**📍 Thư mục:** `modsec/`

#### 🔍 Mô Tả
Giải pháp WAF mạnh mẽ, linh hoạt với:
- 🔧 **Full control** - Tùy biến sâu mọi aspect
- 📜 **OWASP CRS** - Core Rule Set industry standard
- 🎯 **Performance** - Tối ưu tài nguyên
- 🔍 **Custom rules** - Viết rules theo nhu cầu
- 📊 **NginxUI** - Optional web management

#### 📦 Components

| Component | Mô tả | Required |
|-----------|-------|----------|
| Nginx | Web server/reverse proxy | ✅ Yes |
| ModSecurity 3 | WAF engine | ✅ Yes |
| ModSecurity-nginx | Nginx connector | ✅ Yes |
| OWASP CRS | Rule sets | ✅ Yes |
| NginxUI | Web management (optional) | ⚙️ Optional |

#### 🚀 Quick Start

**Chi tiết trong:** `modsec/Nginx-Modsec-UI.md`

```bash
# 1. Cài đặt Nginx
add-apt-repository ppa:ondrej/nginx-mainline -y
apt update && apt install nginx-full git -y

# 2. Compile ModSecurity v3
cd /usr/local/src
git clone --depth 1 -b v3/master https://github.com/SpiderLabs/ModSecurity
cd ModSecurity
./build.sh && ./configure && make -j2 && make install

# 3. Compile ModSecurity-nginx connector
cd /usr/local/src
git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
wget http://nginx.org/download/nginx-1.27.1.tar.gz
tar xf nginx-1.27.1.tar.gz && cd nginx-1.27.1
./configure --with-compat --add-dynamic-module=/usr/local/src/ModSecurity-nginx
make modules
cp objs/ngx_http_modsecurity_module.so /usr/share/nginx/modules/

# 4. Load module trong nginx.conf
load_module modules/ngx_http_modsecurity_module.so;

# 5. Download OWASP CRS
wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v4.6.0.tar.gz
tar xf v4.6.0.tar.gz
mv coreruleset-4.6.0 /etc/nginx/modsec/
cd /etc/nginx/modsec/coreruleset-4.6.0
mv crs-setup.conf.example crs-setup.conf

# 6. Configure
mkdir -p /etc/nginx/modsec
cp /usr/local/src/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
# Edit: SecRuleEngine On

# 7. Main config
cat > /etc/nginx/modsec/main.conf << EOF
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset-4.6.0/crs-setup.conf
Include /etc/nginx/modsec/coreruleset-4.6.0/rules/*.conf
EOF

# 8. Test & restart
nginx -t
systemctl restart nginx
```

#### ⚙️ Configuration

**Nginx vhost với ModSecurity:**

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    # Enable ModSecurity
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    
    location / {
        proxy_pass http://backend;
    }
}
```

**ModSecurity modes:**

| Mode | Mô tả | Khi nào dùng |
|------|-------|--------------|
| `SecRuleEngine On` | Block threats | ✅ Production |
| `SecRuleEngine DetectionOnly` | Log only | Testing/tuning |
| `SecRuleEngine Off` | Disabled | Troubleshooting |

#### 📚 Tài Liệu

- ModSecurity: https://github.com/SpiderLabs/ModSecurity
- OWASP CRS: https://coreruleset.org/
- Nginx Connector: https://github.com/SpiderLabs/ModSecurity-nginx

#### ✅ Ưu Điểm
- ✅ Hiệu năng cao, ít overhead
- ✅ Linh hoạt tối đa
- ✅ Community lớn, nhiều rules
- ✅ OWASP CRS industry standard
- ✅ Chi phí thấp (không cần containers phức tạp)

#### ❌ Nhược Điểm
- ❌ Phức tạp khi setup từ đầu
- ❌ Cần kiến thức sâu để tune
- ❌ Không có UI mặc định (cần cài NginxUI riêng)
- ❌ False positives cần thời gian điều chỉnh

#### 🎯 Khi Nào Dùng
- ✅ Production cao tải, cần performance
- ✅ Cần tùy biến sâu rules
- ✅ Team có kinh nghiệm về WAF
- ✅ Hạ tầng bare-metal hoặc VM truyền thống
- ✅ Budget hạn chế (không cần license)

---

## 📄 Templates

**📍 Thư mục:** `template/`

### 🔍 Mô Tả
Collection các template Nginx configuration production-ready với full security hardening.

### 📦 Files

| File | Mô tả | Use Case |
|------|-------|----------|
| `nginx-template.conf` | Base template với security headers | General purpose |
| `README-NGINX-CONFIG.md` | **Training guide chi tiết** | 📚 Học tập & Reference |
| `sample.com.vn.conf` | Ví dụ cấu hình hoàn chỉnh | Production example |
| `antm.lab.vn.conf` | Lab/testing config | Development |
| `DEPLOY-antm.lab.vn.md` | Deployment guide | Deployment |

### 🎯 Tính Năng Template

✅ **SSL/TLS Hardening**
- TLS 1.2 & 1.3 only
- Strong cipher suites
- OCSP stapling
- DH parameters
- Perfect Forward Secrecy

✅ **Security Headers (A+ SSL Labs)**
- Strict-Transport-Security (HSTS)
- Content-Security-Policy (CSP)
- X-Frame-Options
- X-Content-Type-Options
- Referrer-Policy
- Permissions-Policy

✅ **Rate Limiting & DDoS Protection**
- Request rate limiting
- Connection limiting
- Burst handling
- Timeout tuning

✅ **ModSecurity Integration**
- WAF enabled
- OWASP CRS rules
- Custom rules support

✅ **Optimization**
- Static file caching
- Gzip compression
- HTTP/2
- Keepalive connections

### 📚 Chi Tiết Training

**File:** `README-NGINX-CONFIG.md` - **400+ lines training document**

Bao gồm:
- 📖 Giải thích chi tiết **TỪNG tham số**
- 📊 Bảng so sánh các options
- 💡 Best practices & recommendations
- 🎯 Use-case specific configurations
- ⚠️ Security warnings & pitfalls
- ✅ Deployment checklist
- 🔧 Troubleshooting guide

**Topics covered:**
1. Rate Limiting Zones
2. Upstream Backend Configuration
3. HTTP to HTTPS Redirect
4. HTTPS Server Block
5. SSL/TLS Configuration
6. Security Headers (chi tiết 10+ headers)
7. ModSecurity WAF
8. Rate Limiting & DDoS
9. Logging
10. Location Blocks (static, PHP, API...)

### 🚀 Quick Usage

```bash
# 1. Copy template
cp template/nginx-template.conf /etc/nginx/sites-available/yourdomain.com

# 2. Customize
vim /etc/nginx/sites-available/yourdomain.com
# - Thay SERVER_NAME
# - Thay SSL paths
# - Thay backend upstream

# 3. Enable site
ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/

# 4. Test & reload
nginx -t
systemctl reload nginx
```

### 🎓 Training Path

Cho System Engineers mới:

1. **Đọc:** `README-NGINX-CONFIG.md` (hiểu lý thuyết)
2. **Lab:** Deploy `antm.lab.vn.conf` trên test server
3. **Practice:** Tự tạo config từ template
4. **Test:** SSL Labs, Security Headers scan
5. **Production:** Apply lên production với monitoring

---

## 📤 Upload Rules

**📍 Thư mục:** `modsec/upload-rules/`

### 🔍 Mô Tả
Custom ModSecurity rules siết chặt upload file, chỉ cho phép:
- Excel: `.xlsx`, `.xls`
- Images: `.jpg`, `.jpeg`, `.png`
- PDF: `.pdf`

### 📦 Files

| File | Mô tả |
|------|-------|
| `REQUEST-910-STRICT-UPLOADS.conf` | Main rule file |
| `REQUEST-910-STRICT-UPLOADS-new.conf` | Updated version |
| `README-STRICT-UPLOADS.md` | Documentation |

### 🎯 Features

✅ **Whitelist Extensions**
- Chỉ cho phép các extension an toàn
- Chặn double-extension attacks (`.pdf.php`)

✅ **Content-Type Validation**
- Bắt buộc khớp extension ↔ Content-Type
- Chặn upload file .exe giả dạng .jpg

✅ **MIME Type Enforcement**
- Kiểm tra multipart/form-data headers
- Validation cho từng file part

✅ **Optional Magic Bytes Check**
- `@inspectFile` với libmagic
- Kiểm tra chữ ký file thật

### 📚 Rules Overview

| Rule ID | Description |
|---------|-------------|
| `999100-999102` | Skip checks nếu không phải multipart/upload |
| `999110` | Block extensions không trong whitelist |
| `999111` | Block dangerous extensions (double-ext) |
| `999120-999124` | Extension ↔ Content-Type mapping |
| `999130` | Content-Type whitelist |
| `999140` | Optional file signature check |

### 🚀 Installation

```bash
# 1. Copy rule file
cp modsec/upload-rules/REQUEST-910-STRICT-UPLOADS.conf \
   /etc/nginx/modsec/coreruleset-4.6.0/rules/

# 2. Include trong main.conf (tự động nếu dùng wildcard)
# Hoặc add explicitly:
echo "Include /etc/nginx/modsec/coreruleset-4.6.0/rules/REQUEST-910-STRICT-UPLOADS.conf" \
  >> /etc/nginx/modsec/main.conf

# 3. Reload
nginx -t && nginx -s reload
```

### 🧪 Testing (PowerShell)

```powershell
# Valid PNG upload
curl.exe -F "file=@C:\path\ok.png;type=image/png" https://example.com/upload -v

# Invalid: JPG with wrong Content-Type (blocked)
curl.exe -F "file=@C:\path\image.jpg;type=application/octet-stream" https://example.com/upload -v

# Invalid: EXE upload (blocked)
curl.exe -F "file=@C:\path\malware.exe;type=application/octet-stream" https://example.com/upload -v

# Valid PDF
curl.exe -F "file=@C:\path\doc.pdf;type=application/pdf" https://example.com/upload -v

# Valid XLSX
curl.exe -F "file=@C:\path\sheet.xlsx;type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" https://example.com/upload -v
```

### 🔧 Customization

**Thêm file type mới (ví dụ: .docx):**

1. Update extension whitelist:
```
(?i)\.(?:xlsx|xls|jpg|jpeg|png|pdf|docx)$
```

2. Thêm Content-Type mapping:
```
SecRule FILES_NAMES "@rx (?i)\.docx$" \
    "id:999125,\
    phase:2,\
    chain,\
    deny,\
    status:403,\
    msg:'Invalid Content-Type for DOCX file'"
    SecRule MULTIPART_PART_HEADERS:Content-Type "!@rx ^application/vnd\.openxmlformats-officedocument\.wordprocessingml\.document$"
```

3. Update allowlist:
```
(?i)^(?:image/(?:jpeg|png)|application/(?:pdf|vnd\.openxmlformats-officedocument\.(?:spreadsheetml\.sheet|wordprocessingml\.document)|vnd\.ms-excel))$
```

---

## 📁 Cấu Trúc Thư Mục

```
waf/
├── README.md                          # 📄 Document này
├── bunkerweb/                         # 🐳 BunkerWeb solution
│   ├── docker-compose.yaml            # Main compose file
│   └── bunker.env                     # Environment variables (optional)
├── modsec/                            # 🔧 Nginx + ModSecurity
│   ├── Nginx-Modsec-UI.md            # 📚 Hướng dẫn cài đặt từ A-Z
│   └── upload-rules/                  # 📤 Custom upload rules
│       ├── REQUEST-910-STRICT-UPLOADS.conf       # Production rule
│       ├── REQUEST-910-STRICT-UPLOADS-new.conf   # Updated version
│       └── README-STRICT-UPLOADS.md              # Documentation
└── template/                          # 📄 Nginx templates
    ├── README-NGINX-CONFIG.md         # 📚 Training guide (400+ lines)
    ├── nginx-template.conf            # Base template
    ├── sample.com.vn.conf             # Example config
    ├── antm.lab.vn.conf              # Lab config
    └── DEPLOY-antm.lab.vn.md         # Deployment guide
```

---

## ⚖️ So Sánh Giải Pháp

| Tiêu Chí | BunkerWeb | Nginx + ModSecurity |
|----------|-----------|---------------------|
| **Độ khó cài đặt** | ⭐ Dễ (docker-compose up) | ⭐⭐⭐ Khó (compile from source) |
| **Performance** | ⭐⭐⭐ Tốt (overhead containers) | ⭐⭐⭐⭐⭐ Xuất sắc (native) |
| **Tài nguyên** | ⭐⭐ Nhiều (6 containers) | ⭐⭐⭐⭐ Ít (1 nginx process) |
| **Quản lý** | ⭐⭐⭐⭐⭐ Web UI trực quan | ⭐⭐ CLI/files (hoặc +NginxUI) |
| **Tùy biến** | ⭐⭐⭐ Tốt (env vars, UI) | ⭐⭐⭐⭐⭐ Hoàn toàn (edit files) |
| **Learning curve** | ⭐⭐ Dễ học | ⭐⭐⭐⭐ Cần kinh nghiệm |
| **Community** | ⭐⭐⭐ Đang phát triển | ⭐⭐⭐⭐⭐ Rất lớn |
| **Rules** | ⭐⭐⭐⭐ Built-in + tự viết | ⭐⭐⭐⭐⭐ OWASP CRS + unlimited custom |
| **Multi-site** | ⭐⭐⭐⭐⭐ Auto-config perfect | ⭐⭐⭐ Manual config |
| **Logging** | ⭐⭐⭐⭐ Centralized | ⭐⭐⭐⭐ File-based (cần ELK stack) |
| **Updates** | ⭐⭐⭐⭐⭐ Docker pull | ⭐⭐ Recompile |
| **Production** | ⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ Battle-tested |

### 💡 Decision Matrix

**Chọn BunkerWeb nếu:**
- ✅ Hạ tầng Docker/K8s
- ✅ Cần deploy nhanh
- ✅ Team ít kinh nghiệm WAF
- ✅ Ưu tiên UI quản lý
- ✅ Multi-site với auto-config
- ✅ Có đủ tài nguyên server

**Chọn Nginx + ModSecurity nếu:**
- ✅ Cần performance tối ưu
- ✅ Bare-metal hoặc VM truyền thống
- ✅ Team có kinh nghiệm
- ✅ Cần tùy biến sâu
- ✅ Hạ tầng hiện tại đã có Nginx
- ✅ Budget/tài nguyên hạn chế

---

## 🎯 Khuyến Nghị Triển Khai

### 🏢 Production Deployment

#### 1️⃣ **Preparation Phase**

- [ ] **Chọn giải pháp** phù hợp với hạ tầng
- [ ] **Setup test environment** giống production
- [ ] **Backup** configuration hiện tại
- [ ] **Document** current setup
- [ ] **Plan rollback** strategy

#### 2️⃣ **Testing Phase**

**BunkerWeb:**
```bash
# Test trên lab environment
cd bunkerweb/
docker-compose up -d

# Monitor logs
docker-compose logs -f bunkerweb

# Test với traffic thật (clone)
# Tune configuration
# Document customizations
```

**Nginx + ModSecurity:**
```bash
# DetectionOnly mode trước
SecRuleEngine DetectionOnly

# Monitor logs 1-2 tuần
tail -f /var/log/modsec/audit.log

# Identify false positives
# Tạo exclusion rules
# Switch to blocking mode
SecRuleEngine On
```

#### 3️⃣ **Deployment Phase**

- [ ] **Deploy ngoài giờ cao điểm**
- [ ] **Enable monitoring** (metrics, logs)
- [ ] **Gradual rollout** (1 server → all)
- [ ] **Keep rollback plan** ready
- [ ] **On-call team** standby

#### 4️⃣ **Post-Deployment**

- [ ] **Monitor false positives** (1-2 tuần)
- [ ] **Tune rules** dựa trên real traffic
- [ ] **Performance testing**
- [ ] **Document final config**
- [ ] **Update runbooks**

### 📊 Monitoring & Alerting

**Metrics cần theo dõi:**
- ✅ WAF block rate
- ✅ False positive rate
- ✅ Response time impact
- ✅ Server resources (CPU, RAM)
- ✅ Log volume

**Alert thresholds:**
- 🚨 Block rate tăng đột biến (DDoS?)
- 🚨 False positive > 1% traffic
- 🚨 Response time tăng > 20%
- 🚨 Error rate 5xx

### 🔐 Security Best Practices

#### ✅ General

1. **Keep updated**
   - WAF software
   - Rule sets (OWASP CRS)
   - Nginx version

2. **Layer defense**
   - WAF không thay thế secure coding
   - Combine với: IDS/IPS, SIEM, AV scanning
   - Network segmentation

3. **Regular audits**
   - Review logs weekly
   - Tune rules monthly
   - Penetration testing quarterly

#### ✅ BunkerWeb Specific

```yaml
# Security hardening
environment:
  - AUTO_LETS_ENCRYPT=yes              # Auto SSL
  - USE_MODSECURITY=yes                # Enable WAF
  - USE_MODSECURITY_CRS=yes           # OWASP CRS
  - BAD_BEHAVIOR_STATUS_CODES=400 401 403 404 405 429 444
  - BAD_BEHAVIOR_BAN_TIME=86400       # Ban 24h
  - BAD_BEHAVIOR_THRESHOLD=10         # 10 bad requests
  - LIMIT_REQ_RATE=20r/s              # Rate limit
  - USE_ANTIBOT=captcha               # Bot protection
```

#### ✅ ModSecurity Specific

```nginx
# modsecurity.conf tuning
SecAuditEngine RelevantOnly           # Chỉ log khi block
SecAuditLogType Serial                # hoặc Concurrent
SecAuditLogFormat JSON                # JSON cho SIEM
SecRequestBodyLimit 13107200          # 12.5MB
SecRequestBodyNoFilesLimit 131072     # 128KB
SecResponseBodyLimit 524288           # 512KB
```

### 🔧 Troubleshooting

#### ❌ False Positives

**Identify:**
```bash
# Check audit log
grep -i "blocked" /var/log/modsec/audit.log | tail -20

# Extract rule IDs
grep -oP 'id "(\d+)"' /var/log/modsec/audit.log | sort | uniq -c
```

**Fix:**
```nginx
# Disable specific rule
SecRuleRemoveById 942100

# Disable for specific URL
<LocationMatch "/api/upload">
    SecRuleRemoveById 942100 942101
</LocationMatch>

# Disable for specific parameter
SecRuleUpdateTargetById 942100 "!ARGS:json_data"
```

#### ❌ Performance Issues

**Diagnose:**
```bash
# Check Nginx performance
nginx -V 2>&1 | grep --color -o --  '--with[^ ]*'

# ModSecurity processing time
grep -oP 'ModSecurity.*?(\d+)' /var/log/nginx/error.log

# Top blocking rules
awk '{print $10}' /var/log/modsec/audit.log | sort | uniq -c | sort -rn | head
```

**Optimize:**
```nginx
# Reduce body inspection
SecRequestBodyNoFilesLimit 65536     # Giảm xuống 64KB

# Disable for static files
<LocationMatch "\.(jpg|css|js)$">
    SecRuleEngine Off
</LocationMatch>

# Parallel processing (nếu có)
SecAuditEngine Concurrent
```

#### ❌ BunkerWeb Container Issues

```bash
# Check container status
docker-compose ps

# Check logs
docker-compose logs bunkerweb | tail -100
docker-compose logs bw-autoconf

# Database connection
docker-compose exec bw-db mysql -u bunkerweb -p

# Restart services
docker-compose restart bunkerweb bw-autoconf
```

---

## 📚 Tài Liệu Tham Khảo

### 🔗 Official Documentation

- **BunkerWeb:** https://docs.bunkerweb.io/
- **ModSecurity:** https://github.com/SpiderLabs/ModSecurity
- **OWASP CRS:** https://coreruleset.org/docs/
- **Nginx:** https://nginx.org/en/docs/

### 📖 Guides & Tutorials

- **ModSecurity Handbook:** https://www.feistyduck.com/books/modsecurity-handbook/
- **Nginx Security:** https://www.nginx.com/blog/mitigating-ddos-attacks-with-nginx-and-nginx-plus/
- **WAF Testing:** https://owasp.org/www-project-web-security-testing-guide/

### 🛠️ Tools

- **SSL Labs:** https://www.ssllabs.com/ssltest/
- **Security Headers:** https://securityheaders.com/
- **Mozilla Observatory:** https://observatory.mozilla.org/
- **CSP Evaluator:** https://csp-evaluator.withgoogle.com/

### 👥 Community

- **ModSecurity Discord:** https://discord.gg/modsecurity
- **OWASP Slack:** https://owasp.org/slack/invite
- **r/netsec:** https://reddit.com/r/netsec

---

## 🤝 Contributing

Contributions welcome! Please:

1. Fork repository
2. Create feature branch
3. Test thoroughly
4. Document changes
5. Submit pull request

---

## 📝 Changelog

### v1.0.0 (2025-11-07)
- ✅ Initial repository structure
- ✅ BunkerWeb docker-compose configuration
- ✅ Nginx + ModSecurity installation guide
- ✅ Production templates with security hardening
- ✅ Custom upload rules
- ✅ Comprehensive documentation

---

## 📄 License

This repository contains configuration examples and documentation.

- **Templates & Configs:** MIT License (use freely)
- **Third-party software:**
  - BunkerWeb: AGPLv3
  - Nginx: 2-clause BSD
  - ModSecurity: Apache 2.0
  - OWASP CRS: Apache 2.0

---

## 👤 Author

**Infrastructure Team**

For questions or support:
- 📧 Email: admin@yourdomain.com
- 💬 Slack: #waf-support
- 📋 Issues: GitHub Issues

---

## 🎓 Training Resources

**Internal training materials:**
- 📚 `template/README-NGINX-CONFIG.md` - Nginx configuration deep-dive
- 📚 `modsec/Nginx-Modsec-UI.md` - ModSecurity installation
- 📚 `modsec/upload-rules/README-STRICT-UPLOADS.md` - Custom rules

**Recommended learning path:**
1. Week 1: Nginx basics + templates
2. Week 2: SSL/TLS configuration
3. Week 3: ModSecurity installation
4. Week 4: OWASP CRS tuning
5. Week 5: Custom rules development
6. Week 6: Production deployment

---

**📌 Last Updated:** November 7, 2025  
**📌 Version:** 1.0.0  
**📌 Status:** Production Ready

---

> 💡 **Pro Tip:** Bắt đầu với BunkerWeb cho quick wins, sau đó migrate sang Nginx + ModSecurity khi cần optimize performance hoặc tùy biến sâu hơn.
