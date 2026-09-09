# 01 — Foundation (Level 0: VPS Foundation)

Depends on: [00-master-plan.md](00-master-plan.md)

## Scope

- **Input:** 1 VPS trắng (chưa cấu hình gì), quyền root/console ban đầu.
- **Output:** VPS có SSH key-based access, 1 non-root sudo user để vận hành
  hàng ngày, root login qua SSH bị tắt, firewall cơ bản bật, sẵn sàng cho
  Level 1 (deploy `catalog-service`).
- **Out of scope (để phase sau):** domain/DNS/TLS thật (Level 1), cài Docker
  (Level 1), bất kỳ ứng dụng nào.

## Repository Evaluation (đầy đủ, theo mục 2 của system prompt)

### Option A (chosen) — spring-boot-microservices-series-v2

- **Tên project:** Spring Boot Microservices Series V2
- **GitHub URL:** https://github.com/rajadilipkolli/spring-boot-microservices-series-v2
- **Dùng để học phần gì:** toàn bộ hành trình — deploy, CI/CD, DB +
  migration (Liquibase), message queue (Kafka), observability
  (Prometheus/Grafana/OTel), security (OAuth2/Keycloak), performance testing
  (Gatling), và sau này là microservices + Kubernetes.
- **Architecture hiện tại:** microservices đầy đủ — `api-gateway`,
  `config-server`, `service-registry` (Eureka), `catalog-service`,
  `inventory-service`, `order-service`, `payment-service`,
  `retail-store-webapp`. Mỗi service có Dockerfile riêng, docker-compose
  riêng (Postgres + Kafka cho catalog-service), GitHub Actions workflow
  riêng, Testcontainers cho integration test.
- **Cần thay đổi/điều chỉnh gì:** chỉ chạy `catalog-service` độc lập ở
  Level 0–2 (bỏ qua Eureka/Gateway/Config Server ban đầu); thay Testcontainers
  runtime bằng Postgres thật trên VPS; tự viết lại pipeline CI/CD thay vì chỉ
  tái sử dụng workflow có sẵn (mục tiêu là **học**, không phải copy).
- **Mức độ khó:** trung bình (khởi động) → cao (khi mở rộng full
  microservices ở Level 7–8).
- **Vì sao đáng dùng cho lab doanh nghiệp:** một repo duy nhất map gần như
  1:1 với toàn bộ target architecture ở mục 11 của system prompt (gateway,
  nhiều service, DB, queue, observability), tránh phải quản lý nhiều repo
  rời rạc; đang được maintain tích cực (push mới nhất ngay hôm đánh giá,
  2026-08-16); license MIT rõ ràng.
- **Verify tồn tại:** xác nhận qua GitHub API — 59 sao, không bị archive,
  Java 21/Spring Boot 3.x, license MIT.

### Option B (alternative, dự phòng) — spring-boot-realworld-example-app

- **Tên project:** RealWorld Example App (Spring Boot / "Conduit")
- **GitHub URL:** https://github.com/gothinkster/spring-boot-realworld-example-app
- **Dùng để học phần gì:** monolith REST API cơ bản, JWT authentication,
  JPA + Postgres, test coverage tốt — phù hợp nếu muốn khởi động **đơn
  giản hơn**, không có sẵn microservices/queue.
- **Architecture hiện tại:** modular monolith, Spring Security + JWT filter,
  hỗ trợ H2/SQLite/Postgres, có thể build image qua Spring Boot Buildpacks.
- **Cần thay đổi gì:** phải tự thêm toàn bộ phần message queue, multi-service,
  observability, CI/CD nếu muốn đi hết roadmap 8 level — repo không có sẵn.
- **Mức độ khó:** thấp lúc bắt đầu, tăng mạnh khi tự mở rộng về sau (vì phải
  tự thiết kế phần mở rộng, không có sẵn trong repo).
- **Vì sao cân nhắc:** 1.5k+ sao, MIT, chất lượng code/test tốt, nhưng ít
  "bề mặt học" hơn cho các phase Kafka/microservices/observability so với
  Option A.
