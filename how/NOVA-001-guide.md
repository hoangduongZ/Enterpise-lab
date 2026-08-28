# Guided Execution — NOVA-001

Mode: [guided-execution-mode.md](guided-execution-mode.md). Ticket gốc:
[../plans/01-foundation.md](../plans/01-foundation.md).

Môi trường: AlmaLinux 8.9 — dùng `dnf`, `firewalld`, SELinux enforcing.
Không dùng lệnh kiểu Ubuntu (`apt`, `ufw`).

Đi đúng thứ tự các bước bên dưới. Mỗi bước: đọc "Vì sao", gõ tay lệnh, rồi
**dừng lại giải thích cho tôi bạn hiểu gì trước khi sang bước kế tiếp.**

---

## Bước 0 — Kiểm tra trạng thái hiện tại trước khi đổi gì

**Vì sao:** không bao giờ đổi cấu hình khi chưa biết trạng thái gốc — nếu VPS
đã có sẵn user/firewall từ trước, các bước sau có thể thừa hoặc gây xung đột.

Trên VPS (đang SSH bằng root):

```bash
whoami
cat /etc/os-release
id nova-ops 2>/dev/null || echo "chưa có user nova-ops"
systemctl status firewalld --no-pager
```

*(`nova-ops` chỉ là tên gợi ý cho user vận hành — bạn có thể chọn tên khác,
nhớ dùng nhất quán cho các bước sau.)*

🛑 Giải thích: 4 lệnh trên cho bạn biết gì về trạng thái VPS? Nếu `id
nova-ops` KHÔNG báo "chưa có", nghĩa là sao — bạn sẽ làm gì khác đi?
- Lệnh đầu là kiểm tra tên user hiện tại đang hoạt động là ai
- Lệnh 2 là kiểm tra thông tin os hiện tại
- Lệnh 3 là kiểm tra thông tin user 'nova-ops', chưa tồn tại thì tạo, tồn tại rồi thì thôi, còn 2> và /dev/null tôi chưa hiểu hãy giải thích
- Lệnh 4 tôi không biết hãy giải thích

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ — Lệnh 1 & 2 rõ ràng, dễ hiểu |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — Chưa có ví dụ output thực tế |
| **Độ chính xác** | ⭐⭐⭐☆☆ — Lệnh 1 & 2 đúng, lệnh 3 đúng một nửa, lệnh 4 chưa trả lời |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐⭐ — Thành thật nói "chưa hiểu" thay vì bịa |

**Nhận xét:**
- ✅ Thái độ Feynman đúng: thừa nhận chỗ chưa hiểu thay vì bịa
- ✅ Lệnh 1 (`whoami`) và lệnh 2 (`cat /etc/os-release`) giải thích đúng bản chất
- ⚠️ Lệnh 3 — hiểu đúng phần `id nova-ops` nhưng **chưa trả lời câu hỏi chính**: "nếu user đã tồn tại thì làm gì khác?"
- ⚠️ Lệnh 4 — chưa trả lời, cần tìm hiểu `systemctl` và `firewalld`

**💡 Câu trả lời mẫu theo Feynman:**

> Tưởng tượng bạn vừa nhận bàn giao một căn nhà (VPS) — trước khi sửa chữa, đi một vòng kiểm tra:
>
> 1. **`whoami`** → "Tôi đang đứng trong nhà với tư cách ai?" — phải là `root` thì mới có quyền sửa.
>
> 2. **`cat /etc/os-release`** → "Nhà này xây bằng gạch gì?" — AlmaLinux 8.9 thì dùng `dnf`, không phải `apt`. Biết sai OS = gõ lệnh sai.
>
> 3. **`id nova-ops 2>/dev/null || echo "chưa có user nova-ops"`**
>    - `id nova-ops` = kiểm tra user `nova-ops` tồn tại không
>    - `2>/dev/null` = nếu lệnh bị lỗi, **giấu dòng lỗi đi**. `2>` chuyển hướng **stderr** (kênh báo lỗi), `/dev/null` là "thùng rác"
>    - `||` = "nếu lệnh trước THẤT BẠI thì chạy lệnh sau"
>    - **Nếu user đã tồn tại** → không tạo lại, mà kiểm tra: đúng group `wheel` chưa? Có SSH key chưa?
>
> 4. **`systemctl status firewalld --no-pager`** → "Nhà có hệ thống an ninh (tường lửa) đang chạy không?"
>    - `systemctl` = công cụ quản lý dịch vụ trên Linux (bật/tắt/xem trạng thái)
>    - `firewalld` = tường lửa, quyết định ai được phép kết nối vào
>    - `--no-pager` = in thẳng ra terminal, không mở trình cuộn trang

