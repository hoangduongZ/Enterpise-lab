# 🏢 Bản Vẽ Sơ Đồ Căn Hộ Linux (Root Directory '/')

Mọi thứ trong Linux đều bắt đầu từ gốc `/` (Đại sảnh). Đây là bản vẽ các khu vực chính của toàn bộ hệ thống:

## 1. 🛠️ Khu Vực Công Cụ & Máy Móc (Binaries & Libraries)
*   **/bin & /usr/bin**: Hộp đồ nghề cơ bản. Chứa các câu lệnh hàng ngày ai cũng dùng được (như `ls`, `cd`).
*   **/sbin & /usr/sbin**: Hộp đồ nghề hạng nặng. Chứa các công cụ quản trị hệ thống chỉ dành cho người có quyền cao nhất.
*   **/lib, /lib64, /usr/lib**: Kho linh kiện. Chứa các đoạn mã dùng chung (libraries) để các phần mềm lấy ra sử dụng.

## 2. 🎛️ Khu Trung Tâm Điều Khiển & Động Cơ
*   **/etc**: Bảng điều khiển trung tâm. Chứa TOÀN BỘ các tệp tin cấu hình (luật lệ, cài đặt) của các phần mềm trong hệ thống.
*   **/boot**: Cục khởi động. Chứa những tệp tin đầu tiên để mồi lửa đánh thức "bộ não" Linux (Kernel) khi bật máy.

## 3. 👻 Vùng Đất Của Ảo Ảnh (Ảo hóa trên RAM)
*(Những thư mục này không ghi trên ổ cứng cứng, chúng là dữ liệu sống của hệ điều hành!)*
*   **/dev (Devices)**: Nơi Linux biến các thiết bị vật lý (ổ cứng, USB, card mạng) thành các tệp tin. (Chân lý: Mọi thứ đều là tập tin).
*   **/proc (Processes) & /sys**: Cửa sổ nhìn vào dòng suy nghĩ của hệ điều hành. Chứa các thông tin về phần cứng và các chương trình đang chạy ngay trong RAM.

## 4. 🛋️ Không Gian Sinh Hoạt & Thùng Rác
*   **/root**: Căn phòng Master của ông chủ nhà quyền lực nhất (`root`).
*   **/home**: Khu phòng trọ cho các người dùng (User) bình thường khác.
*   **/var (Variable)**: Thùng chứa những dữ liệu liên tục phình to ra theo thời gian (như sổ ghi chép Log lỗi, cơ sở dữ liệu).
*   **/tmp (Temporary)**: Thùng rác tạm thời. Bị quét sạch mỗi khi khởi động lại máy.

---
> 💡 **Chú ý:** Nếu dùng lệnh `ls -l /` mà thấy các thư mục có dấu chấm `.` ở cuối phần quyền hạn (ví dụ: `dr-xr-xr-x.`), điều đó có nghĩa là căn phòng này đang bị Đội vệ sĩ **SELinux** giám sát nghiêm ngặt!