- **Verify tồn tại:** xác nhận qua GitHub API — 1577 sao, không archive,
  license MIT, push gần nhất 2024-07-13 (không còn active gần đây).

### Quyết định

Chọn **Option A** làm project chính cho toàn bộ chương trình. Lý do: bề mặt
học rộng hơn nhiều, map trực tiếp vào target architecture, và cho phép mô
phỏng đúng hành trình "monolith seed → tiến hóa thành microservices" bằng
cách **chỉ dùng một phần của repo trước**, thay vì phải ghép nhiều repo lại.

## Environment (confirmed by user, 2026-08-16)

- OS: **AlmaLinux 8.9** (RHEL family) — dùng `dnf`, `firewalld`, SELinux
  enforcing by default. Không áp dụng lệnh kiểu Ubuntu (`apt`, `ufw`).
- Access hiện tại: root qua SSH, full quyền, chưa có non-root user.

## Level 0 Goals

1. Người học có quyền truy cập VPS an toàn, không phụ thuộc mật khẩu root.
2. Có 1 user vận hành hàng ngày (non-root, có sudo).
3. Root login qua SSH bị tắt.
4. Firewall cơ bản chỉ mở cổng cần thiết (SSH, sau này 80/443).
5. Người học giải thích được **vì sao** từng bước cần thiết (không chỉ
   copy-paste).

## Ticket NOVA-001

```text
Ticket ID: NOVA-001
Title: Thiết lập quyền truy cập vận hành an toàn cho VPS mới
Priority: P1
Severity: N/A (onboarding task, not incident)
Business Context:
  NovaCart vừa thuê 1 VPS trắng để dần chuyển hệ thống lên đó. Hiện tại
  chỉ có root access qua console nhà cung cấp. Trước khi cài bất kỳ ứng
  dụng nào, đội vận hành yêu cầu quyền truy cập phải an toàn và theo đúng
  chuẩn (không dùng password, không thao tác trực tiếp bằng root).
Problem:
  VPS hiện chưa có user vận hành, root login qua SSH đang mở (mặc định),
  chưa có firewall.
Expected Outcome:
  - SSH key-based access hoạt động cho 1 user thường có quyền sudo.
  - Root không thể login qua SSH nữa.
  - Firewall chỉ cho phép cổng SSH (và cổng bạn đang dùng để không tự khóa
    mình ra ngoài).
Constraints:
  - Không được tự khóa mình ra khỏi VPS (mất kết nối SSH giữa chừng khi
    chưa xác nhận cấu hình mới hoạt động).
  - Không dùng script tự động có sẵn — tự tay thực hiện để hiểu từng bước.
Acceptance Criteria:
  - [x] `ssh <user>@<vps-ip>` bằng SSH key thành công, không hỏi password.
  - [x] `ssh root@<vps-ip>` (hoặc bất kỳ hình thức password login nào) bị
        từ chối.
  - [x] User vận hành chạy được `sudo` sau khi nhập password của chính nó.
  - [x] Firewall đang bật, `<firewall tool> status` cho thấy chỉ các cổng
        cần thiết được mở.
Technical Notes:
  - Nếu tự khóa mình ra ngoài, hầu hết nhà cung cấp VPS có "recovery
    console/VNC console" trên web control panel để vào lại mà không cần SSH.
Reflection Questions (Feynman):
  1. Private key rơi vào tay người khác nhưng không có passphrase — họ
     login được không? Vì sao? Passphrase bảo vệ cái gì, không bảo vệ cái gì?
> Private key rơi vào tay người khác nhưng họ không có passphrase thì họ không login được, vì passphrase dùng để bảo vệ private key, phải nhập đúng thì mới được login, không bảo vệ cái gì thì tao đéo biết
```

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐☆☆ |
| **Ví dụ minh hoạ** | ⭐☆☆☆☆ — chưa có ví dụ |
| **Độ chính xác** | ⭐☆☆☆☆ — hiểu ngược tình huống đề bài đặt ra |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — thành thật "đéo biết" thay vì bịa |