---

## Bước 1 — Tạo SSH keypair trên máy Mac (nếu chưa có)

**Vì sao:** SSH key-based access dùng cặp khóa bất đối xứng — private key
giữ trên máy bạn, public key đưa lên server. Server không bao giờ cần biết
private key.

Trên **máy Mac** (không phải VPS):

```bash
ls -la ~/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -C "nova-ops"
```

🛑 Giải thích: vì sao lệnh này kiểm tra `id_ed25519.pub` (public key) chứ
không phải `id_ed25519` (private key)? Nếu ai đó lấy được file `.pub` của
bạn, họ làm được gì?
- private key không có extension .pub, tôi đoán vậy
- .pub nằm trên server, và họ sẽ chẳng làm được gì, nếu không có private key

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐☆☆ — Hiểu ý nhưng diễn đạt hơi lộn |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — Chưa có ví dụ so sánh |
| **Độ chính xác** | ⭐⭐⭐☆☆ — Câu 2 đúng ý chính, câu 1 chưa trả lời đúng câu hỏi |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — Ngôn ngữ tự nhiên, không lý thuyết suông |

**Nhận xét:**
- ✅ Nắm đúng bản chất: có `.pub` mà không có private key = **vô hại**
- ⚠️ Câu 1 trả lời "private key không có extension .pub" — đúng nhưng **không phải câu hỏi**. Câu hỏi là: **VÌ SAO kiểm tra `.pub` chứ không phải private key?**
- ⚠️ ".pub nằm trên server" — **không chính xác**. File `.pub` tồn tại **cả trên Mac** (nơi tạo) **lẫn trên server** (nơi copy lên). Lệnh `ls ~/.ssh/id_ed25519.pub` đang chạy trên Mac, không phải server

**💡 Câu trả lời mẫu theo Feynman:**

> Hình dung SSH key như **ổ khoá và chìa khoá**:
> - **Public key** (`.pub`) = ổ khoá — gắn lên cửa nhà (server), ai nhìn thấy cũng không sao
> - **Private key** (không có `.pub`) = chìa khoá — chỉ bạn giữ trên máy Mac
>
> `ssh-keygen` luôn tạo **cả cặp cùng lúc**. Nên nếu thấy `.pub` tồn tại → private key chắc chắn cũng đã có → **không cần tạo lại**.
>
> Kiểm tra `.pub` thay vì private key vì: cả hai luôn tạo cùng lúc, check một cái là đủ. Và `.pub` là file "an toàn" — nếu vô tình lộ trong log cũng không sao.
>
> Ai đó lấy được `.pub`? **Chẳng làm được gì.** Giống như chụp hình ổ khoá — không mở được cửa nếu không có chìa khoá.

---

## Bước 2 — Tạo non-root user trên VPS

**Vì sao:** không thao tác hàng ngày bằng root — giảm blast radius nếu gõ
nhầm lệnh, và tạo audit trail rõ ràng ("ai" chứ không phải "root" làm gì).

Trên VPS (vẫn đang là root):

```bash
useradd -m -s /bin/bash nova-ops
passwd nova-ops
usermod -aG wheel nova-ops
```

