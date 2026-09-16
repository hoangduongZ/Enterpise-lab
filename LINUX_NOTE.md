---
description: dùng để note những kiến thức về linux rút ra trong quá trình học và practice
---

# Linux Notes

## 1. `getent` dùng để làm gì trong Linux?

Trước đây, ta từng biết chân lý "mọi thứ đều là tập tin" — ví dụ muốn xem
danh bạ người dùng thì mở cuốn sổ `/etc/passwd`, muốn xem nhóm thì lật sổ
`/etc/group`. Nhưng cách đó có một hạn chế lớn:

- Các file trong `/etc/` chỉ là danh bạ **cục bộ** (nội bộ căn hộ).
- Nếu máy chủ của cậu được kết nối vào một hệ thống mạng doanh nghiệp (như
  dùng LDAP, Active Directory), thông tin tài khoản người dùng không hề nằm
  trên ổ cứng VPS, mà được lưu ở một máy chủ trung tâm khác.
- Khi cậu dùng lệnh `cat /etc/passwd`, cậu sẽ bị mù tịt về những tài khoản
  mạng này.

Lúc này, `getent` xuất hiện: nó không tự đi đọc mò file, mà nó nhìn vào
cuốn sổ cấu hình điều hướng dịch vụ đặt tên (`/etc/nsswitch.conf`). Nó sẽ
hỏi cả cuốn sổ nội bộ `/etc/passwd` **lẫn** máy chủ mạng bên ngoài, rồi gom
toàn bộ kết quả về trình bày trước mắt cậu.

### Ví dụ

- `getent passwd` — liệt kê toàn bộ người dùng mà hệ thống nhận diện được
  (kể cả người dùng cục bộ lẫn người dùng mạng).
- `getent passwd nova-ops` — tìm kiếm riêng người dùng `nova-ops`, trả về
  đúng 1 dòng thông tin định danh tương tự như trong file `/etc/passwd` mà
  không cần phải dùng lệnh `grep` để lọc thủ công.

**Tra cứu địa chỉ mạng / hostname (`hosts`):**

- `getent hosts google.com` — hỏi xem tên miền này tương ứng với địa chỉ IP
  nào, dựa theo thứ tự ưu tiên (`/etc/hosts` trước hay máy chủ DNS trước).

---

## 2. Owner vs group — Linux check quyền theo thứ tự nào?

Không cộng dồn owner + group. Linux chỉ dùng **đúng 1 bộ** quyền:

- UID trùng owner → dùng bộ **owner**, luôn vậy dù user đó có thuộc group
  sở hữu file hay không.
- UID không trùng owner, nhưng GID (kể cả group phụ) trùng group sở hữu
  file → dùng bộ **group**.
- Không khớp gì cả → dùng bộ **other**.

Ví dụ: thư mục `user-A:group-X`. `user-A` không thuộc `group-X` vẫn được
xử lý theo owner bits. `user-B` thuộc `group-X` (không phải owner) thì
dùng group bits.

---

## 3. `chgrp` + setgid (`chmod 2775`) khác `chmod 775` thường ở đâu?

- `chgrp <group> <dir>` — đổi group sở hữu thư mục.
- `775`/`755`... chỉ trả lời "ai được làm gì" (rwx cho owner/group/other).
- Số `2` đứng đầu (`2775`) = **setgid bit** — trả lời câu khác: "file/thư
  mục MỚI tạo bên trong sẽ mang group nào?". Không có setgid → file mới
  mang group CHÍNH của người tạo. Có setgid → file mới tự động mang group
  của thư mục cha, bất kể ai tạo ra nó.
- Hai cơ chế độc lập: setgid quyết định file mới **thuộc group nào**, còn
  chữ số group (`7` hay `5`) quyết định thành viên group đó **có ghi được
  hay không**. Setgid mà group không có quyền ghi (`2755`) thì cũng chẳng
  ai trong group tạo được file mới trong đó.

