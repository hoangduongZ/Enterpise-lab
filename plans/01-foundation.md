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

## Handoff to Next Plan

Completed:
- (đang chờ người học hoàn thành NOVA-001)

Artifacts Created:
- (chưa có — sẽ ghi lại các thay đổi thật trên VPS sau khi review)

Configuration Changed:
- (chưa có)

Decisions:
- Repo chính: `rajadilipkolli/spring-boot-microservices-series-v2`,
  bắt đầu bằng `catalog-service` only.

Known Issues:
- (chưa có)

Prerequisites for Next Plan:
- NOVA-001 hoàn tất và được senior mentor review pass.

Next Plan:
- `02-application-deployment.md` — clone repo, chạy `catalog-service` với
  Docker/Docker Compose trên VPS.