🛑 Giải thích: `-m` và `-s /bin/bash` trong `useradd` làm gì? Vì sao
`usermod -aG` dùng flag `-a`, nếu bỏ `-a` thì rủi ro gì xảy ra (gợi ý: user
này có sẵn group nào khác không)? `wheel` trên AlmaLinux đóng vai trò gì?
- -m dùng để create user's home directory
- -s chỉnh định shell được sử dụng cho user
- useradd để thêm, tạo 1 user mới trong hệ thống linux
- -a là để thêm, bỏ đi thì không biết chuyện gì xảy ra
- wheel hình như là group cho phép quyền sudo

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ — Giải thích ngắn gọn, dễ hiểu |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — Chưa có ví dụ cụ thể cho từng flag |
| **Độ chính xác** | ⭐⭐⭐☆☆ — `-m`, `-s`, `useradd` đúng; `-a` và `wheel` chưa rõ |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — Ngôn ngữ tự nhiên, thành thật nói "không biết" |

**Nhận xét:**
- ✅ `-m` (tạo home directory) — **đúng**
- ✅ `-s /bin/bash` (chỉ định shell) — **đúng**
- ✅ `useradd` (tạo user mới) — **đúng**
- ✅ `wheel` cho phép sudo — **đúng**, dám đoán dù chưa chắc → thái độ Feynman tốt
- ⚠️ `-a` chỉ nói "để thêm" nhưng **chưa trả lời câu hỏi chính**: bỏ `-a` thì rủi ro gì? Đây là điểm quan trọng nhất của câu hỏi
- ⚠️ Chưa trả lời: `passwd nova-ops` dùng để làm gì (tuy câu hỏi không hỏi trực tiếp, nhưng hiểu cả block lệnh là cần thiết)

**💡 Câu trả lời mẫu theo Feynman:**

> Hình dung bạn đang tạo thẻ nhân viên mới cho toà nhà (VPS):
>
> **`useradd -m -s /bin/bash nova-ops`** = tạo nhân viên tên `nova-ops`
> - `-m` = cấp cho nhân viên **một phòng riêng** (home directory `/home/nova-ops`). Không có `-m` → nhân viên có tên trong danh sách nhưng không có phòng để làm việc, không có chỗ lưu file
> - `-s /bin/bash` = cho nhân viên dùng **bàn phím loại bash** (shell). Nếu không chỉ định, một số hệ thống mặc định cho shell `/bin/sh` (bàn phím rút gọn, thiếu tính năng) hoặc thậm chí `/sbin/nologin` (không cho đăng nhập luôn)
>
> **`passwd nova-ops`** = đặt mật khẩu cho nhân viên — cần thiết vì `sudo` sẽ hỏi mật khẩu này mỗi khi dùng quyền cao
>
> **`usermod -aG wheel nova-ops`** = thêm nhân viên vào **nhóm bảo vệ** (group `wheel`)
> - `wheel` trên AlmaLinux/RHEL = nhóm được phép dùng `sudo` (chạy lệnh với quyền root)
> - `-G wheel` = gán vào group `wheel`
> - `-a` = **append** (thêm vào, giữ nguyên các group cũ)
> - **Bỏ `-a` thì sao?** → `-G wheel` sẽ **thay thế toàn bộ** danh sách group phụ của user, chỉ còn lại `wheel`. Nếu user đang thuộc các group khác (ví dụ `docker`, `nginx`) → mất hết quyền truy cập các tài nguyên của những group đó. Giống như nói "nhân viên này CHỈ thuộc phòng bảo vệ" thay vì "THÊM vào phòng bảo vệ"

---

## Bước 3 — Đưa public key lên VPS cho `nova-ops`

**Vì sao:** đây là bước biến `nova-ops` từ "user có password" thành "user
login được bằng key".

Từ **máy Mac**:

```bash
ssh-copy-id nova-ops@<vps-ip>
```

Nếu `ssh-copy-id` không có sẵn, làm thủ công trên VPS (đang là root):