**Nhận xét:**
- ⚠️ Đọc lại đề: tình huống là **private key KHÔNG CÓ passphrase** (key trần, không mã hoá) — không phải "attacker không biết passphrase của một key có passphrase". Câu trả lời đang xử lý đúng tình huống ngược lại với đề bài.
- ⚠️ Vì key không hề được đặt passphrase, nên kẻ lấy được file **login được ngay lập tức**, không hề bị chặn gì cả — kết luận đúng phải ngược lại hoàn toàn với câu trả lời hiện tại ("không login được").
- ⚠️ Chưa trả lời phần 2 của câu hỏi (passphrase bảo vệ cái gì / không bảo vệ cái gì) — nhưng thừa nhận thẳng "không biết" vẫn tốt hơn đoán bừa.

**💡 Câu trả lời mẫu theo Feynman:**
> Coi private key như **chìa khoá nhà**, và passphrase như **một cái két sắt bọc quanh chìa khoá** khi nó không nằm trên tay bạn (tức là lúc nó nằm im dưới dạng file trên ổ cứng).
>
> - Key **không có passphrase** = chìa khoá để trần, không bọc két. Ai lấy được file đó là **cầm chìa khoá thật ngay**, login được luôn, không cần biết thêm gì.
> - Key **có passphrase** = file chìa khoá bị mã hoá, giống bọc trong két. Lấy được file vẫn chỉ cầm một cục sắt vô dụng — phải biết passphrase để "mở két" lấy chìa ra dùng.
>
> **Passphrase bảo vệ:** chỉ bảo vệ **file private key đang nằm im trên đĩa** — chống lại việc file bị đánh cắp (USB, backup rò rỉ, laptop bị hack).
>
> **Passphrase KHÔNG bảo vệ:**
> - Không bảo vệ khi key **đã giải mã và đang dùng** — ví dụ `ssh-agent` đã load key vào RAM rồi thì ai chiếm được session đó dùng key thoải mái, không cần gõ lại passphrase.
> - **Server hoàn toàn không biết** private key của bạn có passphrase hay không — sshd chỉ kiểm tra chữ ký hợp lệ khớp với public key nó có, không quan tâm phía client bảo vệ private key kiểu gì. Đây là chuyện thuần phía client/máy của bạn.
> - Không chống được keylogger/malware trên máy bạn ghi lại passphrase lúc gõ.

```text
  2. User thường vẫn sudo lên root được — vậy tắt "root login qua SSH" thực
     sự ngăn được điều gì? Bản chất khác nhau giữa "là root" và "login
     bằng tài khoản root" là gì?
> chắc là ngăn không bị tấn công trực tiếp vào tài khoản root, tôi không rõ
```

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐☆☆ |
| **Ví dụ minh hoạ** | ☆☆☆☆☆ — chưa có ví dụ |
| **Độ chính xác** | ⭐⭐☆☆☆ — đúng một phần, chưa chạm vào ý chính |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — thành thật "tôi không rõ" thay vì bịa |

**Nhận xét:**
- ✅ Đúng một mảnh: root là tài khoản bị brute-force nhiều nhất (username cố định, ai cũng biết), nên chặn login trực tiếp loại bỏ được kiểu tấn công đó.
- ⚠️ Chưa trả lời đúng trọng tâm câu hỏi: đề bài nhấn mạnh "user thường **vẫn** sudo lên root được" — nghĩa là việc **trở thành root vẫn xảy ra bình thường**, cái bị chặn không phải "quyền root" mà là **con đường vào root**. Câu trả lời chưa nói rõ: chặn được gì cụ thể khi quyền root vẫn còn nguyên đó.
- ⚠️ Chưa trả lời phần 2 của câu hỏi: khác biệt bản chất giữa "là root" (đang có quyền root) và "login bằng tài khoản root" (đăng nhập trực tiếp bằng chính account root) — đây mới là phần lõi của câu hỏi, và câu trả lời bỏ trống hoàn toàn.

