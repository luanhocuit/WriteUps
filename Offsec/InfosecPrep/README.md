	# InfosecPrep

![](title.png)

![](description.png)

**Target** : Lab này yêu cầu chúng ta khai thác website để tìm thông tin bị lộ nhằm lấy được SSH private key, sau đó tìm cách để đăng nhập vào được thông qua SSH. Từ đó, leo thang đặc quyền lên quyền `root` để lấy được file `proof.txt` và `local.txt`.
> Ý tưởng đơn giản sau khi đọc Description của đề xong là mình sẽ quét bằng nmap để xem thử có bị lộ link nào không ? Nếu có thì check -> Nếu ra private key luôn thì tốt, mình sẽ dùng cái đó đăng nhập vào thông qua SSH.

## 1. Recon & Vulnerability Assessment

Đầu tiên, chúng  ta sẽ dùng lệnh `nmap` để quét xem port nào đang mở và dịch vụ liên kết với web có gì ?
```bash
nmap -sC -sV <IP>
```
`-sC` : Hiển thị script mặc định của `nmap` (tương tác một ít với web) 
`-sV` : Hiển thị **version** của dịch vụ

![](nmap_scan.png)

Sau khi chạy xong, ta thấy được port 22(ssh) và 80(http) đang mở và thấy được file rất khả nghi `/secret.txt` và mình đọc thử file.
![](secret.png)

=> Nhìn qua có thể thấy đây là nội dung đã được base64 encode
> Decode ra  thì có thể thấy đây là SSH Private Key

Các bạn hãy lưu nội dung đó về và `chmod` để sử dụng private key này :
```bash
base64 -d key.b64 > id_rsa // key.b64 lưu giá trị của /secret.txt
chmod 600 id_rsa
```

## 2. Initial Access

Để vào được thông qua SSH thì mình phải biết username nên mình sẽ dò tìm username bằng `wpscan` thử
```bash
wpscan --url <URL> -e u
```
![](wpscan.png)

=> Chỉ thấy mỗi user = admin, mình sẽ thử login vào thông qua lệnh :
```bash
ssh -i id_rsa admin@192.168.127.89
```

![](ssh_login_admin.png)

> Vậy là private key này không thể dùng để đăng nhập vào tài khoản admin, nên mình phải kiếm tài khoản khác dể đăng nhập vào thử

Khi mà vào web thì mình có thấy `OSCP Voucher` là title nên có thể đây là hint về tên user.

![](main.png)

-> Mình sẽ thử đăng nhập vào qua username `oscp`

![](ssh_login_oscp.png)

> Mình đã  đăng nhập thành công vào shell với quyền user `oscp`

## 3. Privilege Escalation

Sau khi vào được shell dưới  quyền `oscp`, chúng ta sẽ tìm cách leo lên quyền `root`

Đầu tiên mình sẽ check xem `/var/www/html/wp-config.php` có lưu password để mình có thể vào admin không ?

![](wp_config.png)

Mình thử đăng nhập vào root với username là `admin` và password là `0scp12345!`
>Kết quả : Incorrect password

-> Mình sẽ tìm hướng khác là dùng `find` để tìm các file chứa SUID để leo lên root
```bash
find / -type f -perm /4000 2>/dev/null
```

![](bin_bash_suid.png)

> Ta có thể thấy /bin/bash có chứ SUID nên mình sẽ thử dùng /bin/bash để leo lên root

```bash
/bin/bash -p
```

![](root.png)

> **Kết quả :** Chúng ta đã leo lên được quyền root

## 4. Result

Chúng ta sẽ tìm file `proof.txt` và `local.txt` và lấy flag

![](flag.png)

> 🚩Flag :
> `proof.txt` :  2059ee008652086d9a05d66ccbbd346c
> `local.txt` : 00eba666bbc5036d776910e6c537a920