```bash
mkdir -p /home/nova-ops/.ssh
chmod 700 /home/nova-ops/.ssh
echo "<nội dung file id_ed25519.pub>" >> /home/nova-ops/.ssh/authorized_keys
chmod 600 /home/nova-ops/.ssh/authorized_keys
chown -R nova-ops:nova-ops /home/nova-ops/.ssh
```

🛑 Giải thích: vì sao `.ssh` phải là `700` và `authorized_keys` phải là
`600`? Thử đoán: nếu để `authorized_keys` là `644` (ai cũng đọc được),
sshd có chấp nhận login bằng key đó không? Vì sao permission lại ảnh hưởng
đến việc này (đây không phải là bảo mật do "khó đọc", mà do chính sshd chủ
động kiểm tra permission và từ chối nếu quá lỏng).
- .ssh là 700 vì chỉ muốn root có quyền với thư mục này, các thằng khác không có quyền
- authorized_keys là 600 vì co scope file này chỉ có quyền đọc và ghi không có thực thi script nào cả
---

## Bước 4 — Test login bằng key TRƯỚC khi khoá root

**Vì sao:** đây là bước bắt buộc theo constraint của ticket — không được tự
khoá mình ra ngoài. Phải xác nhận đường lui hoạt động trước khi đóng đường
cũ.

Mở **một terminal MỚI** trên Mac (giữ nguyên session root cũ đang mở):

```bash
ssh nova-ops@<vps-ip>
```

Sau khi vào được, test sudo:

```bash
sudo whoami
```

🛑 Giải thích: lệnh `sudo whoami` hỏi password của ai — `nova-ops` hay
`root`? Kết quả in ra là gì, vì sao? Nếu lệnh này thất bại, bạn có nên đi
tiếp Bước 5 không?
- sudo whoami hỏi password của nova-opsv và khi in ra nó in ra là root, vì với vai trò sudo người dùng đang tương tác hệ thống với tư cách là root
- Lệnh này thất bại tức là sudo không thành công
**Không đi tiếp nếu bước này chưa chạy đúng.**

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ — Giải thích tự nhiên, dễ hiểu |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — Chưa có ví dụ cụ thể |
| **Độ chính xác** | ⭐⭐⭐⭐☆ — Đúng cả 2 ý chính |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — Ngôn ngữ tự nhiên |

**Nhận xét:**
- ✅ **Đúng**: `sudo whoami` hỏi password của **`nova-ops`** (user đang login), không phải root
- ✅ **Đúng**: output in ra `root` — vì `whoami` trả về user đang thực thi lệnh, mà `sudo` chạy lệnh với quyền root
- ⚠️ Câu "lệnh này thất bại tức là sudo không thành công" — **đúng nhưng quá chung chung**. Câu hỏi hỏi: "bạn có nên đi tiếp Bước 5 không?" → cần trả lời **vì sao không được đi tiếp**, không chỉ nói "không thành công"
- ⚠️ Chưa giải thích: nếu thất bại nghĩa là `nova-ops` **chưa có quyền sudo** → mà Bước 5 sẽ khoá root login → bạn sẽ **tự khoá mình ra ngoài**, không ai có quyền admin nữa

**💡 Câu trả lời mẫu theo Feynman:**

> Hình dung bạn đang thử chìa khoá dự phòng **trước khi đổi ổ khoá cửa chính**:
>
> **`sudo whoami` hỏi password của ai?** → Hỏi password của **`nova-ops`** (người đang gõ lệnh). `sudo` không bao giờ hỏi password root — nó hỏi password của chính bạn để xác nhận "đúng bạn đang ngồi đây, không phải ai đó bỏ đi quên đăng xuất".
>
> **Output in ra gì?** → `root`. Vì `whoami` hỏi "tôi là ai?", mà `sudo` chạy lệnh **với tư cách root** → câu trả lời là `root`.
>
> **Nếu thất bại, đi tiếp Bước 5 không?** → **TUYỆT ĐỐI KHÔNG.** Bước 5 sẽ khoá root login qua SSH. Nếu `nova-ops` chưa sudo được mà bạn khoá root → không ai có quyền admin nữa → **tự khoá mình ra ngoài vĩnh viễn**. Giống như bạn chưa thử chìa dự phòng mà đã vứt chìa chính đi.