**💡 Câu trả lời mẫu theo Feynman:**
> Hình dung tài khoản `root` như **một chiếc chìa khoá vạn năng dùng chung**, không khắc tên ai lên đó. Còn tài khoản cá nhân (`nova-ops`) + `sudo` giống như **mỗi người có chìa khoá riêng khắc tên mình**, và mỗi lần muốn dùng phòng VIP (quyền root) phải quẹt chìa riêng đó qua máy quét trước — máy quét ghi lại "ai vừa quẹt, lúc mấy giờ".
>
> **Tắt root login qua SSH ngăn được gì, dù user vẫn sudo lên root được?**
> Nó không ngăn việc *có quyền root* — quyền đó vẫn còn nguyên qua `sudo`. Cái nó ngăn là **con đường đi thẳng vào root mà không qua ai cả**:
> - Trước khi tắt: attacker chỉ cần đoán đúng **1 thứ duy nhất** — mật khẩu của `root` (username đã biết sẵn, giống nhau trên mọi Linux server). Đoán ra là vào thẳng, không ai biết là ai.
> - Sau khi tắt: attacker phải đoán ra **2 thứ** — username hợp lệ của một người cụ thể (không phải `root`, thường không công khai) VÀ mật khẩu/key của chính người đó. Khó hơn hẳn, và nếu vào được thì log vẫn ghi rõ **"nova-ops" vừa login**, không phải một cái tên chung chung "root".
>
> **"Là root" khác "login bằng root" ở đâu?**
> - **"Là root"** (qua `sudo`) = có một người **danh tính rõ ràng** (`nova-ops`) tạm thời mượn quyền root để chạy 1 lệnh, có log ghi "nova-ops đã sudo lúc X, chạy lệnh Y".
> - **"Login bằng root"** = không còn khái niệm "ai" nữa — mọi hành động từ lúc đó chỉ được ghi nhận là "root làm", dù có 5 người cùng biết mật khẩu root và thay nhau dùng, log cũng không phân biệt được là ai trong 5 người đó.
>
> Nói ngắn gọn: tắt root login không bớt quyền lực của root đi chút nào — nó chỉ **buộc mọi người phải đi qua cửa có tên, có camera**, thay vì có một cửa sau không ai biết ai ra vào.

```text
  3. Nếu bạn bật firewalld trước khi chắc chắn service ssh nằm trong zone
     đang active, chuyện gì sẽ xảy ra ngay lập tức? Tại sao thứ tự thao tác
     ở bước này quan trọng hơn bản thân câu lệnh?
> Nếu bật firewalld mà ssh không nằm trong phạm vi zone được active, sẽ bị nhốt ở bên ngoài, tức không được sử dụng dịch vụ ssh từ bên ngoài và phải dùng tới biện phát cuối cùng vào terminal của nhà bán VPS.
Và thứ tự quan trọng vì tránh việc bị mất quyền kiểm soát
```

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐⭐☆ |
| **Ví dụ minh hoạ** | ⭐⭐☆☆☆ — chưa có ví dụ/tình huống cụ thể |
| **Độ chính xác** | ⭐⭐⭐☆☆ — đúng hậu quả, hơi phóng đại cơ chế "bị nhốt ngay lập tức" |
| **Tránh thuật ngữ rỗng** | ⭐⭐⭐⭐☆ — ngôn ngữ tự nhiên, dễ hiểu |

**Nhận xét:**
- ✅ Đúng hậu quả cuối cùng: mất khả năng SSH từ ngoài vào, phải cầu cứu VNC/console của nhà cung cấp — đây là đáp án đúng phần "chuyện gì xảy ra".
- ⚠️ "Bị nhốt ở bên ngoài **ngay lập tức**" hơi phóng đại cơ chế. Giống bài học ở Bước 5 (`sshd restart`): firewalld có luật ngầm cho phép kết nối **đã thiết lập sẵn (ESTABLISHED)** tiếp tục chạy. Nên phiên SSH **hiện tại** (session root/nova-ops đang mở) thường **không bị rớt ngay** — cái bị chặn là **kết nối MỚI** từ bây giờ trở đi. Đó là lý do guide luôn dặn "giữ nguyên terminal cũ, mở terminal MỚI để test" — terminal cũ chính là đường cứu hộ, y hệat logic ở Bước 5.
- ⚠️ Phần 2 ("vì sao thứ tự quan trọng hơn bản thân câu lệnh") trả lời đúng hướng nhưng hơi chung chung ("tránh mất quyền kiểm soát") — chưa nói rõ **cơ chế** vì sao thứ tự lại là biến số quyết định, trong khi câu lệnh (`firewall-cmd`, `systemctl enable --now firewalld`) hoàn toàn giống nhau ở cả 2 thứ tự.

