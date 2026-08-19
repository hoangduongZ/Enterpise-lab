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
  - [ ] `ssh <user>@<vps-ip>` bằng SSH key thành công, không hỏi password.
  - [ ] `ssh root@<vps-ip>` (hoặc bất kỳ hình thức password login nào) bị
        từ chối.
  - [ ] User vận hành chạy được `sudo` sau khi nhập password của chính nó.
  - [ ] Firewall đang bật, `<firewall tool> status` cho thấy chỉ các cổng
        cần thiết được mở.
Technical Notes:
  - Nếu tự khóa mình ra ngoài, hầu hết nhà cung cấp VPS có "recovery
    console/VNC console" trên web control panel để vào lại mà không cần SSH.
Reflection Questions (Feynman):
  1. Private key rơi vào tay người khác nhưng không có passphrase — họ
     login được không? Vì sao? Passphrase bảo vệ cái gì, không bảo vệ cái gì?
  2. User thường vẫn sudo lên root được — vậy tắt "root login qua SSH" thực
     sự ngăn được điều gì? Bản chất khác nhau giữa "là root" và "login
     bằng tài khoản root" là gì?
  3. Nếu bạn bật firewalld trước khi chắc chắn service ssh nằm trong zone
     đang active, chuyện gì sẽ xảy ra ngay lập tức? Tại sao thứ tự thao tác
     ở bước này quan trọng hơn bản thân câu lệnh?
  4. SELinux và firewalld giải quyết hai loại rủi ro khác nhau. Nếu phải
     giải thích cho một dev không rành ops, bạn sẽ nói firewalld chặn gì,
     SELinux chặn gì, bằng ví dụ cụ thể?
```

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