---

## Bước 5 — Khoá root login qua SSH + tắt password authentication

**Vì sao:** ngay cả khi `nova-ops` sudo lên root được, "login trực tiếp bằng
root" và "sudo lên root" là hai việc khác nhau về mặt audit và rủi ro (đây là
đúng câu hỏi số 2 trong Reflection Questions của ticket — nghĩ trước khi làm
bước này).

Trên VPS, vẫn giữ session root cũ mở, kiểm tra trước khi sửa:

```bash
grep -E "^(PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config
ls /etc/ssh/sshd_config.d/ 2>/dev/null
```

*(AlmaLinux/RHEL 8+ có thể có file trong `sshd_config.d/` override lại giá
trị bạn sửa ở file chính — cần kiểm tra trước khi kết luận cấu hình cuối
cùng là gì.)*

Sửa `/etc/ssh/sshd_config` (dùng `vi`/`nano`, tự tay sửa, không dùng `sed`
một dòng để "cho nhanh" — bạn cần đọc và hiểu ngữ cảnh xung quanh dòng đó):

```
PermitRootLogin no
PasswordAuthentication no
```

Áp dụng:

```bash
sshd -t
systemctl restart sshd
```

🛑 Giải thích: `sshd -t` làm gì và tại sao chạy nó TRƯỚC `restart` chứ không
sau? Nếu `sshd -t` báo lỗi mà bạn vẫn `restart`, chuyện gì có thể xảy ra?
- Không biết, t có thể là test
- Nếu file sshd sai, mà khởi động lại thì run sẽ lỗi, mà lỗi thì SSH không vào được

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ — Ngắn gọn, thẳng vào vấn đề |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — Chưa có ví dụ |
| **Độ chính xác** | ⭐⭐⭐⭐☆ — Đúng cả 2 ý, thiếu 1 chi tiết quan trọng |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐⭐ — Thành thật "không biết" rồi đoán → rất Feynman |

**Nhận xét:**
- ✅ **Đoán đúng**: `-t` = **test** — kiểm tra cú pháp file config trước khi áp dụng thật
- ✅ **Đúng hậu quả**: config sai + restart → sshd lỗi → SSH không vào được
- ⚠️ Thiếu 1 chi tiết quan trọng: khi `restart` mà sshd **không khởi động lại được**, các **session SSH đang mở vẫn sống** (vì chúng là process cũ). Chỉ **connection mới** mới bị ảnh hưởng. Đó là lý do guide bảo **giữ nguyên session root cũ** — đó chính là đường cứu hộ
- ⚠️ Chưa giải thích: vì sao chạy `-t` **TRƯỚC** chứ không **SAU** restart? (vì sau khi restart fail thì sshd đã chết, chạy `-t` lúc đó biết lỗi nhưng không cứu được nữa — phải sửa config rồi start lại thủ công bằng session cũ)

**💡 Câu trả lời mẫu theo Feynman:**

