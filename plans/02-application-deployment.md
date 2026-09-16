# 02 — Application Deployment (Level 1: Deploy catalog-service)

Depends on: [01-foundation.md](01-foundation.md) — NOVA-001 (SSH/user/firewall)
và NOVA-002 (RBAC team) đúng ra phải **hoàn tất và được review** trước khi
mở file này (xem "Prerequisites for Next Plan" trong `01-foundation.md`).

> **Ghi chú thứ tự (2026-09-09):** plan này được tạo **trước khi** Level 0
> hoàn tất, theo yêu cầu trực tiếp của người học (muốn xem trước lộ trình
> Level 1). Đây là ngoại lệ có chủ đích — xem "Important Decisions" trong
> [00-master-plan.md](00-master-plan.md). Không được bắt đầu thực thi
> (`how/NOVA-003-guide.md` trở đi) khi NOVA-001/NOVA-002 chưa xong; đọc
> trước để hình dung, làm sau theo đúng thứ tự.

## Scope

- **Input:** VPS đã có (khi Level 0 xong): SSH key-based access, user vận
  hành + `dev1`/`dev2`/`qa1` theo RBAC của NOVA-002, firewall cơ bản chỉ mở
  `ssh`, thư mục scaffold `/opt/catalog-service` (group `deploy`, setgid).
- **Output:** `catalog-service` (từ repo đã chọn ở `01-foundation.md`) chạy
  thật trên VPS bằng Docker Compose, nối Postgres thật (không Testcontainers),
  có persistent volume, có reverse proxy + TLS (tạm self-signed) đứng trước,
  firewall chỉ mở đúng port cần thiết, và phần RBAC bị defer từ NOVA-002
  (quyền Docker cho `dev2`) được hoàn tất.
- **Out of scope (để phase sau):** Kafka, `config-server`, `service-registry`
  (Eureka), `api-gateway`, các service khác (`inventory`/`order`/`payment`) —
  đúng quyết định "monolith seed trước" ở `00-master-plan.md`. CI/CD tự động
  (GitHub Actions) thuộc `03-ci-cd.md`. Domain/DNS thật + Let's Encrypt thật
  thuộc phase sau (khi có domain — xem Known Risks).

## Repository Facts (verified qua GitHub, 2026-09-09)

Xác nhận trực tiếp từ repo thật (`rajadilipkolli/spring-boot-microservices-series-v2`,
thư mục `catalog-service/`) — không suy đoán:

- README xác nhận service nghe port **`18080`**, context-path
  **`/catalog-service`** (`swagger-ui.html`, `actuator`, `actuator/health` đều
  nằm dưới path này).
- `catalog-service/docker/docker-compose-app.yml` (compose mẫu của chính
  service này) yêu cầu đúng các biến môi trường:
  `SPRING_PROFILES_ACTIVE=docker`, `SPRING_DATASOURCE_DRIVER_CLASS_NAME`,
  `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`,
  `SPRING_DATASOURCE_PASSWORD` — đây là hợp đồng thật giữa app và DB.
- `catalog-service/docker/docker-compose.yml` (compose hạ tầng dùng chung của
  repo) gộp chung Postgres + Kafka + Config Server + Eureka + cả
  `inventory-service` trong **1 file** — đây là lý do NOVA-003 bên dưới cấm
  dùng nguyên file này.
- Không tìm thấy `Dockerfile` trần ở gốc `catalog-service/` — nhiều khả năng
  build image qua Spring Boot Maven plugin (Buildpacks) hoặc tương tự, không
  phải Dockerfile thủ công. Việc đầu tiên khi thực thi ticket là tự xác nhận
  lại bằng `pom.xml`/README, không giả định.
- `src/main/resources` chỉ có `application.properties` +
  `application-local.properties` (không phải `.yml`) — không có sẵn
  `application-docker.properties`; cấu hình cho môi trường Docker chủ yếu đi
  qua biến môi trường (đúng như compose mẫu ở trên), không qua file profile
  riêng.

