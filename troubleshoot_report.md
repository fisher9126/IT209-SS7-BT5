# Troubleshoot Report - Bài 5: Xử lý xung đột cổng (Address Already in Use)

> Thực hiện trên VPS Ubuntu Server (SSH, quyền root).

## 1. Vấn đề
Khởi chạy ứng dụng ở cổng `8082` báo lỗi `Address already in use` vì có một tiến trình
khác đang chiếm giữ cổng này. Cần chẩn đoán, tìm PID, và kết thúc tiến trình đó để giải phóng cổng.

## 2. Giả lập cổng 8082 bị chiếm dụng
```bash
nohup python3 -m http.server 8082 > /dev/null 2>&1 &
```

## 3. Chẩn đoán cổng mạng - tìm PID đang giữ cổng 8082
```bash
sudo ss -tlnp | grep 8082
```
Kết quả mẫu:
```
LISTEN 0  5  0.0.0.0:8082  0.0.0.0:*  users:(("python3",pid=4212,fd=3))
```
- Cột cuối trong ngoặc: tên tiến trình `python3` và `pid=4212` -> đây là PID đang chiếm cổng.
- (Có thể dùng thay thế: `sudo lsof -i :8082`.)

## 4. Kết thúc tiến trình chiếm cổng
```bash
kill 4212
```
Nếu `kill` thường không giải phóng được, dùng cưỡng chế:
```bash
kill -9 4212
```

## 5. Xác nhận cổng đã được giải phóng
```bash
sudo ss -tlnp | grep 8082
```
-> Không còn dòng nào trả về = cổng 8082 đã trống.

## 6. Khởi động lại dịch vụ để Java chiếm cổng 8082
```bash
systemctl restart spring-app.service
sudo ss -tlnp | grep 8082
```
Kết quả mong đợi: cổng 8082 giờ do tiến trình `java` (chạy app.jar) lắng nghe,
không phải tiến trình python giả lập ban đầu.

> Ghi chú: bước này cần `spring-app.service` (Session 07 EX3) thực sự chạy app Java bind cổng 8082.
> Nếu service đó cấu hình ExecStart kiểu `sleep` (không mở cổng) thì sẽ không thấy java giữ 8082 —
> cần ExecStart là lệnh java mở cổng, ví dụ: /usr/bin/java -jar /opt/app/app.jar (app lắng nghe 8082).

## 7. Kết quả thực tế

### Tái hiện lỗi "Address already in use"
Cổng 8082 đã bị chiếm từ trước (python3 pid 4212). Chạy thêm 1 python nữa -> thất bại:
```
root@fisher-9126:~# nohup python3 -m http.server 8082 > /dev/null 2>&1 &
[2] 4284
[2]+  Exit 1   nohup python3 -m http.server 8082 > /dev/null 2>&1
```

### ss trước khi kill (python đang giữ 8082)
```
root@fisher-9126:~# ss -tlnp | grep 8082
LISTEN 0  5  0.0.0.0:8082  0.0.0.0:*  users:(("python3",pid=4212,fd=3))
```
-> Tiến trình chiếm cổng: `python3`, PID = **4212**.

### Lệnh kill
```
root@fisher-9126:~# kill 4212
[1]+  Terminated   nohup python3 -m http.server 8082 > /dev/null 2>&1
```

### ss sau khi kill (cổng đã trống)
```
root@fisher-9126:~# ss -tlnp | grep 8082
(khong con dong nao -> cong 8082 da duoc giai phong)
```

### Xác nhận tiến trình đã chết
```
root@fisher-9126:~# kill -9 4212
-bash: kill: (4212) - No such process
```

## 8. Tóm tắt quy trình chẩn đoán
1. `ss -tlnp` / `lsof -i :8082` -> xác định PID + tên tiến trình chiếm cổng.
2. `kill <PID>` (hoặc `kill -9` nếu lì) -> giải phóng cổng.
3. Xác nhận cổng trống bằng `ss` lần nữa.
4. Khởi động lại dịch vụ đích để nó chiếm cổng hợp lệ.
