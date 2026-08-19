```
macbook@Hoangs-Macbook-M1-MAX Redis-lab % ssh root@103.252.137.184
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:bXh2znCrNPSRM6lYsNI+y/u2FL0PTVTiAmZp0suEcUs.
Please contact your system administrator.
Add correct host key in /Users/macbook/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/macbook/.ssh/known_hosts:4
Host key for 103.252.137.184 has changed and you have requested strict checking.
Host key verification failed.
```
> Đây là lỗi Cảnh báo thay đổi nhận dạng máy chủ (Host key verification failed).
> Lỗi này là một tính năng bảo mật của SSH. Nó xảy ra vì trước đây bạn đã từng kết nối vào IP 103.252.137.184, máy Mac của bạn đã lưu lại "chữ ký" (host key) của server đó. Nhưng hiện tại, "chữ ký" của server đã bị thay đổi nên máy Mac từ chối kết nối để bảo vệ bạn.
> ! Nguyên nhân phổ biến nhất: Bạn vừa cài lại hệ điều hành (rebuild/reinstall OS) cho VPS/Server này, hoặc địa chỉ IP này vừa được cấp cho một server hoàn toàn mới.
> Cách khắc phục. Bạn chỉ cần xóa "chữ ký" cũ đã lưu trên máy Mac bằng cách chạy lệnh sau trong Terminal:
```bash
ssh-keygen -R 103.252.137.184
```
> Thủ công: $HOME/.ssh/known_hosts xoá sign đi
---