## Environment

- Vẫn AlmaLinux 8.9 — cài Docker Engine qua **repo chính thức của Docker**
  cho RHEL/CentOS (`dnf config-manager --add-repo ...`), không dùng
  `podman-docker` mặc định của distro (khác API, khác hành vi).
- Chưa có domain thật trỏ vào VPS (ghi nhận ở Known Risks của
  `00-master-plan.md`) — NOVA-005 dùng self-signed cert tạm.

## Level 1 Goals

1. `catalog-service` chạy thật trên VPS, nối Postgres thật, có persistent
   volume — không phụ thuộc Testcontainers ở runtime.
2. Secrets (DB password...) nằm trong `.env`, không hardcode trong compose.
3. Phần RBAC bị defer từ NOVA-002 (quyền vận hành Docker cho `dev2`) được
   quyết định và hoàn tất — không để treo mãi.
4. Có reverse proxy + TLS đứng trước app, firewall chỉ mở đúng port cần
   thiết — không expose port ứng dụng (`18080`) trực tiếp ra internet.
5. Người học giải thích được **vì sao** không dùng nguyên compose/RBAC mẫu
   có sẵn của repo, không chỉ copy-paste cho chạy được.

## Ticket NOVA-003

```text
Ticket ID: NOVA-003
Title: Clone và chạy catalog-service bằng Docker Compose với Postgres thật
Priority: P1
Severity: N/A (deployment task)
Business Context:
  NovaCart cần đưa service catalog đầu tiên lên VPS thật, dùng Postgres thật
  thay vì Testcontainers (chỉ dùng cho test), chuẩn bị nền cho CI/CD và các
  service khác ở phase sau.
Problem:
  VPS (sau Level 0) có user/group/firewall nhưng chưa có Docker, chưa có
  source code, chưa có gì chạy thật. Thư mục scaffold /opt/catalog-service
  đang trống.
Expected Outcome:
  - Docker Engine + Compose plugin cài qua repo chính thức của Docker,
    service docker chạy (systemctl enable --now docker).
  - Source code catalog-service (chỉ phần này, không cần toàn bộ monorepo)
    nằm trong /opt/catalog-service, vẫn giữ group deploy + setgid.
  - Một file docker-compose.yml TỰ VIẾT, tối giản: chỉ 2 service
    (catalog-service + postgres) — KHÔNG dùng nguyên
    catalog-service/docker/docker-compose.yml của repo (file đó kéo theo
    Kafka/Eureka/Config Server/inventory-service, ngoài phạm vi Level 0-2).
  - Postgres dùng persistent volume (named volume hoặc bind mount, tự chọn
    và giải thích được đánh đổi) — restart container không mất data.
  - Toàn bộ secret (DB password...) nằm trong .env, không hardcode trong
    docker-compose.yml, không commit .env nếu thư mục có git riêng.
  - catalog-service container start thành công, kết nối được Postgres thật,
    trả actuator health UP.
Constraints:
  - Không dùng docker-compose.yml/docker-compose-app.yml gốc của repo
    nguyên trạng — viết file compose riêng.
  - Không dùng Testcontainers ở runtime thật trên VPS.
  - Tự xác nhận port/context-path/cách build image bằng cách đọc
    README/pom.xml của repo thật — không đoán, không bịa tên biến môi
    trường (danh sách biến bắt buộc đã liệt kê ở "Repository Facts").
  - Không chạy thao tác Docker hàng ngày bằng root trực tiếp.
Acceptance Criteria:
  - [ ] `docker compose ps` cho thấy 2 container đang Up (catalog-service +
        postgres).
  - [ ] curl vào đúng actuator health endpoint (port/context-path xác nhận
        từ repo) trả về status UP.
  - [ ] `docker volume ls` cho thấy 1 volume riêng cho Postgres data.
  - [ ] Sau `docker compose restart postgres`, 1 record test tự tạo trước
        đó vẫn còn nguyên (chứng minh persistent volume hoạt động).
  - [ ] `.env` không xuất hiện trong `git status` (nếu có git ở đây) hoặc có
        trong `.gitignore`.
  - [ ] File/thư mục mới trong /opt/catalog-service vẫn thuộc group deploy.
Technical Notes:
  - Xem mục "Repository Facts" ở đầu plan này — đó là toàn bộ thông tin thật
    đã verify, không cần đoán thêm biến môi trường nào khác.
  - depends_on trong Docker Compose (bản không dùng healthcheck) chỉ đảm bảo
    THỨ TỰ START container, không đảm bảo Postgres đã sẵn sàng NHẬN
    connection — đây là nguồn lỗi race-condition rất phổ biến.
Reflection Questions (Feynman):
  1. Vì sao ticket cấm dùng nguyên docker-compose.yml gốc của repo dù nó
     "chạy được ngay", trong khi bạn phải tự viết bản tối giản hơn? Đánh đổi
     ở đây là gì?
  2. .env chứa password Postgres dạng plain text trên đĩa VPS — việc này an
     toàn hơn hardcode trong compose ở điểm nào, và KHÔNG an toàn hơn ở
     điểm nào?
  3. Named volume khác gì bind mount trỏ ra /opt/catalog-service/pgdata? Nếu
     VPS bị cài lại OS nhưng ổ đĩa data giữ nguyên, cách nào phục hồi data
     dễ hơn, vì sao?
  4. depends_on có đảm bảo Postgres "sẵn sàng nhận connection" trước khi
     catalog-service tự kết nối không? Nếu không, catalog-service nên xử lý
     tình huống đó thế nào (retry? fail cứng?), và bạn kiểm chứng điều đó
     bằng cách nào thay vì đoán?
```

