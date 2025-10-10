# Screenshot-keylogging-email-sender
Chương trình gửi Chụp màn hình, key logger và gửi thông tin về mail
Chức năng:
- Tạo persistence (cố gắng tự chạy lại khi khởi động) bằng cách ghi sudoers.d và chỉnh crontab.
- Thu thập dữ liệu đầu vào từ thiết bị bàn phím /dev/input/event* và ghi vào file log.
- Chụp màn hình định kỳ (sử dụng scrot với DISPLAY và XAUTHORITY).
- Gửi đều đặn (periodic) các file log và ảnh chụp màn hình tới email qua SMTP (sử dụng libcurl và xác thực bằng password cứng trong mã).
- Tự khởi chạy hai luồng nền (thread): một luồng chụp màn hình và một luồng gửi email.
## Các Phần Chính Trong Code:
- keycodes[]: bảng ánh xạ mã phím (scan code) -> chuỗi ký tự (ví dụ "<ESC>", "a", "1"). Dùng để chuyển ev.code từ input_event thành text ghi vào log.- 

- screen_capture(void*):
    Tạo thư mục SCREENSHOT_DIR.
    Chu kỳ: chờ 5s, rồi vào vòng lặp vô hạn:
    - Lấy timestamp, tạo tên file .png.
    - Nếu chưa có, tìm file .Xauthority trong /home bằng find để lấy quyền truy cập X của user.
    - Gọi scrot với DISPLAY=:0 và XAUTHORITY trỏ tới file tìm được để chụp màn hình.
    - Sleep 10s (chụp theo chu kỳ ~10s).
    - Ghi chú: phụ thuộc vào sự tồn tại scrot và quyền truy cập vào session X của user.
    - find_keyboard_device():
    - Mở /proc/bus/input/devices và parse nội dung để tìm handler chứa Keyboard và kbd, trích eventN.
    - Trả về /dev/input/eventN (chuỗi đường dẫn thiết bị) hoặc rỗng nếu không tìm thấy.
- attach_files_from_dir(curl_mime* mime, const std::string& dir_path, const std::string& ext):
    Mở thư mục, lặp các file, với tên chứa ext thì thêm file đó là part của mime message để gửi email. (dùng curl_mime_* API).

- email_sender(void*):
    Vòng lặp vô hạn: sleep(20) (gửi mỗi 20s).
    Dùng libcurl để tạo message MIME: một phần text + attach tất cả .log từ LOG_DIR và .png từ SCREENSHOT_DIR.
    Cấu hình libcurl để kết nối smtps://smtp.gmail.com:465, đặt USERNAME, PASSWORD từ macro, thiết lập MAIL_FROM, MAIL_RCPT và header Subject.
    Gọi curl_easy_perform để gửi, rồi cleanup.

- install_crontab():
    Lấy đường dẫn thực thi hiện tại (/proc/self/exe).
    Xác định username (dùng SUDO_USER hoặc whoami).
    Viết 1 file sudoers tạm /tmp/keylogger_sudo với nội dung username ALL=(ALL) NOPASSWD: /path/to/exe rồi move vào /etc/sudoers.d/keylogger_sudo và chmod 440 (dùng sudo hệ thống qua system()).
    Tạo entry cron @reboot sudo /path/to/exe >/dev/null 2>&1 bằng cách đọc crontab -l sang /tmp/current_cron, kiểm tra xem đã có entry tương tự chưa, nếu chưa thì append và cài crontab.
    Mục đích: đảm bảo chương trình được chạy tự động khi khởi động (persistence), và có thể thực thi bằng sudo mà không cần mật khẩu.

- main():
    Gọi install_crontab().
    Tạo thư mục log.
    Tạo hai luồng con: screen_capture và email_sender, cả hai pthread_detach.
    Gọi find_keyboard_device(); mở device tương ứng để đọc input_event.

Tạo file log với định dạng tên theo thời gian; mở std::ofstream append.

Vòng lặp chính: đọc input_event từ file descriptor; nếu ev.type == EV_KEY && ev.value == 1 (sự kiện key press) và ev.code hợp lệ, lấy chuỗi từ keycodes[ev.code] và log << key.
