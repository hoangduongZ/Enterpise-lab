Chủ nhà làm sao để kiểm soát hành động của người dùng khác trong hệ thống ?
1. .bash_history
Kiểm soát qua tệp .bash_history trong thư mục home/{user}/.bash_history của họ
-> Nhược điểm: Nếu ngừoi dùng am hiểu về linux có thể crud nó được
Output minh hoạ:
```
ls -l
cd /var/www/html
nano index.html
ping google.com
sudo systemctl restart nginx
```
2. /var/log/audit
Đây là dữ liệu do kernel của hệ điều hành ghi lại, user không thể chạm tới