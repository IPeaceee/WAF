# Quy tắc WAF: Chặn upload sai định dạng (Nginx + ModSecurity + CRS)

Bộ quy tắc này siết chặt upload file khi dùng Nginx kết hợp ModSecurity và OWASP Core Rule Set (CRS). Mục tiêu: chỉ cho phép upload các loại sau với đúng phần mở rộng và Content-Type theo từng phần multipart:

- Excel: .xlsx (`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`), .xls (`application/vnd.ms-excel`)
- Ảnh: .jpg/.jpeg (`image/jpeg`), .png (`image/png`)
- PDF: .pdf (`application/pdf`)

File cấu hình chính: `REQUEST-910-STRICT-UPLOADS.conf`

## Nguyên tắc hoạt động

- Chỉ áp dụng cho request `POST|PUT|PATCH` có `Content-Type: multipart/form-data` và có phần file (ModSecurity có biến `FILES_NAMES`).
- Kiểm tra allowlist phần mở rộng: chỉ cho `.xlsx .xls .jpg .jpeg .png .pdf`.
- Chặn các tên file có phần mở rộng nguy hiểm (phòng double-extension kiểu `invoice.pdf.php`).
- Với từng file part, bắt buộc `Content-Type` phải khớp phần mở rộng (ví dụ `.png` phải là `image/png`). Khi biến `MULTIPART_PART_HEADERS` không khả dụng, rule bỏ qua phần kiểm tra này nhưng vẫn kiểm phần mở rộng.
- Chặn mọi `Content-Type` không thuộc allowlist khi biến trên khả dụng.
- Có gợi ý tùy chọn kiểm tra chữ ký file bằng `@inspectFile` (nếu build hỗ trợ libmagic) – mặc định để comment.

## Vị trí và cách include với CRS

Khuyến nghị đặt file theo thứ tự CRS phase 2 (910/920 trước, 930/949 sau). Ví dụ cấu trúc thư mục:

```
/etc/modsecurity/
  crs/
    crs-setup.conf
    rules/
      REQUEST-901-INITIALIZATION.conf
      ...
      REQUEST-910-IP-REPUTATION.conf
      REQUEST-910-STRICT-UPLOADS.conf   <-- (file này)
      REQUEST-920-PROTOCOL-ENFORCEMENT.conf
      ...
```

Cách nạp trong nginx (connector ModSecurity):

```
modsecurity on;
modsecurity_rules_file /etc/modsecurity/modsecurity.conf;
modsecurity_rules_file /etc/modsecurity/crs/crs-setup.conf;
modsecurity_rules_file /etc/modsecurity/crs/rules/REQUEST-901-INITIALIZATION.conf;
# ... các file CRS khác ...
modsecurity_rules_file /etc/modsecurity/crs/rules/REQUEST-910-STRICT-UPLOADS.conf;  # thêm tại đây
modsecurity_rules_file /etc/modsecurity/crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf;
# ...
```

Lưu ý:
- Nếu bạn dùng cơ chế include wildcard `rules/*.conf`, chỉ cần đặt file vào cùng thư mục và đảm bảo thứ tự nạp hợp lý (prefixed 910 để chạy sớm trong phase 2).
- Đảm bảo `SecRequestBodyAccess On`, `SecRequestBodyLimit` đủ lớn và Nginx `client_max_body_size` phù hợp.

## Các rule ID chính

- `999100-999102`: bỏ qua nếu không phải multipart hoặc không có file, hoặc method không phù hợp.
- `999110`: chặn phần mở rộng không nằm trong allowlist.
- `999111`: chặn tên file chứa phần mở rộng nguy hiểm (nghi vấn double-extension).
- `999120-999124`: đối sánh extension ↔ Content-Type theo từng loại (JPEG/PNG/PDF/XLSX/XLS) nếu biến header có sẵn.
- `999130`: allowlist Content-Type tổng hợp cho mọi file part.
- `999140` (comment): gợi ý kiểm tra chữ ký file (libmagic) nếu môi trường hỗ trợ.

## Kiểm thử nhanh (Windows PowerShell)

Dùng `curl.exe` có sẵn trên Windows. Thay URL upload bằng endpoint thật của bạn.

- Upload PNG hợp lệ:

```powershell
curl.exe -F "file=@C:\\path\\ok.png;type=image/png" https://example.com/upload -v
```

- Upload JPG với Content-Type sai (bị chặn 403):

```powershell
curl.exe -F "file=@C:\\path\\image.jpg;type=application/octet-stream" https://example.com/upload -v
```

- Upload EXE nguy hiểm (bị chặn 403):

```powershell
curl.exe -F "file=@C:\\path\\malware.exe;type=application/octet-stream" https://example.com/upload -v
```

- Upload PDF đúng:

```powershell
curl.exe -F "file=@C:\\path\\doc.pdf;type=application/pdf" https://example.com/upload -v
```

- Upload XLSX đúng:

```powershell
curl.exe -F "file=@C:\\path\\sheet.xlsx;type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" https://example.com/upload -v
```

## Lưu ý triển khai

- Một số ứng dụng có thể đặt `Content-Type` của file part không chuẩn (ví dụ `application/octet-stream`). Hãy cập nhật ứng dụng để gửi đúng Content-Type hoặc mở rộng allowlist có kiểm soát.
- Hãy theo dõi audit log để tinh chỉnh nếu có false positive.
- Cân nhắc bật `@inspectFile` nếu connector/modsecurity hỗ trợ libmagic để kiểm tra chữ ký file thật.
- Kết hợp với kiểm tra phía ứng dụng: parse lại MIME, magic bytes, kích thước, quét AV, lưu ngoài webroot.

## Mở rộng allowlist

Nếu cần thêm loại file (ví dụ `.docx`), sửa 2 nơi trong `REQUEST-910-STRICT-UPLOADS.conf`:
- Thêm đuôi vào regex phần mở rộng allowlist.
- Thêm Content-Type tương ứng vào regex allowlist và thêm khối mapping extension ↔ Content-Type.
