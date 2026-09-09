# Guided Execution — NOVA-002

Mode: [guided-execution-mode.md](guided-execution-mode.md). Ticket gốc:
[../plans/01-foundation.md#ticket-nova-002](../plans/01-foundation.md#ticket-nova-002).
Team roster: [../plans/01-foundation.md#team-roster-novacart-engineering](../plans/01-foundation.md#team-roster-novacart-engineering).

Depends on: `NOVA-001` đã hoàn tất — bạn (`nova-ops` hoặc tên bạn đã chọn)
đang SSH vào VPS bằng key, có `wheel`, và đó là user bạn dùng để làm toàn bộ
các bước dưới đây (không còn login bằng `root` nữa).

Môi trường: AlmaLinux 8.9 — dùng `dnf`, `firewalld`, SELinux enforcing.
Không dùng lệnh kiểu Ubuntu (`apt`, `ufw`, `adduser` kiểu Debian).

Đi đúng thứ tự các bước bên dưới. Mỗi bước: đọc "Vì sao", gõ tay lệnh, rồi
**dừng lại giải thích cho tôi bạn hiểu gì trước khi sang bước kế tiếp.**
Tự tay gõ từng lệnh — không dùng script cấu hình có sẵn (đúng constraint
của ticket).

---

## Bước 0 — Kiểm tra trạng thái hiện tại trước khi tạo gì

**Vì sao:** giống hệt lý do ở NOVA-001 Bước 0 — không biết baseline thì
không biết mình đang thêm mới hay đang đè lên thứ đã có sẵn.

```bash
getent group deploy catalog-logs 2>/dev/null || echo "chưa có group nào"
getent passwd dev1 dev2 qa1 2>/dev/null || echo "chưa có user nào"
ls -ld /opt/catalog-service 2>/dev/null || echo "chưa có thư mục"
ls -l /etc/sudoers.d/
sudo -l
```

🛑 Giải thích: lệnh cuối (`sudo -l`, không có `-U`) đang hỏi về quyền sudo
của **ai**? Vì sao nên chạy lệnh này TRƯỚC khi bắt đầu, chứ không phải chỉ
sau khi xong hết mọi thứ?

---

## Bước 1 — Tạo 2 group dùng cho phân quyền theo nhóm

**Vì sao:** ticket yêu cầu `dev1`/`dev2` có chung một bộ quyền (`deploy`,
`catalog-logs`), không phải quyền may đo riêng cho từng người. Group là cách
Linux cấp/thu quyền hàng loạt.

```bash
groupadd deploy
groupadd catalog-logs
getent group deploy catalog-logs
```

🛑 Giải thích: nếu ticket chỉ có 2 người (`dev1`, `dev2`) chứ không phải một
team đang phát triển, dùng group ở đây có còn đáng không, hay setgroup
riêng cho từng người cũng được? Điều gì thay đổi khi team lên 10 người?

---

## Bước 2 — Tạo thư mục scaffold `/opt/catalog-service`

**Vì sao:** Level 1 sẽ deploy `catalog-service` vào đây. Ticket này chỉ
dựng khung thư mục với đúng group + setgid, chưa có ứng dụng thật.

```bash
mkdir -p /opt/catalog-service
chgrp deploy /opt/catalog-service
chmod 2775 /opt/catalog-service
ls -ld /opt/catalog-service
```

🛑 Giải thích: số `2775` tách ra nghĩa là gì (`2` là gì, `775` là gì)? Ticket
đã cảnh báo "setgid trên thư mục chỉ ảnh hưởng file/thư mục MỚI tạo sau đó" —
thử đoán: nếu bạn `chmod 2775` một thư mục ĐÃ có sẵn file cũ bên trong, các
file cũ đó có tự đổi group thành `deploy` không? Vì sao?

---

## Bước 3 — Tạo `dev1`, `dev2`, gán đúng group, KHÔNG gán `wheel`

**Vì sao:** đúng nguyên tắc least-privilege của ticket — 2 developer không
cần sudo để làm việc hàng ngày.

```bash
useradd -m -s /bin/bash dev1
passwd dev1
usermod -aG deploy,catalog-logs dev1

useradd -m -s /bin/bash dev2
passwd dev2
usermod -aG deploy,catalog-logs dev2

id dev1
id dev2
```

🛑 Giải thích: nhìn output `id dev1` — làm sao xác nhận được `dev1` KHÔNG có
`wheel`? Tại Bước 2 của NOVA-001, `usermod -aG wheel nova-ops` chỉ gán 1
group. Ở đây `-aG deploy,catalog-logs` gán 2 group cùng lúc bằng dấu phẩy —
nếu bạn gõ nhầm thành `-G deploy,catalog-logs` (bỏ `-a`) cho `dev1` khi
`dev1` đã có sẵn group phụ nào khác, chuyện gì xảy ra?

---

## Bước 4 — Tạo khung sudoers riêng cho `dev2` (chưa điền lệnh thật)

**Vì sao:** `dev2` là senior dev, backup on-call — ticket muốn `dev2` có
thêm 1 quyền hạn chế qua `sudoers.d`, tách biệt khỏi quyền group. Nhưng
`catalog-service` chưa được deploy (Level 1 chưa chạy) nên chưa biết lệnh
restart/status cụ thể là gì — ticket cố ý để đây là khung TODO.

```bash
visudo -f /etc/sudoers.d/dev2
```

Nội dung file (gõ tay trong editor mở ra):

```
# NOVA-002: khung quyền hạn chế cho dev2 (senior dev, backup on-call).
# Lệnh thật sẽ điền ở 02-application-deployment.md, sau khi chốt chạy
# catalog-service bằng systemd unit hay docker compose.
Cmnd_Alias CATALOG_SVC_TODO = /bin/true
dev2 ALL=(root) NOPASSWD: CATALOG_SVC_TODO
```

Sau khi lưu, verify ngay (đừng đóng session hiện tại trước khi verify):

```bash
sudo -l -U dev2
ls -l /etc/sudoers.d/dev2
sudo -l
```

🛑 Giải thích: vì sao dùng `visudo -f` thay vì mở file đó bằng `vi`/`nano`
trực tiếp? (gợi ý: liên hệ với vai trò của `sshd -t` ở NOVA-001 Bước 5).
Lệnh `sudo -l` cuối cùng (kiểm tra chính bạn, không phải `dev2`) đang phòng
tránh rủi ro gì mà ticket đã ghi trong Constraints?

---

## Bước 5 — Tạo `qa1`: không shell thường, chuẩn bị SSH key

**Vì sao:** QA không cần và không nên có shell tương tác — chỉ cần xem log
qua SFTP. Cấp shell thật cho một account chỉ cần đọc log là thừa quyền.

```bash
useradd -m -s /sbin/nologin qa1
mkdir -p /home/qa1/.ssh
chmod 700 /home/qa1/.ssh
echo "<nội dung public key của qa1>" >> /home/qa1/.ssh/authorized_keys
chmod 600 /home/qa1/.ssh/authorized_keys
chown -R qa1:qa1 /home/qa1/.ssh
```

🛑 Giải thích: `/sbin/nologin` khác gì với việc chỉ đơn giản là không đưa
mật khẩu cho `qa1`? Nếu ai đó lấy được private key tương ứng và thử
`ssh qa1@<vps-ip>` (không phải `sftp`), bạn đoán chuyện gì sẽ xảy ra —
`/sbin/nologin` có chặn được không, chặn ở đâu?

---

## Bước 6 — Khoá `ChrootDirectory` và cấu hình `sshd` cho `qa1`

**Vì sao:** đây là bước dễ tự khoá mình (hoặc khoá `qa1`) nhất trong cả
ticket — OpenSSH có yêu cầu permission rất nghiêm ngặt cho `ChrootDirectory`,
đúng như Technical Notes đã cảnh báo.

Trước tiên, đổi ownership của **chính thư mục `ChrootDirectory`** (không
phải nội dung bên trong nó):

```bash
chown root:root /home/qa1
chmod 755 /home/qa1
ls -ld /home/qa1
ls -ld /home/qa1/.ssh
```

Sau đó thêm block sau vào **cuối** `/etc/ssh/sshd_config` (dùng `vi`/`nano`,
tự tay sửa):

```
Match User qa1
    ChrootDirectory /home/qa1
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

Test và áp dụng — **giữ nguyên session hiện tại đang mở**, giống hệt kỷ luật
ở NOVA-001 Bước 5:

```bash
sshd -t
systemctl restart sshd
```

🛑 Giải thích: sau khi `chown root:root /home/qa1`, `qa1` còn tự ghi được
gì vào thẳng home directory của chính nó không? Vì sao `/home/qa1/.ssh` vẫn
còn thuộc `qa1:qa1` mà không vi phạm yêu cầu "ChrootDirectory phải thuộc
root:root" của OpenSSH — hai thứ này có mâu thuẫn không, vì sao? `Match
User qa1` chỉ áp dụng cho riêng `qa1` — nếu bạn đặt nhầm block này ở **đầu**
file thay vì cuối file, trước cả các directive global khác, rủi ro gì có
thể xảy ra cho những user khác?

---

## Bước 7 — Cấp ACL read-only cho `qa1` trên thư mục log

**Vì sao:** ticket yêu cầu `qa1` đọc được log qua ACL riêng (không phải qua
group `catalog-logs` — nhìn lại Team Roster, `qa1` không nằm trong group
đó), và log thật (`/var/log/catalog-service`) không nằm trong chroot của
`qa1` — cần "mượn" nó vào bằng bind mount.

```bash
mkdir -p /var/log/catalog-service
setfacl -m u:qa1:r-x /var/log/catalog-service
setfacl -d -m u:qa1:r-x /var/log/catalog-service
getfacl /var/log/catalog-service

mkdir -p /home/qa1/logs
mount --bind /var/log/catalog-service /home/qa1/logs
mount -o remount,bind,ro /home/qa1/logs
findmnt /home/qa1/logs
```

Để bind mount này sống sót qua reboot, thêm vào `/etc/fstab`:

```
/var/log/catalog-service  /home/qa1/logs  none  bind,ro  0  0
```

🛑 Giải thích: vì sao không thể chỉ tạo một **symlink** từ
`/home/qa1/logs` trỏ ra `/var/log/catalog-service` thay vì bind mount (gợi
ý: symlink lưu một đường dẫn dạng chữ — điều gì xảy ra với đường dẫn đó một
khi `qa1` đã bị nhốt trong chroot)? Vì sao cần chạy `mount --bind` rồi
**thêm một lệnh remount riêng** với `ro` thay vì làm luôn trong 1 lệnh?
`getfacl` khác `ls -l /var/log/catalog-service` ở điểm nào — vì sao ticket
nhấn mạnh phải dùng `getfacl` để debug quyền ACL?

---

## Bước 8 — Xác nhận quyết định "không tạo account cho Product Owner"

**Vì sao:** ticket yêu cầu quyết định này phải được **ghi lại có lý do**,
không phải im lặng bỏ qua.

```bash
getent passwd | grep -i owner || echo "xác nhận: không có account Product Owner"
```

🛑 Giải thích: viết 1-2 câu (bằng lời bạn) lý do vì sao Product Owner
**không cần** một Linux account, dựa trên Team Roster ở `01-foundation.md`.
Nếu sau này họ cần xem log gấp một lần, cách nào đúng hơn — tạo tạm 1
account rồi xoá, hay có cách khác? (Đây cũng là Reflection Question 4 của
NOVA-002 — trả lời ở đây trước, rồi copy sang phần Reflection Questions.)

---

## Bước 9 — Verify toàn bộ theo Acceptance Criteria của NOVA-002

```bash
getent group deploy catalog-logs
ls -ld /opt/catalog-service
id dev1
id dev2
sudo -l -U dev2
ssh qa1@<vps-ip>              # phải bị từ chối (không có shell thường)
sftp qa1@<vps-ip>             # phải vào được, chỉ thấy đúng thư mục chroot
getfacl /var/log/catalog-service
getent passwd | grep -i owner  # phải không ra dòng nào
```

Đối chiếu với checklist gốc trong `01-foundation.md` (Acceptance Criteria
của NOVA-002) — tick từng dòng, đừng tự tick nếu chưa thật sự chạy lệnh và
nhìn output.

---

## Sau khi xong: Reflection Questions

Quay lại đúng 4 câu hỏi "Reflection Questions (Feynman)" của ticket
`NOVA-002` trong [`plans/01-foundation.md`](../plans/01-foundation.md) và
trả lời bằng lời của bạn. Báo lại đây khi xong từng bước, tôi sẽ review như
một senior xem PR — giống cách đã làm với NOVA-001.
