| Người | Vai trò | Linux account | Quyền trên VPS |
|---|---|---|---|
| Bạn | DevOps/SRE (mới tuyển) | user tạo ở NOVA-001 | `wheel` — sudo đầy đủ |
| — | Developer | `dev1` | group `deploy`, `catalog-logs`; không sudo |
| — | Developer (senior, backup on-call) | `dev2` | group `deploy`, `catalog-logs`; + 1 file riêng trong `sudoers.d` (Cmnd_Alias giới hạn) |
| — | QA / Test Engineer | `qa1` | không có shell thường — chỉ SFTP chroot jail + ACL read-only trên log |
| — | Product Owner | *(không tạo account)* | không truy cập server; xem dashboard khi Level 4 (observability) có |

Nguyên tắc: 3/5 người **không** có sudo, 1/5 người **không** có account nào.
Đây là lesson chính — "5 người trong team" không có nghĩa là "5 sudo user".

groupadd deploy catalog-logs
sudo useradd -m -s "/bin/bash" dev1
sudo useradd -m -s "/bin/bash" dev2