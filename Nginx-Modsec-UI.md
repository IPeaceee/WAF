**Link tham khảo**: https://launchpad.net/~ondrej/+archive/waf/nginx

### Bước 1: Cài đặt Nginx
```
root@waf:~# add-apt-repository ppa:ondrej/nginx-mainline -y
root@waf:~# apt update
root@waf:~# apt install nginx-core nginx-common nginx nginx-full git -y
```
### Bước 2: Thiết lập khoá Reposistory
```
root@waf:~# vim /etc/apt/sources.list.d/ondrej-waf-nginx-jammy.list 
#deb http://ppa.launchpad.net/ondrej/nginx-mainline/waf focal main
#deb-src http://ppa.launchpad.net/ondrej/nginx-mainline/waf focal main
```
Sau khi khoá xong tiến hành update để kiểm tra
```
root@waf:~# apt update
```
### Bước 4: Thiết lập vị trí chứa source
**Lưu ý**: Tạo ra vị trí chứa source của ứng dụng để dễ dàng quản lý

Thực hiện tạo folder và phân quyền cho folder chứa source
```
root@waf:~# mkdir -p /usr/local/src/nginx
root@waf:~# chown _apt:root /usr/local/src/nginx
root@waf:~# cd /usr/local/src/nginx/
```

### Bước 5: Download and Compile ModSecurity v3 từ Source Code
Thực hiện cài đặt các thư viện cần thiết trước khi compile ModSec
```
root@waf:/usr/local/src/nginx# sudo apt install dpkg-dev
root@waf:/usr/local/src/nginx# cd /usr/local/src/
root@waf:/usr/local/src# apt install -y gcc make build-essential autoconf automake libtool libcurl4-openssl-dev liblua5.3-dev libfuzzy-dev ssdeep gettext pkg-config libpcre3 libpcre3-dev libxml2 libxml2-dev libcurl4 libgeoip-dev libyajl-dev doxygen
```

Thực hiện clone từ github vào folder chứa source vừa tạo /usr/local/src/ModSecurity/
```
root@waf:/usr/local/src# git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity /usr/local/src/ModSecurity/
root@waf:/usr/local/src# cd /usr/local/src/ModSecurity/
```

Cài đặt các gói cần thiết cho submodule
```
root@waf:/usr/local/src/ModSecurity# git submodule init
root@waf:/usr/local/src/ModSecurity# git submodule update
```

Kiểm tra các file đang có trong /usr/local/src/ModSecurity/
```
root@waf:/usr/local/src/ModSecurity# ls
AUTHORS   build.sh      doc       LICENSE                       modsecurity.pc.in  src    unicode.mapping
bindings  CHANGES       examples  Makefile.am                   others             test
build     configure.ac  headers   modsecurity.conf-recommended  README.md          tools
```

Thực hiện compile từ source vừa download
```
root@waf:/usr/local/src/ModSecurity# ./build.sh
root@waf:/usr/local/src/ModSecurity# ./configure
root@waf:/usr/local/src/ModSecurity# make -j2
root@waf:/usr/local/src/ModSecurity# make install
```

**Lưu ý:** 
 + Modsecurity sẽ được cài đặt ở vị trí /usr/local/src/ModSecurity
 + Lỗi này không cần quan tâm "fatal: No names found, cannot describe anything."
 + "make -j2" nghĩa là dùng 2CPU để biên dịch


### Bước 6: Download and Compile ModSecurity Nginx Connector từ Source Code
```
root@waf:/usr/local/src/ModSecurity# cd /usr/local/src/
root@waf:/usr/local/src# git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
root@waf:/usr/local/src# cd /usr/local/src/nginx
root@waf:/usr/local/src/nginx# wget http://nginx.org/download/nginx-1.27.1.tar.gz
root@waf:/usr/local/src/nginx# tar -xvzf nginx-1.27.1.tar.gz
root@waf:/usr/local/src/nginx# cd nginx-1.27.1
```

Cài đặt các gói phụ thuộc để biên dịch nginx
```
# apt install uuid-dev
```


Tiếp tục ta sẽ thực hiện compile module "ModSecurity Nginx Connector". Ta sử dụng --with-compat sẽ tạo module dạng binary-compatible tương thích với Nginx hiện hành. Bạn cần lưu ý thư mục "objs" sau khi biên dịch module

```
# ./configure --with-compat --add-dynamic-module=/usr/local/src/ModSecurity-nginx
# ./configure --with-compat --add-dynamic-module=/usr/local/src/ModSecurity-nginx --without-http_gzip_module
# make modules
# cp objs/ngx_http_modsecurity_module.so /usr/share/nginx/modules/
# vi /etc/nginx/nginx.conf
```


### Bước 7: Thực hiện load Modsecurity với Nginx

Tiến hành vào file **/etc/nginx/nginx.conf** thực hiện thêm cấu hình `load_module modules/ngx_http_modsecurity_module.so;`và `modsecurity_rules_file /etc/nginx/modsec/main.conf;`
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

#Load ModSecurity Module
load_module modules/ngx_http_modsecurity_module.so;

