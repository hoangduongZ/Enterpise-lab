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

## Process là gì, Domain là gì (giải thích Feynman)

- Process: chương trình đang chạy, có PID, có "chủ sở hữu" là 1 Linux user. Khái niệm Linux bình thường, không liên quan SELinux.
- Domain: KHÔNG phải user, KHÔNG phải quyền Linux. Domain là 1 cái nhãn SELinux dán lên process, dùng để tra policy xem process này được làm gì.

- Ẩn dụ dễ nhớ
```
User (Linux)   = thẻ nhân viên, ghi tên bạn là ai (apache, root...)
                 → bảo vệ cửa chính chỉ hỏi "thẻ tên gì"

Domain (SELinux) = bộ đồng phục/vai trò đang mặc (vd: httpd_t)
                 → camera an ninh không quan tâm tên, chỉ nhìn đồng phục
                   để quyết định mở phòng nào, sờ máy nào

=> 2 lớp bảo vệ độc lập: Linux hỏi "user nào", SELinux hỏi "domain nào"
   cả hai đều phải ALLOW mới qua được
```

- Vì sao domain tách rời khỏi user
```
Domain được gán lúc process được khởi tạo/exec
(dựa vào domain transition rule: ai gọi ai, entrypoint nào)
KHÔNG dựa vào tên user

=> cùng user apache:
   - httpd tự start   → domain httpd_t
   - SSH login apache → shell mới → domain khác (vd unconfined_t)

=> domain gắn theo LOẠI CHƯƠNG TRÌNH đang chạy, không gắn theo NGƯỜI chạy nó
```

- Nối lại với phần "hack httpd" ở trên
```
Attacker chiếm process httpd
→ chiếm cái đang chạy, mang sẵn domain httpd_t
→ KHÔNG tự nhiên có credential login SSH user apache
  (chiếm process ≠ có quyền đăng nhập)
→ dù "là" user apache, vẫn bị khóa trong domain httpd_t
  → SELinux chặn theo domain, bất kể Linux permission user apache có cho phép
```

- Tóm gọn 1 câu: Process = ai đang chạy. User = chạy dưới danh nghĩa ai (Linux). Domain = đang khoác vai trò an ninh nào (SELinux) — domain quyết định process chạm được tài nguyên nào, độc lập với việc nó là user gì.