---

## 4. Khoá `ChrootDirectory` và cấu hình `sshd` cho `ec-qa1`

Biến tài khoản `ec-qa1` thành một người dùng chỉ được phép truyền tệp qua
SFTP và "nhốt" hoàn toàn trong phòng riêng của họ, không thể nhìn thấy bất
kỳ ngóc ngách nào khác của hệ thống.

Thêm block sau vào **cuối** `/etc/ssh/sshd_config` (mở bằng `sudo vi
/etc/ssh/sshd_config` hoặc `sudo nano ...` — file này `root:root`, user
thường không mở ghi được), **sau** block `Match User qa1` đã có từ bản gốc
(không chèn xen vào giữa):

```
Match User ec-qa1
    ChrootDirectory /home/ec-qa1
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

### Khái niệm Chroot (Change Root) là gì?

Bình thường, gốc rễ của toàn bộ căn hộ VPS bắt đầu từ `/` (đại sảnh). Một
người dùng bình thường có thể gõ `cd /etc` hay `cd /var` để đi dạo xem các
tệp tin hệ thống.

`ChrootDirectory /home/ec-qa1` làm một trò ảo thuật: nó nói với hệ thống
rằng đối với riêng anh chàng `ec-qa1`, thư mục `/home/ec-qa1` chính là gốc
`/`.

Anh ta đứng trong phòng mình và nhìn quanh, tưởng rằng căn phòng đó là toàn
bộ vũ trụ. Anh ta không thể `cd ..` để đi lùi ra ngoài đại sảnh được nữa —
**blast radius** của anh ta bị thu hẹp tuyệt đối về con số 0 bên ngoài căn
phòng đó.

### Giải phẫu các quy tắc trong khối cấu hình `sshd_config`

- `Match User ec-qa1` — chỉ áp dụng những luật nghiêm ngặt bên dưới cho
  đúng một mình người dùng `ec-qa1`.
- `ChrootDirectory /home/ec-qa1` — nhốt anh ta trong phòng `/home/ec-qa1`.
- `ForceCommand internal-sftp` — ép buộc anh ta chỉ được dùng giao thức
  truyền tệp SFTP, cấm tuyệt đối việc mở terminal gõ lệnh (`bash`/`sh`).
- `AllowTcpForwarding no` & `X11Forwarding no` — chặn các tính năng chuyển
  tiếp mạng và giao diện đồ hoạ, tránh bị lợi dụng làm cầu nối hack ra
  ngoài.

### Tại sao bắt buộc phải chạy `chown root:root /home/ec-qa1` và `chmod 755`?

Đây chính là "yêu cầu nghiêm ngặt" được nhắc tới trong phần "Vì sao":

Quy luật bất di bất dịch của OpenSSH: thư mục dùng làm `ChrootDirectory`
bắt buộc phải thuộc sở hữu của `root:root`, và không một ai khác (kể cả
chính `ec-qa1`) được phép có quyền ghi (`w`) vào thư mục gốc đó.

Nếu cậu để `ec-qa1` sở hữu thư mục `/home/ec-qa1`
(`chown ec-qa1:ec-qa1`), ông gác cổng `sshd` sẽ coi đó là một lỗ hổng bảo
mật cực lớn (vì người bị nhốt có thể tự sửa cấu hình phòng giam để vượt
ngục) và sẽ từ chối toàn bộ kết nối ngay lập tức.

### `sudo sshd -t` và `sudo systemctl restart sshd` để làm gì?

- `sudo sshd -t` — nhờ ông gác cổng đọc, kiểm tra thử xem cậu có gõ sai cú
  pháp nào trong file cấu hình không, **trước khi** áp dụng thật. Màn hình
  im lặng → an toàn.
- `sudo systemctl restart sshd` — khởi động lại dịch vụ SSH để ông gác
  cổng nạp bộ luật mới vào RAM.