http {
      modsecurity_rules_file /etc/nginx/modsec/main.conf;
}
```
Tạo thư mục và các file cấu hình cho modsecurity đã được mô tả ở trên:
```
root@waf:~# sudo mkdir /etc/nginx/modsec/
root@waf:~# sudo cp /usr/local/src/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
```

### Bước 8: Thiết lập cấu hình cho modsecurity.conf
- Tiến hành vào file `vim /etc/nginx/modsec/modsecurity.conf` và thực hiện tinh chỉnh một số nội dung sau:

- Dòng số 7 chuyển "DetectionOnly" -> "On" mục tiêu để bậc phát hiện và thực hiện hành động
```
#SecRuleEngine DetectionOnly
SecRuleEngine On
```
- Thực hiện tìm SecAuditLogParts và chỉnh sửa "ABIJDEFHZ" -> "ABCEFHJKZ"
```
#SecAuditLogParts ABIJDEFHZ
SecAuditLogParts ABCEFHJKZ
```
Thực hiện tìm SecResponseBodyAccess và chỉnh sửa
```
# SecResponseBodyAccess On
SecResponseBodyAccess Off
```
Thực hiện tìm thiết lập log format JSON cho audit mode
```
SecAuditLogType Serial
SecAuditLogFormat JSON
SecAuditLog /var/log/modsec_audit.log
```

**Lưu ý**: SecAuditLogParts **ABCEFHJKZ** có ý nghĩa như bản mô tả bên dưới:
| Letter | Description | 
| -------- | -------- |
| A | Audit log header (**bắt buộc**). |
| B | Request headers. |
| C | Request body. |
| D | Dành cho các tiêu đề phản hồi trung gian; chưa được thực thi. |
| E | response body, chỉ xuất hiện nếu ModSecurity được định cấu hình intercept response bodies và audit log engine được định cấu hình để ghi lại nội dung đó |
| F | final response headers (*excluding the Date and Server headers*) |
| G | Dành riêng the actual response body; not implemented yet. |
| H | Audit log trailer (**bắt buộc**) |
| I | Đây là phần thay thế cho C.Nó ghi nhận log giống với C hoàn toàn ngoại trừ khi multipart/form-data encoding được sử dụng. Trong trường hợp này, nó sẽ ghi lại phần thân giả mạo application/x-www-form-urlencoded có chứa thông tin về các tham số chứ không phải về các tệp. Điều này rất hữu ích nếu không muốn có các tệp lớn được lưu trữ trong audit log của mình. |
| J | Phần này chứa thông tin về các tệp được tải lên bằng multipart/form-data encoding. |
| K | Phần này chứa danh sách đầy đủ các rule khớp (mỗi quy tắc trên một dòng) theo thứ tự chúng được khớp|
| Z | Final boundary (*Ranh giới kết thúc*), biểu thị sự kết thúc của mục (**bắt buộc**). |

#### Bước 9: Tạo ra file "/etc/nginx/modsec/main.conf" và include file "/etc/nginx/modsec/modsecurity.conf"
Tiến hành vào file `vim /etc/nginx/modsec/main.conf` và thực hiện thêm
```
Include /etc/nginx/modsec/modsecurity.conf
```
```
root@waf:~# cp /usr/local/src/ModSecurity/unicode.mapping /etc/nginx/modsec/
root@waf:~# sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

root@waf:~# sudo systemctl restart nginx
root@waf:~# sudo netstat -ltunp | grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      31466/nginx: master
```

### Bước 10: Download và Kích hoạt OWASP Core Rule Set. Đường link https://github.com/coreruleset/coreruleset
```
root@waf:~# wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v4.6.0.tar.gz
root@waf:~# tar xf v4.6.0.tar.gz 
root@waf:~# sudo mv coreruleset-4.6.0/ /etc/nginx/modsec/
root@waf:~# cd /etc/nginx/modsec/coreruleset-4.6.0
root@waf:~# cp -v  crs-setup.conf.example crs-setup.conf.example.backup
root@waf:/etc/nginx/modsec/coreruleset-4.6.0# mv crs-setup.conf.example crs-setup.conf
```
Tại ra file `/etc/nginx/modsec/main.conf` và thực hiện chỉnh sửa thêm các dòng dưới đây vào file (`vim /etc/nginx/modsec/main.conf`)
*Chú ý thay đổi CRS version*
```
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset-4.6.0/crs-setup.conf
Include /etc/nginx/modsec/coreruleset-4.6.0/rules/*.conf
```
**Lưu ý**: File "/etc/nginx/modsec/main.conf" sẽ include 3 file là "main.conf" "crs-setup.conf" và các file rule để nhận dạng tấn công

Thực hiện kiểm tra và restart lại dịch vụ nginx
```
root@waf:~# sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
root@waf:~# sudo systemctl restart nginx
```
### Bước 11: Thực hiện đấu nối với web lỗi để tiến hành thử nghiệm
Tạo file dvwa.conf ở vị trí `/etc/nginx/sites-available` với nội dung:
```
server {
        listen       *:80;
        server_name  dvwa.antm.lab 192.168.255.153;
        modsecurity on;

        location / {
                proxy_pass http://192.168.255.155;
        }
}
```

Thực hiện enable site vừa đấu nối và tiến hành reload lại dịch vụ
```
cd /etc/nginx/sites-enable/
ln -s /etc/nginx/sites-available/dvwa.conf dvwa.conf
nginx -t
nginx -s reload
```


### Bước 12: Thực hiện cài đặt NginxUI để dễ dàng quản lý cấu hình bằng giao diện 

Thực hiện cài đặt nginxUI for linux theo tài liệu: https://github.com/0xJacky/nginx-ui#script-for-linux
```
bash <(curl -L -s https://raw.githubusercontent.com/0xJacky/nginx-ui/master/install.sh) install
```

Start nginx-ui và cho phép nginxUI khởi động cùng hệ thống sau đó kiểm tra port. The default listening port là 9000
```
systemctl start nginx-ui
systemctl enable nginx-ui
netstat -ltunp | grep 9000
```