- Các lớp bảo vệ trước khi tương tác được với file
```
Một process muốn đọc file

        ↓

1. Linux permission có cho user của process không?
        ↓
2. SELinux có cho domain của process không?
        ↓
Cả hai ALLOW

=> mới đọc được
```
- 2 thằng này nằm trên 2 process khác nhau
```
Chiếm httpd process
≠
đăng nhập được user apache
```
- Hack được thì cũng chỉ nằm trong phạm vi domain của process hiện tại
```
Hack httpd
→ attacker điều khiển process hiện tại
→ vẫn mang user apache + domain httpd_t
```
- Process mới sau khi login SSH với user apache
```
SSH apache
→ hệ thống tạo login session mới
→ tạo shell process mới
→ vẫn là user apache
→ nhưng SELinux domain có thể khác
```
- Flow dễ nhớ
```
Process
  |
  | domain
  v
httpd_t
  |
  | action: read/write/connect...
  v
Resource
  |
  | type
  v
httpd_sys_content_t

        ↓

SELinux Policy

        ↓

ALLOW / DENY
```