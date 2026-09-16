# Guided Execution — NOVA-002 (Bản B — electrostore-backend)

Mode: [guided-execution-mode.md](guided-execution-mode.md). Ticket gốc (bản
chất RBAC không đổi, chỉ đổi tên resource):
[../plans/01-foundation.md#ticket-nova-002](../plans/01-foundation.md#ticket-nova-002).
Mapping tên resource đầy đủ:
[../plans/01-foundation.md#nova-002--bản-b-song-song-trên-project-thật-electrostore-backend](../plans/01-foundation.md).
Bản gốc (catalog-service) để đối chiếu: [NOVA-002-guide.md](NOVA-002-guide.md).

Depends on: `NOVA-001` đã hoàn tất — bạn đang SSH vào **cùng 1 VPS** đó bằng
key, có `wheel`. Bản B này chạy song song với NOVA-002 gốc trên cùng VPS,
KHÔNG phải VPS riêng — vì vậy mọi tên user/group/thư mục bên dưới đều có
tiền tố `ec-` để không đụng `dev1`/`dev2`/`qa1`/`deploy`/`catalog-logs` đã
(hoặc sẽ) tồn tại từ NOVA-002 gốc.

Project thật đứng sau bản B: **electrostore-backend**
(https://github.com/hoangduongZ/e-commerce-backend) — Java 21 / Spring Boot
3.5, modular monolith, Postgres 16 + Redis 7 (đã có `docker-compose.yml`
riêng cho local dev), actuator health tại `:8080/actuator/health`, CI có sẵn
ở `.github/workflows/ci.yml`. Đây là project bạn sẽ thật sự đưa lên VPS sau
này — khác `catalog-service` vốn chỉ để học. `ec-dev1`/`ec-dev2`/`ec-qa1` là
account **thực hành đúng pattern quyền**, không gắn với người thật (project
cá nhân, không có team 5 người) — mục tiêu là nắm chắc RBAC trên đúng tên
resource sẽ dùng khi deploy thật, không phải dựng thêm 1 công ty hư cấu nữa.

Môi trường: AlmaLinux 8.9 — dùng `dnf`, `firewalld`, SELinux enforcing.
Không dùng lệnh kiểu Ubuntu (`apt`, `ufw`, `adduser` kiểu Debian).

Đi đúng thứ tự các bước bên dưới. Mỗi bước: đọc "Vì sao", gõ tay lệnh, rồi
**dừng lại giải thích cho tôi bạn hiểu gì trước khi sang bước kế tiếp.**
Tự tay gõ từng lệnh — không dùng script cấu hình có sẵn.

---

## Bước 0 — Kiểm tra trạng thái hiện tại trước khi tạo gì

**Vì sao:** VPS này đã (hoặc sắp) có `deploy`/`catalog-logs`/`dev1`/`dev2`/
`qa1` từ NOVA-002 gốc — phải xác nhận rõ baseline của **namespace `ec-*`**
riêng, và tiện thể xác nhận namespace gốc không bị đụng ngược lại.

```bash
getent group ec-deploy ec-logs 2>/dev/null || echo "chưa có group ec-* nào"
getent passwd ec-dev1 ec-dev2 ec-qa1 2>/dev/null || echo "chưa có user ec-* nào"
ls -ld /opt/electrostore-backend 2>/dev/null || echo "chưa có thư mục"
sudo ls -l /etc/sudoers.d/
sudo -l
```

*(Từ NOVA-001, bạn không còn login bằng `root` nữa — mọi lệnh cần quyền root
bên dưới đều phải có `sudo` phía trước. `getent`/`id`/`ls -ld` trên các path
thường không cần, nhưng `/etc/sudoers.d/` mặc định chỉ `root` đọc được nên
cần `sudo` ngay từ bước kiểm tra này.)*

🛑 Giải thích: vì sao bước này còn quan trọng hơn cả ở NOVA-002 gốc — nếu
lỡ tay tạo trùng tên với `dev1`/`dev2`/`qa1`/`deploy`/`catalog-logs` đã có
sẵn trên VPS, chuyện gì sẽ xảy ra với quyền của bản gốc?

> dev1 là "dân nhà catalog-service", lẽ ra không được đụng vào "nhà" electrostore-backend. Gõ nhầm lệnh → nó có chìa (group) vào nhà kia luôn → vi phạm đúng nguyên tắc mỗi nhà chỉ dân nhà đó ra vào.

---

## Bước 1 — Tạo 2 group riêng cho electrostore-backend

**Vì sao:** giữ ranh giới quyền độc lập giữa 2 ứng dụng trên cùng 1 VPS —
một group/app bị lộ không tự động kéo theo quyền trên app còn lại.

```bash
sudo groupadd ec-deploy
sudo groupadd ec-logs
getent group ec-deploy ec-logs
```

🛑 Giải thích: nếu dùng lại đúng group `deploy`/`catalog-logs` cho cả 2 app
(thay vì tách `ec-deploy`/`ec-logs`), điều gì bị mất về mặt cô lập quyền
giữa `catalog-service` và `electrostore-backend`?
> user trong cùng group có thể vào được app của nhau, vi phạm quy tắc
> review: Không phải "user vào được app của nhau" (nghe như 2 group khác nhau đụng nhau) — mà là chỉ còn 1 group deploy chung, nó phải chgrp cho cả 2 thư mục scaffold. Nên bất kỳ ai bạn thêm vào deploy (dù định cấp quyền cho app nào) đều tự động có quyền trên cả 2 app cùng lúc — mất khả năng cấp quyền riêng cho từng app
---

## Bước 2 — Tạo thư mục scaffold `/opt/electrostore-backend`

**Vì sao:** đây sẽ là nơi checkout/deploy `electrostore-backend` thật sau
này (giống vai trò của `/opt/catalog-service` ở bản gốc, nhưng cho project
thật). Bước này chỉ dựng khung thư mục với đúng group + setgid.

```bash
sudo mkdir -p /opt/electrostore-backend
sudo chgrp ec-deploy /opt/electrostore-backend
sudo chmod 2775 /opt/electrostore-backend
ls -ld /opt/electrostore-backend
```

🛑 Giải thích: `electrostore-backend` build bằng Maven (`./mvnw`), không có
Dockerfile sẵn ở root — repo dùng `docker-compose.yml` riêng chỉ để bật
Postgres/Redis cho local dev, không phải để chạy chính app. Theo bạn, thư
mục `/opt/electrostore-backend` này sau này nên chứa gì khi deploy thật lên
VPS (source code để build tại chỗ, hay chỉ artifact/image đã build sẵn)? Vì
sao lựa chọn đó ảnh hưởng đến việc VPS có cần cài JDK 21 + Maven hay không?
> Trả lời: Trong tương lai tôi sẽ đóng gói app đó thành image, để thuận tiện cho việc release nhanh mà không cần phải mang cả source code đi

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — chưa có ví dụ cụ thể |
| **Độ chính xác** | ⭐⭐⭐☆☆ — chọn đúng hướng (image), nhưng bỏ trống nửa sau câu hỏi |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ |

**Nhận xét:**
- ✅ Chọn "image" đúng hướng — build 1 lần, chạy nhiều nơi, release nhanh, đúng thực hành production.
- ⚠️ Chưa trả lời phần quan trọng nhất của câu hỏi: **vì sao lựa chọn đó quyết định VPS có cần JDK 21 + Maven hay không.**
- ⚠️ Hệ quả chưa nhắc tới: nếu chỉ chạy image, `/opt/electrostore-backend` thật ra không cần giữ **source code** nữa — nó sẽ chứa deploy config (`docker-compose.yml`, `.env`), còn bản thân image nằm trong registry/local docker image store, không nằm trong thư mục này.

**💡 Bổ sung:**
> Build image nghĩa là bước "biên dịch" (`mvnw package` → `.jar` → đóng vào image) xảy ra ở **nơi khác** — máy CI (repo `electrostore-backend` đã có sẵn `.github/workflows/ci.yml`) — không xảy ra trên VPS. VPS chỉ làm 1 việc: `docker pull` + `docker run`/`docker compose up`, tức là chỉ cần **Docker Engine**, không cần JDK 21 hay Maven.
>
> Ngược lại — đưa source code lên `/opt/electrostore-backend` rồi build tại chỗ (`./mvnw package` ngay trên VPS) — thì VPS bắt buộc phải cài JDK 21 + Maven, biến VPS production thành vừa build-server vừa runtime-server: tăng attack surface (thêm toolchain phải vá lỗi), build không tái lập được (dev/CI/VPS có thể ra kết quả khác nhau), và chậm hơn vì phải build lại mỗi lần deploy thay vì chỉ pull image có sẵn.

---

## Bước 3 — Tạo `ec-dev1`, `ec-dev2`, gán đúng group, KHÔNG gán `wheel`

**Vì sao:** đúng nguyên tắc least-privilege — không ai ngoài bạn cần sudo
toàn quyền chỉ để làm việc hàng ngày với electrostore-backend.

```bash
sudo useradd -m -s /bin/bash ec-dev1
sudo passwd ec-dev1
sudo usermod -aG ec-deploy,ec-logs ec-dev1

sudo useradd -m -s /bin/bash ec-dev2
sudo passwd ec-dev2
sudo usermod -aG ec-deploy,ec-logs ec-dev2

id ec-dev1
id ec-dev2
```

🛑 Giải thích: chạy `id ec-dev1` và `id dev1` (bản gốc) cạnh nhau — hai user
này có chia sẻ group nào không? Vì sao việc **không** chia sẻ group nào cả
chính là bằng chứng cho thấy 2 namespace đang thật sự độc lập, chứ không chỉ
khác tên?

---

## Bước 4 — Tạo khung sudoers riêng cho `ec-dev2` (chưa điền lệnh thật)

**Vì sao:** giống vai trò `dev2` ở bản gốc, nhưng lệnh thật (`docker
compose ...` hay lệnh restart systemd unit) chưa xác định được vì
electrostore-backend chưa được deploy thật lên VPS — để khung TODO, chốt
lệnh cụ thể khi quyết định cách chạy production (Docker Compose hay
systemd/jar trực tiếp).

```bash
sudo visudo -f /etc/sudoers.d/ec-dev2
```

Nội dung file (gõ tay trong editor mở ra):

```
# NOVA-002-B: khung quyền hạn chế cho ec-dev2 trên electrostore-backend.
# Lệnh thật sẽ điền khi chốt cách chạy production (docker compose hay
# systemd unit chạy jar Spring Boot trực tiếp).
Cmnd_Alias ELECTROSTORE_SVC_TODO = /bin/true
ec-dev2 ALL=(root) NOPASSWD: ELECTROSTORE_SVC_TODO
```

Sau khi lưu, verify ngay (đừng đóng session hiện tại trước khi verify):

```bash
sudo -l -U ec-dev2
sudo ls -l /etc/sudoers.d/ec-dev2
sudo -l
```

🛑 Giải thích: file `/etc/sudoers.d/dev2` (bản gốc) và
`/etc/sudoers.d/ec-dev2` (bản B) đọc độc lập với nhau — nếu sau này bạn xoá
nhầm file `dev2` khi định dọn dẹp bản gốc, `ec-dev2` có bị ảnh hưởng gì
không? Vì sao?
> Trả lời: Không bị ảnh hưởng gì

---

## Bước 5 — Tạo `ec-qa1`: không shell thường, chuẩn bị SSH key

**Vì sao:** giống `qa1` ở bản gốc — chỉ cần đọc log qua SFTP, không cần
shell tương tác.

```bash
sudo useradd -m -s /sbin/nologin ec-qa1
sudo mkdir -p /home/ec-qa1/.ssh
sudo chmod 700 /home/ec-qa1/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKfHOrCWTDhVLyADG4WLDaVNksGRYmNuYMQJ3Lv5MkSP ec-qa1
" | sudo tee -a /home/ec-qa1/.ssh/authorized_keys > /dev/null
sudo chmod 600 /home/ec-qa1/.ssh/authorized_keys
sudo chown -R ec-qa1:ec-qa1 /home/ec-qa1/.ssh
```

🛑 Giải thích: dòng ghi `authorized_keys` không thể viết `sudo echo "..." >>
file` — vì sao? (gợi ý: `>>` được **shell hiện tại của bạn** mở file để ghi
TRƯỚC KHI `sudo` chạy `echo`, không phải bản thân tiến trình `echo` chạy
dưới quyền root mở file). Đây là lý do phải dùng `sudo tee -a` thay vì
`sudo echo ... >>`.

🛑 Giải thích: nếu bạn dùng **chung một cặp SSH key** cho cả `qa1` (bản gốc)
và `ec-qa1` (bản B) thay vì tạo cặp key riêng, rủi ro gì phát sinh khi một
trong hai app bị revoke quyền truy cập của QA?
-> Cả 2 user đều sẽ không có quyền truy cập 

---

## Bước 6 — Khoá `ChrootDirectory` và cấu hình `sshd` cho `ec-qa1`
**Mục đích:** Biến tài khoản ec-qa1 thành một người dùng chỉ được phép truyền tệp qua SFTP và "nhốt" hoàn toàn trong phòng riêng của họ, không thể nhìn thấy bất kỳ ngóc ngách nào khác của hệ thống.

**Vì sao:** cùng yêu cầu nghiêm ngặt của OpenSSH như bản gốc — sai bước này
sshd từ chối toàn bộ login, kể cả SFTP.

```bash
sudo chown root:root /home/ec-qa1
sudo chmod 755 /home/ec-qa1
ls -ld /home/ec-qa1
ls -ld /home/ec-qa1/.ssh
```

Thêm block sau vào **cuối** `/etc/ssh/sshd_config` (mở bằng `sudo vi
/etc/ssh/sshd_config` hoặc `sudo nano ...` — file này root:root, user
thường không mở ghi được), **sau** block `Match User qa1` đã có từ bản gốc
(không chèn xen vào giữa):

```
Match User ec-qa1
    ChrootDirectory /home/ec-qa1
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

Test và áp dụng — **giữ nguyên session hiện tại đang mở**:

```bash
sudo sshd -t
sudo systemctl restart sshd
```

🛑 Giải thích: `sshd_config` giờ có 2 block `Match User` (một cho `qa1`, một
cho `ec-qa1`). Mỗi block `Match User` chỉ áp dụng cho đúng user được nêu tên
— nhưng thứ tự các block trong file có quan trọng không nếu hai user hoàn
toàn khác nhau và không có `Match` nào dùng điều kiện khác (như `Match
Group`)? Nếu sau này bạn đổi 1 trong 2 block thành `Match Group qa`, việc
"không đụng nhau" ở trên còn đúng không?

---

## Bước 7 — Cấp ACL read-only cho `ec-qa1` trên thư mục log

**Vì sao:** giống bản gốc, nhưng log là của electrostore-backend, và
`ec-qa1` không nằm trong group `ec-logs` — quyền đọc log phải cấp qua ACL
riêng.

```bash
sudo mkdir -p /var/log/electrostore-backend
sudo setfacl -m u:ec-qa1:r-x /var/log/electrostore-backend
sudo setfacl -d -m u:ec-qa1:r-x /var/log/electrostore-backend
sudo getfacl /var/log/electrostore-backend

sudo mkdir -p /home/ec-qa1/logs
sudo mount --bind /var/log/electrostore-backend /home/ec-qa1/logs
sudo mount -o remount,bind,ro /home/ec-qa1/logs
findmnt /home/ec-qa1/logs
```

Để bind mount này sống sót qua reboot, thêm vào `/etc/fstab` (mở bằng
`sudo vi /etc/fstab`) — **thêm dòng mới**, không sửa dòng đã có của bản gốc
(`/var/log/catalog-service ...`):

```
/var/log/electrostore-backend  /home/ec-qa1/logs  none  bind,ro  0  0
```

🛑 Giải thích: `/etc/fstab` giờ có 2 dòng bind-mount, một cho mỗi app. Nếu
một trong hai dòng bị gõ sai cú pháp, `mount -a` (chạy lúc boot) xử lý dòng
còn lại thế nào — cả hệ thống fail-to-boot, hay chỉ riêng dòng lỗi đó không
mount được?

---

## Bước 8 — Xác nhận: bản B không cần "Product Owner" hư cấu

**Vì sao:** ticket gốc yêu cầu quyết định "không tạo account" phải được ghi
lý do. Ở bản B, lý do khác bản gốc — không phải vì PO xem dashboard, mà vì
đây là project cá nhân, không có vai trò đó.

```bash
getent passwd | grep -i "^ec-" 
```

🛑 Giải thích: liệt kê đúng 3 account `ec-*` vừa tạo (`ec-dev1`, `ec-dev2`,
`ec-qa1`) — không có account thứ 4 nào khác. Viết 1-2 câu: vì sao bản B chỉ
cần 3 account thay vì mô phỏng đủ "5 người" như roster NovaCart gốc?

---

## Bước 9 — Verify toàn bộ, đối chiếu với Acceptance Criteria của NOVA-002

Acceptance Criteria giữ nguyên như ticket NOVA-002 gốc trong
`01-foundation.md`, chỉ thay tên resource theo bảng mapping:

```bash
getent group ec-deploy ec-logs
ls -ld /opt/electrostore-backend
id ec-dev1
id ec-dev2
sudo -l -U ec-dev2
ssh ec-qa1@<vps-ip>              # phải bị từ chối (không có shell thường)
sftp ec-qa1@<vps-ip>             # phải vào được, chỉ thấy đúng thư mục chroot
sudo getfacl /var/log/electrostore-backend
getent passwd | grep -i "^ec-"   # đúng 3 dòng: ec-dev1, ec-dev2, ec-qa1
```

Đối chiếu thêm với namespace gốc để chắc chắn không có tác dụng phụ:

```bash
id dev1        # vẫn y hệt trước khi làm bản B — không có group ec-* nào lạ
id dev2
sudo -l -U dev2   # vẫn chỉ CATALOG_SVC_TODO, không lẫn ELECTROSTORE_SVC_TODO
```

---

## Sau khi xong: Reflection Questions

Trả lời lại đúng 4 câu "Reflection Questions (Feynman)" của ticket
`NOVA-002` trong [`plans/01-foundation.md`](../plans/01-foundation.md),
lần này liên hệ ví dụ bằng namespace `ec-*` thay vì `dev1`/`dev2`/`qa1`. Báo
lại đây khi xong, tôi sẽ review như đã làm với NOVA-001 và NOVA-002 gốc.