**💡 Câu trả lời mẫu theo Feynman:**
> Hình dung bạn đang đứng **trong nhà**, cửa chính đang mở, và bạn sắp lắp một hệ thống khoá cửa tự động (firewalld) chỉ cho phép người có tên trong danh sách được vào.
>
> **Chuyện gì xảy ra nếu bật khoá trước khi thêm tên "ssh" vào danh sách?**
> - Danh sách khách được duyệt (zone active) sẽ **không có "ssh"** trong đó.
> - Từ giờ, **ai gõ cửa mới** (kết nối SSH mới) đều bị khoá từ chối thẳng.
> - Nhưng bạn **đang đứng sẵn trong nhà** (phiên SSH hiện tại đã "được thiết lập" từ trước) — thường **không bị đuổi ra ngay**, vì hệ thống khoá có luật ngầm "ai đã vào rồi thì cứ để yên" (ESTABLISHED connections). Cái nguy hiểm là: nếu phiên đó rớt (mất mạng, đóng nhầm terminal) → bạn **không còn cách nào gõ cửa lại được nữa** từ bên ngoài, phải gọi nhà cung cấp VPS mở console khẩn cấp.
>
> **Vì sao THỨ TỰ quan trọng hơn bản thân câu lệnh?**
> Vì câu lệnh y hệt nhau (`firewall-cmd --add-service=ssh`, `systemctl enable --now firewalld`) — chạy **trước khi verify** hay **sau khi verify** đều dùng đúng lệnh đó, không lệnh nào "sai cú pháp" cả. Cái khác nhau là **thời điểm bạn phát hiện vấn đề**:
> - Verify trước, bật sau → nếu thiếu "ssh", bạn thấy ngay lúc còn an toàn, sửa xong mới bật khoá — không có rủi ro nào.
> - Bật trước, verify sau → bạn chỉ biết thiếu "ssh" **sau khi** khoá đã hoạt động — lúc đó lối vào start đã đóng, muốn sửa cũng không SSH vào lại được nữa (trừ khi may mắn phiên cũ chưa rớt).
>
> Nói cách khác: rủi ro không nằm trong câu lệnh, mà nằm trong việc bạn có **cơ hội sửa sai trước khi khoá cửa lại** hay không.

```text
  4. SELinux và firewalld giải quyết hai loại rủi ro khác nhau. Nếu phải
     giải thích cho một dev không rành ops, bạn sẽ nói firewalld chặn gì,
     SELinux chặn gì, bằng ví dụ cụ thể?
> SELinux là vệ sĩ bảo vệ những folder mặc định của hệ thống, cái nó bảo vệ là ... nhằm ...., tức là người ta đã vào được trong hệ thống rồi nhưng không tương tác được những vùng bị bảo vệ
Firewalld là tường lửa, chặn truy cập từ ngoài vào, tức là nếu bị chặn bằng 1 phương thức nào đó, thì người ngoài không thể vào được nhà
```

#### 🔍 AI Review

| Tiêu chí | Đánh giá |
|---|---|
| **Độ đơn giản** | ⭐⭐⭐☆☆ — phần firewalld ổn, phần SELinux bỏ dở bằng "..." |
| **Ví dụ minh hoạ** | ☆☆☆☆☆ — câu hỏi yêu cầu ví dụ cụ thể nhưng chưa có ví dụ nào |
| **Độ chính xác** | ⭐⭐⭐☆☆ — firewalld đúng, SELinux đúng hướng nhưng chưa đủ |
| **Tránh thuật ngữ rỗng** | ⭐⭐☆☆☆ — "vệ sĩ bảo vệ folder mặc định", "vùng bị bảo vệ" là mô tả chung chung, chưa nói rõ nó bảo vệ theo cơ chế nào |