## Ticket NOVA-004

Depends on: `NOVA-003` hoàn tất (đã biết rõ cách chạy `catalog-service` bằng
Docker Compose). Đây chính là phần bị defer từ `NOVA-002`
("dev2 vào group docker và Cmnd_Alias cụ thể" — xem
[01-foundation.md](01-foundation.md#ticket-nova-002)).

```text
Ticket ID: NOVA-004
Title: Hoàn tất RBAC bị defer từ NOVA-002 cho quyền vận hành Docker của dev2
Priority: P2
Severity: N/A (RBAC follow-up, không phải incident)
Business Context:
  NOVA-002 cố ý để trống nội dung Cmnd_Alias cho dev2 vì lúc đó chưa cài
  Docker, chưa biết cách vận hành catalog-service. NOVA-003 vừa xong, giờ
  biết rõ: catalog-service chạy qua Docker Compose tại /opt/catalog-service.
Problem:
  dev2 hiện chỉ có 1 rule vô dụng (Cmnd_Alias trỏ /bin/true) trong sudoers,
  không vận hành được gì thật. Câu hỏi treo từ NOVA-002 (thêm dev2 vào group
  docker hay không) chưa có quyết định.
Expected Outcome:
  - Quyết định rõ ràng, ghi lại lý do: dev2 KHÔNG được thêm vào group
    docker (group này tương đương root — ai trong group docker có thể mount
    toàn bộ filesystem host vào 1 container rồi thao túng từ bên trong).
  - Thay vào đó, /etc/sudoers.d/dev2 được cập nhật: thay
    Cmnd_Alias CATALOG_SVC_TODO = /bin/true bằng lệnh compose thật, giới hạn
    đúng đường dẫn compose file cụ thể (không dùng wildcard rộng).
  - dev1 (không phải on-call) vẫn KHÔNG có quyền chạy bất kỳ lệnh docker nào
    qua sudo, đúng roster ("dev2 kiêm backup on-call").
Constraints:
  - Không thêm bất kỳ ai (kể cả dev2) vào group docker — quyết định giữ
    nguyên trừ khi có lý do mới, ghi lại rõ ràng nếu đổi.
  - Cmnd_Alias phải giới hạn đúng 1 compose file cụ thể
    (/opt/catalog-service/docker-compose.yml), không cho phép chạy
    docker compose với file bất kỳ ở nơi khác trên hệ thống.
Acceptance Criteria:
  - [ ] `getent group docker` (nếu group này tồn tại sau khi cài Docker)
        KHÔNG chứa dev1 hay dev2.
  - [ ] `sudo -l -U dev2` hiện đúng lệnh compose cụ thể, không phải wildcard
        hay ALL=(ALL) ALL.
  - [ ] Đăng nhập bằng dev2: lệnh được cấp qua sudo chạy được; `docker ps`
        chạy trực tiếp (không qua sudo) bị từ chối vì không có quyền truy
        cập docker socket.
Technical Notes:
  - Group docker mặc định không cần password để dùng docker socket — về
    bản chất tương đương cấp sudoers "ALL=(ALL) NOPASSWD: ALL", vì một
    container có thể mount -v /:/host rồi thao túng toàn bộ host từ bên
    trong.
  - sudo -l -U dev2 chỉ hiển thị quyền được CẤU HÌNH, không xác nhận lệnh đó
    thực sự chạy được — luôn test thật bằng cách login là dev2.
Reflection Questions (Feynman):
  1. Vì sao "thêm user vào group docker" và "cấp sudo ALL" lại được coi là
     tương đương nhau về rủi ro, dù group docker nghe có vẻ vô hại hơn hẳn?
  2. Nếu Cmnd_Alias của dev2 giới hạn đúng 1 file compose nhưng KHÔNG giới
     hạn nội dung lệnh con bên trong (ví dụ vẫn cho phép
     docker compose -f <file> exec catalog-service /bin/sh), dev2 có thể
     lợi dụng điều đó để làm gì ngoài ý định ban đầu ("restart service")?
  3. So với việc để trống hoàn toàn (lúc NOVA-002 chưa xong), trì hoãn
     quyết định tới khi có đủ thông tin mang lại lợi ích gì so với "đoán
     bừa và cấp quyền sớm cho xong"?
```

## Ticket NOVA-005

Depends on: `NOVA-003` hoàn tất (`catalog-service` đã chạy thật, biết đúng
port nội bộ).

```text
Ticket ID: NOVA-005
Title: Reverse proxy + TLS tạm thời + firewall trước khi public catalog-service
Priority: P1
Severity: N/A (deployment/security task)
Business Context:
  catalog-service hiện chỉ nghe port 18080 nội bộ trên VPS — chưa ai bên
  ngoài truy cập được qua HTTP(S) chuẩn, và bản thân port 18080 không nên
  public trực tiếp.
Problem:
  Chưa có domain thật trỏ vào VPS (ghi nhận ở Known Risks của
  00-master-plan.md) — chưa thể lấy chứng chỉ Let's Encrypt thật ngay bây
  giờ. Firewall hiện chỉ mở ssh.
Expected Outcome:
  - Reverse proxy (Nginx hoặc Caddy — tự chọn, giải thích lý do) forward từ
    port 80/443 vào 127.0.0.1:18080, KHÔNG expose 18080 ra ngoài.
  - TLS termination tại reverse proxy dùng self-signed certificate tạm thời
    (vì chưa có domain) — ghi rõ đây là giải pháp tạm, nêu bước chuyển sang
    Let's Encrypt thật khi có domain (thay đổi tối thiểu, không đổi kiến
    trúc).
  - firewalld chỉ mở đúng port cần thiết: ssh, http, https. Port 18080
    KHÔNG được mở ra ngoài qua firewall.
Constraints:
  - Không mở port 18080 ra internet dưới bất kỳ hình thức nào.
  - Không dùng domain giả/tự bịa để "cho giống thật" — nếu cần tên để test
    SNI, dùng chính IP hoặc một tên nội bộ đánh dấu rõ "chưa phải domain
    thật".
Acceptance Criteria:
  - [ ] `firewall-cmd --list-all` chỉ hiện ssh, http, https — không có port
        18080.
  - [ ] Từ máy Mac, curl thẳng http://<vps-ip>:18080/... bị timeout/refused.
  - [ ] Từ máy Mac, `curl -k https://<vps-ip>/catalog-service/actuator/health`
        (qua reverse proxy) trả về UP.
  - [ ] Reverse proxy tự redirect HTTP (80) sang HTTPS (443).
Technical Notes:
  - Self-signed cert khiến trình duyệt/curl cảnh báo không tin cậy — đây là
    lý do dùng -k khi test, KHÔNG phải lỗi cấu hình.
  - Khi có domain thật, chỉ cần đổi phần cert issuance (ví dụ Caddy tự động
    xin Let's Encrypt nếu domain trỏ đúng DNS) — kiến trúc reverse proxy
    phía sau không đổi.
Reflection Questions (Feynman):
  1. Vì sao để reverse proxy đứng giữa (thay vì catalog-service tự nghe
     443 và tự làm TLS) lại là pattern chuẩn trong production, ngay cả khi
     chỉ có 1 service duy nhất?
  2. Nếu bạn lỡ quên chặn port 18080 ở firewalld nhưng reverse proxy ở 443
     vẫn chạy đúng — hệ thống có "trông như" vẫn an toàn không? Rủi ro thật
     sự nằm ở đâu?
  3. curl -k bỏ qua việc xác minh chứng chỉ — trong ngữ cảnh nào dùng -k để
     test là chấp nhận được, và trong ngữ cảnh nào (ví dụ production thật)
     dùng -k lại là dấu hiệu nguy hiểm?
```

## Handoff to Next Plan

Completed:
- (đang chờ Level 0 hoàn tất, sau đó NOVA-003 → NOVA-004 → NOVA-005)

Artifacts Created:
- `plans/02-application-deployment.md` (this file).
- (chưa có artifact thật trên VPS — file compose/`.env`/reverse proxy config
  sẽ ghi lại ở đây sau khi người học thực hiện và được review, kèm
  `how/NOVA-003-guide.md`, `how/NOVA-004-guide.md`, `how/NOVA-005-guide.md`
  tương ứng, tạo khi bắt đầu thực thi từng ticket.)

Configuration Changed:
- (chưa có)

Decisions:
- Không dùng compose/RBAC mẫu có sẵn của repo nguyên trạng — tự viết bản tối
  giản, đúng tinh thần "học, không copy" đã quyết ở `01-foundation.md`.
- `dev2` không được thêm vào group `docker` dưới bất kỳ hình thức nào —
  luôn qua `sudoers.d` với `Cmnd_Alias` giới hạn.
- TLS dùng self-signed tạm thời cho đến khi có domain thật.

Known Issues:
- Chưa có domain thật trỏ vào VPS — NOVA-005 dùng giải pháp tạm, cần một
  ticket riêng ở phase sau khi có domain để chuyển sang Let's Encrypt thật.

Prerequisites for Next Plan:
- NOVA-001 → NOVA-005 hoàn tất, được senior mentor review pass.
- Quyết định rõ: có thêm Kafka thật ở Level 3 hay tiếp tục trì hoãn (xem
  Known Risks của `00-master-plan.md`).

Next Plan:
- `03-ci-cd.md` — GitHub Actions build/test/image/registry/deploy/rollback
  cho `catalog-service`, dùng lại pattern RBAC/sudoers vừa hoàn thiện ở
  NOVA-004 cho bước "CI deploy tự động chạy lệnh gì trên VPS".