> Hình dung `sshd` là **cánh cổng duy nhất** để bạn vào nhà (VPS) từ xa:
>
> **`sshd -t` là gì?** → `-t` = **test**. Nó đọc file config (`sshd_config`), kiểm tra cú pháp đúng không, rồi **báo kết quả mà không khởi động lại gì cả**. Giống như đọc lại bản thiết kế trước khi đập tường — phát hiện sai trên giấy thì sửa rẻ hơn nhiều so với đập xong mới thấy sai.
>
> **Vì sao TRƯỚC restart chứ không sau?**
> - `sshd -t` OK → yên tâm restart → cổng mới hoạt động bình thường
> - `sshd -t` báo lỗi → **dừng lại sửa**, cổng cũ vẫn đang chạy tốt
> - Nếu bỏ qua `-t` mà restart luôn với config sai → sshd **không khởi động lại được** → cổng sập → mọi connection mới bị từ chối
>
> **Tin tốt:** các session SSH **đang mở sẵn** (như terminal root từ Bước 4) vẫn sống — chúng là process cũ, không bị ảnh hưởng bởi restart. Đó là đường cứu hộ để sửa config rồi `systemctl start sshd` lại.
>
> **Tin xấu:** nếu bạn đã đóng hết terminal cũ trước khi restart → không còn đường vào → phải dùng VNC/console từ nhà cung cấp VPS.

---

## Bước 6 — Verify từ terminal thứ 3 (KHÔNG đóng 2 terminal cũ)

Mở **terminal thứ 3**:

```bash
ssh nova-ops@<vps-ip>          # phải vào được, không hỏi password
ssh root@<vps-ip>              # phải bị từ chối
ssh -o PubkeyAuthentication=no nova-ops@<vps-ip>   # ép dùng password, phải bị từ chối
```
-> go here
🛑 Giải thích: vì sao phải mở terminal MỚI để test thay vì dùng lại
terminal đã login sẵn từ Bước 4? Nếu bước này fail, 2 terminal cũ (đang mở
sẵn) giúp ích gì cho bạn lúc đó?

**Chỉ đóng 2 terminal cũ sau khi bước này pass hoàn toàn.**

---

## Bước 7 — Bật firewalld, kiểm tra SSH đã nằm trong zone active TRƯỚC khi enforce

**Vì sao:** đây chính là câu hỏi số 3 trong Reflection Questions của ticket —
thứ tự sai ở bước này có thể tự khoá bạn ra ngoài ngay lập tức.

```bash
systemctl status firewalld --no-pager
firewall-cmd --get-active-zones
firewall-cmd --list-all
```

🛑 Giải thích: output của `firewall-cmd --list-all` cho bạn biết `ssh` (dịch
vụ) đã được phép trong zone active hay chưa? Nếu CHƯA thấy `ssh` trong danh
sách `services`, bạn có nên bật `firewalld` ngay không, hay cần làm gì
trước?

Nếu `ssh` đã có sẵn trong zone active → bật firewalld:

```bash
systemctl enable --now firewalld
firewall-cmd --list-all
```

Nếu `ssh` **chưa** có → thêm trước khi enable:

```bash
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
systemctl enable --now firewalld
```

🛑 Giải thích: khác biệt giữa `firewall-cmd --add-service=ssh` (không có
`--permanent`) và có `--permanent` là gì? Vì sao ví dụ trên dùng
`--permanent` rồi `--reload` thay vì chỉ chạy runtime?

---

## Bước 8 — Verify cuối cùng theo đúng Acceptance Criteria của NOVA-001

Từ máy Mac (terminal mới, không phải session cũ):

```bash
ssh nova-ops@<vps-ip> "sudo whoami && sudo firewall-cmd --state && sudo firewall-cmd --list-all"
```

Đối chiếu với checklist gốc trong `01-foundation.md`:
- [ ] SSH key login cho `nova-ops` không hỏi password.
- [ ] `ssh root@<vps-ip>` bị từ chối.
- [ ] `sudo` chạy được, hỏi password của `nova-ops`.
- [ ] `firewall-cmd --list-all` chỉ hiện đúng service/port cần thiết.

---

## Sau khi xong: Reflection Questions

Quay lại đúng 4 câu hỏi "Reflection Questions (Feynman)" trong ticket
`NOVA-001` (`plans/01-foundation.md`) và trả lời bằng lời của bạn — không
chỉ báo "đã tick hết checklist". Báo lại đây khi bạn xong từng bước, tôi sẽ
review như một senior xem PR.