**Nhận xét:**
- ✅ Firewalld: đúng — chặn/kiểm soát kết nối **từ ngoài mạng đi vào**, đúng bản chất "ai được phép gõ cửa nhà".
- ✅ Ý "người ta đã vào được trong hệ thống rồi nhưng không tương tác được những vùng bị bảo vệ" — **đúng hướng**, đây chính là điểm khác biệt cốt lõi: SELinux xử lý rủi ro xảy ra **sau khi** đã có ai/cái gì đó chạy được bên trong máy, khác hẳn firewalld (chặn từ ngoài).
- ⚠️ Câu trả lời SELinux bị bỏ dở ("cái nó bảo vệ là ... nhằm ....") — chưa nói được **cơ chế**: SELinux không chỉ bảo vệ "folder mặc định", mà gán **nhãn (label/context)** cho mọi file, port, tiến trình, và chỉ cho phép tương tác nếu policy khớp nhãn — kể cả khi quyền Unix thông thường (`chmod`/`chown`) đã cho phép.
- ⚠️ Thiếu hoàn toàn ví dụ cụ thể mà câu hỏi yêu cầu — đây là phần quan trọng nhất để chứng minh đã hiểu bản chất, không chỉ nhớ định nghĩa.

**💡 Câu trả lời mẫu theo Feynman:**
> Hình dung một **toà chung cư**:
> - **Firewalld** = bảo vệ đứng ở **cổng khu chung cư**. Nó quyết định ai được phép đi vào khuôn viên toà nhà từ ngoài đường — chặn đúng port/dịch vụ nào được mở (ví dụ chỉ mở cổng cho "khách đi thang máy lên căn hộ", tức port 22/80/443), còn lại đóng hết. Nếu bạn không qua được cổng này, bạn **không bao giờ chạm được vào toà nhà**.
> - **SELinux** = **nội quy riêng của từng căn hộ**, áp dụng cho người **đã vào được bên trong toà nhà rồi**. Dù bảo vệ cổng đã cho bạn vào (firewalld pass), nội quy vẫn quy định: "anh là nhân viên phục vụ tầng 3 (tiến trình web server), anh chỉ được mở đúng các phòng dán nhãn 'khu vực tầng 3' (`/var/www` với label `httpd_sys_content_t`), dù chìa khoá vạn năng của anh (quyền Unix, chủ sở hữu file) có mở được cả phòng tầng 5 (`/home/nova-ops`) đi nữa — nội quy vẫn từ chối, vì phòng đó không nằm trong nhãn cho phép của anh."
>
> **Ví dụ cụ thể:** hacker khai thác lỗ hổng web app, chiếm được tiến trình `httpd` (đã "vào nhà" hợp lệ qua port 80 mà firewalld cho phép). Hacker cố đọc `/etc/shadow` hoặc ghi file lạ vào `/home/nova-ops`. Firewalld **không giúp được gì** ở đây — kết nối đã vào từ trước rồi, không phải chuyện network nữa. Nhưng SELinux thấy: tiến trình đang chạy trong domain `httpd_t` cố chạm vào resource có type `shadow_t`/`user_home_t` — **không nằm trong policy cho phép** → **chặn**, dù user chạy `httpd` về mặt Unix có thể có quyền đọc file đó.
>
> Tóm gọn: **firewalld** quyết định ai được **vào nhà**; **SELinux** quyết định, khi đã vào nhà rồi, mỗi "vai trò" được phép **chạm vào cái gì** — hai lớp phòng thủ độc lập, không thay thế nhau.

## Team Roster (NovaCart Engineering)

Quyết định 2026-08-19 (xem "Important Decisions" trong
[00-master-plan.md](00-master-plan.md)): team kỹ thuật NovaCart cố định ở
**5 người**, mapping sang Linux account trên VPS như sau:

| Người | Vai trò | Linux account | Quyền trên VPS |
|---|---|---|---|
| Bạn | DevOps/SRE (mới tuyển) | user tạo ở NOVA-001 | `wheel` — sudo đầy đủ |
| — | Developer | `dev1` | group `deploy`, `catalog-logs`; không sudo |
| — | Developer (senior, backup on-call) | `dev2` | group `deploy`, `catalog-logs`; + 1 file riêng trong `sudoers.d` (Cmnd_Alias giới hạn) |
| — | QA / Test Engineer | `qa1` | không có shell thường — chỉ SFTP chroot jail + ACL read-only trên log |
| — | Product Owner | *(không tạo account)* | không truy cập server; xem dashboard khi Level 4 (observability) có |

Nguyên tắc: 3/5 người **không** có sudo, 1/5 người **không** có account nào.
Đây là lesson chính — "5 người trong team" không có nghĩa là "5 sudo user".

## Ticket NOVA-002

Depends on: `NOVA-001` hoàn tất (user vận hành + SSH key-based access phải
hoạt động trước khi tạo thêm account).

```text
Ticket ID: NOVA-002
Title: Thiết lập tài khoản & phân quyền cho team NovaCart theo least-privilege
Priority: P1
Severity: N/A (onboarding/RBAC task, không phải incident)
Business Context:
  NovaCart đã có team kỹ thuật 5 người (roster ở trên). Trước khi dev/QA
  khác chạm vào server, NovaCart yêu cầu mọi truy cập phải theo least-
  privilege và audit được — không ai ngoài DevOps/SRE có sudo toàn quyền,
  và người không cần shell thì không được cấp shell.
Problem:
  Sau NOVA-001, VPS chỉ có đúng 1 user (bạn) với sudo toàn quyền. Chưa có
  group nào ngoài mặc định, chưa có account cho dev/QA, chưa có audit trail
  rõ ràng cho từng người.
Expected Outcome:
  - Group `deploy` và `catalog-logs` tồn tại. Có 1 thư mục scaffold (ví dụ
    `/opt/catalog-service`) thuộc group `deploy`, có setgid bit, để Level 1
    dùng lại khi thật sự deploy.
  - `dev1`, `dev2` tồn tại, cả hai thuộc `deploy` + `catalog-logs`, KHÔNG
    thuộc `wheel`.
  - `dev2` có thêm 1 file riêng trong `/etc/sudoers.d/`, dùng `Cmnd_Alias`
    (danh sách lệnh cụ thể để trống/TODO — sẽ chốt ở `02-application-
    deployment.md` khi quyết định chạy bằng systemd unit hay `docker
    compose`).
  - `qa1` tồn tại, KHÔNG có shell thường: chỉ SFTP qua `ChrootDirectory` +
    `internal-sftp`, và có ACL read-only trên 1 thư mục log scaffold (ví dụ
    `/var/log/catalog-service`, tạo trống để dùng sau).
  - Không tạo Linux account cho Product Owner. Quyết định này được ghi lại
    kèm lý do (không phải "quên tạo").
  - `sudo -l -U <user>` cho mỗi account ra đúng danh sách quyền như thiết
    kế — không hơn.
Constraints:
  - Không cấp `wheel`/sudo toàn quyền cho ai ngoài bạn.
  - Docker chưa được cài (thuộc Level 1) → việc thêm `dev2` vào group
    `docker` và nội dung Cmnd_Alias cụ thể cho restart service BỊ DEFER
    sang `02-application-deployment.md`. Ticket này chỉ dựng khung (group,
    file sudoers tồn tại), chưa điền lệnh cụ thể.
  - Tự tay làm từng bước, không dùng script cấu hình có sẵn.
  - Không tự khoá bản thân (`you`) ra khỏi quyền sudo trong lúc thao tác.
Acceptance Criteria:
  - [ ] `getent group deploy catalog-logs` ra đúng 2 group, đúng member
        (`dev1`, `dev2`).
  - [ ] `ls -ld /opt/catalog-service` cho thấy group `deploy` và setgid bit.
  - [ ] `id dev1` và `id dev2` không có `wheel` trong danh sách group.
  - [ ] `sudo -l -U dev2` cho thấy đúng file `/etc/sudoers.d/dev2` áp dụng,
        không phải `ALL=(ALL) ALL`.
  - [ ] `ssh qa1@<vps-ip>` (shell thường) bị từ chối; `sftp qa1@<vps-ip>`
        hoạt động và chỉ thấy đúng thư mục đã chroot.
  - [ ] `getfacl /var/log/catalog-service` cho thấy `qa1` có `r-x`, không có
        quyền ghi.
  - [ ] Không có entry nào cho Product Owner trong `/etc/passwd`.
Technical Notes:
  - Group tồn tại để cấp/thu quyền cho nhiều người cùng lúc — nếu bạn thấy
    mình sửa quyền cho từng user riêng lẻ theo cùng một pattern, đó là dấu
    hiệu nên dùng group thay vì lặp lại thao tác.
  - `setgid` trên thư mục chỉ ảnh hưởng file/thư mục MỚI tạo sau đó — không
    tự đổi ownership của file đã có từ trước.
  - Mỗi file trong `sudoers.d/` áp dụng độc lập, đọc theo thứ tự tên file —
    thứ tự có thể quan trọng nếu hai rule mâu thuẫn nhau.
  - SFTP chroot jail của OpenSSH yêu cầu `ChrootDirectory` (và mọi thư mục
    cha của nó tính đến root) phải thuộc `root:root`, không được group/other
    writable. Sai bước này, sshd từ chối toàn bộ login của user đó — kể cả
    SFTP, không chỉ riêng shell.
  - ACL là lớp bổ sung, không thay thế owner/group/other. Khi debug quyền
    truy cập, nhớ check bằng `getfacl` — `ls -l` không hiện đủ thông tin ACL.
Reflection Questions (Feynman):
  1. Nếu `dev1` nghỉ việc, thu quyền qua group (`gpasswd -d dev1 deploy`)
     khác gì về bản chất so với việc bạn đã cấp quyền qua ACL riêng cho
     từng user? Cách nào dễ audit hơn khi team có 10 người, không phải 2?
  2. `dev2` có quyền từ group `deploy` VÀ một rule sudoers riêng của cá
     nhân. Nếu bạn xoá `dev2` khỏi group `deploy` nhưng quên xoá file
     sudoers, điều gì còn tồn tại, điều gì mất đi? Vì sao hai cơ chế này
     độc lập với nhau?
  3. `qa1` không có shell thường nhưng vẫn "vào" được server qua SFTP. Về
     bản chất, "có Linux account" và "có shell access" khác nhau ở đâu? Nếu
     kẻ tấn công lấy được password của `qa1`, họ làm được gì, không làm
     được gì?
  4. Product Owner không có Linux account nào — giả sử một ngày họ cần xem
     log để debug một vấn đề khẩn cấp, cách nào đúng để cấp truy cập tạm
     thời mà không tạo account vĩnh viễn cho họ? Vì sao "tạo account tạm rồi
     xoá" vẫn khác về bản chất so với "không tạo account từ đầu"?
```

## Handoff to Next Plan

Completed:
- (đang chờ người học hoàn thành NOVA-001, sau đó NOVA-002)

Artifacts Created:
- (chưa có — sẽ ghi lại các thay đổi thật trên VPS sau khi review)

Configuration Changed:
- (chưa có)

Decisions:
- Repo chính: `rajadilipkolli/spring-boot-microservices-series-v2`,
  bắt đầu bằng `catalog-service` only.
- Team roster 5 người + RBAC model (group/sudoers/ACL/chroot) — xem "Team
  Roster" và ticket NOVA-002 ở trên.

Known Issues:
- (chưa có)

Prerequisites for Next Plan:
- NOVA-001 và NOVA-002 hoàn tất, được senior mentor review pass.
- Riêng phần "dev2 vào group `docker`" và Cmnd_Alias cụ thể cho restart
  service trong NOVA-002 bị defer sang plan kế tiếp — phải hoàn thiện khi
  Docker được cài và cách chạy `catalog-service` (systemd unit hay `docker
  compose`) đã được quyết định.

Next Plan:
- `02-application-deployment.md` — clone repo, chạy `catalog-service` với
  Docker/Docker Compose trên VPS; hoàn thiện phần RBAC bị defer từ NOVA-002.
