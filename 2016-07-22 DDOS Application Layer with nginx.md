## DDOS Application Layer

### HTTP Flood

#### Nguyên tắc hoạt động

Gửi các yêu cầu HTTP thành dạng các bản nhỏ, và truyền với tốc độ chậm tới web server hoặc đọc dữ liệu một cách nhỏ giọt => Server sẽ luôn chờ để nhận dữ liệu HTTP đầy đủ

#### Các hình thức tấn công

- HTTP Slow header (slowloris)
- HTTP Slow body
- HTTP slow read

**HTTP slow header**

Cố tình không gửi `CRLF` để hoàn tất request header dẫn tới server phải giữ kết nối. Trạng thái chờ này sẽ kéo dài cho đến khi nhận được đủ request hoặc hết timeout (theo mặc định là 300s). Quá 5 phút này, socket ấy sẽ bị hủy.

**HTTP slow body (slow POST)**

Tương tự `HTTP slow header` nhưng ở đây sẽ tiến hành làm chậm quá trình gửi nội dung thông điệp.

**HTTP slow read**

Gửi request hoàn chỉnh nhưng lại đọc dữ liệu nhỏ giọt từ server.

Cách phát hiện: attack tìm kiếm các nguồn tài nguyên tạo ra bởi server mà có kích thước lớn hơn kích thước bộ đệm của server (thường từ 65 - 128Kb). Sau đó xác định kích thước bộ đệm đọc < kích thước bộ đệm.

Khi server gửi response, attack nhận gói tin đầu tiên, và thông báo lại kích thước của sổ TCP nhỏ nhất có thể (Server dựa vào đây để gửi gói tin tiếp theo)

#### Cách phòng chống

Đối với tấn công `slow header` và `slow body`, trên nginx ta thiết lập lại 2 thông số để giới hạn thời gian sống của `header` cũng như `body`

```
server{
	client_header_timeout 	5s;
	client_body_timeout		5s;
	// default: 60s
}
```

Song song đó, nginx có cơ chế buffering data từ HTTP header của client. Những dạng này ở trạng thái INVAILD (VD: thiếu CRLF) thì loại bỏ.

Đối với `slow read`:

- Không chấp nhận các kết nối với giá trị windown size nhỏ bất thường
- Hạn chế tuyệt đối kết nối với thời gian hợp lý
- Không cho phép kết nối liên tục (persistent connections) trừ khi hưởng lợi từ nó.[1]

### Một số cách phòng tránh khác

**Giới hạn số lượng request**

```
limit_req_zone $binary_remote_addr zone=one:10m rate=`30r/m`;
server{
	location /login.php {
		limit_req zone = one;
	}
}
```

Cho phép mỗi một địa chỉ IP chỉ được truy cập tới `/login.php` 2 giây 1 lần.

**Giới hạn số kết nối**

```
limit_req_zone $binary_remote_addr zone=one:10m
server{
	location /store/{
		limit_conn addr 10;
	}	
}
```

Giới hạn kết nối tới `/store/` tối đa là 10 kết nối với mỗi IP.

**Giới hạn kết nối tới backend**

```
uptream website{
	server 172.16.22.15:8080 max_conns=200;
	queue 10 timeout 30s;
}
```
Giới hạn kết nối tối đa tới backend server là 200 kết nối. `queue`: số kết nối tối đa được giữ lại khi backend đã đạt 200 kết nối. `timeout` thời gian làm mới các request trong hàng đợi. [2]


### Tham khảo

[1]. https://blog.qualys.com/tag/slow-http-attack
[2]. https://www.nginx.com/blog/mitigating-ddos-attacks-with-nginx-and-nginx-plus/