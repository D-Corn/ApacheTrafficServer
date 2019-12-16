# [ApacheTrafficServer](https://trafficserver.apache.org/) as Caching Reverse Proxy
## Giới Thiệu
- Apache Traffic Server là một máy chủ proxy lưu trữ (Caching Proxy Server) có khả năng mở rộng cao có khả năng xử lý khối lượng lớn các yêu cầu đồng thời trong khi duy trì độ trễ rất thấp. Nó cung cấp giải pháp phần mềm hiệu năng cao và có thể mở rộng cho cả Forward Proxy lẫn Reverse Proxy.
- Nó có khả năng xử lý đồng thời một khối lượng lớn request trong lúc đang duy trì độ trễ rất thấp.
- Apache Traffic Server có những khả năng chuyển tiếp lưu lượng HTTP/HTTPS để tăng tốc cho người dùng cuối cũng như giảm tải cho server chính.
## Những đặc điểm chính
- Caching(lưu trữ): disk và trong memory caching cho các request thường xuyên nhất
- Proxy: hỗ trợ ca Forward Proxy lẫn Reverse Proxy vào tùy mục đích sử dụng
- Speed: Có thể xử lý hàng nghìn request mỗi giây
- Configure: File config có các hướng dẫn trên site docs của Apache Traffic Server rất cụ thể, chi tiết. Sẽ có các hướng dẫn theo mục đích sử dụng.
- Secure(bảo mật): Hỗ trợ tích hợp HTTPS/SSL
- Clustering(phân cụm): Hỗ trợ tích hợp cụm cho bộ nhớ cache (volume.config)

## Mô hình triển khai
<p align="center">
  <img src="https://docs.trafficserver.apache.org/en/7.0.x/_images/httprvs.jpg"
  <br/>
</p>

## Cài đặt Apache Traffic Server(ATS)
Ở lab này, chúng ta sẽ cài đặt ATS như là một Caching Reverse Proxy (máy chủ lưu trữ dưới dạng proxy ngược)
### Thiết lập
- Một máy vật lý Ubuntu 18.04 LTS(có thể sử dụng laptop) - 127.0.0.1
- Một Cloud Server Ubuntu 18.04\
(Cloud Server đã được máy vật lý ssh, đã mở port 80 và port 8080)

### Bước 1: Cài đặt Traffic Server
- Trên máy Cloud Server sẽ cài Traffic Server và cấu hình nó.

- Traffic Server có sẵn trên kho mặc định của Ubuntu 18.04 nên có thể cài từ **apt-get**.

- Trước hết hãy update các gói cài đặt, sử dụng các command sau:
``` 
 $sudo apt-get update
 $sudo apt-get install trafficserver
```
- Traffic Server truy cập trên cổng 8080 theo mặc định. Tuy nhiên, muốn truy cập được vào ta phải cấu hình cho nó và cài đặt Apache2

### Bước 2: Cài đặt Web Server Apache2
- Trên máy Cloud Server ta cài một Web Server bởi vì một Reverse Proxy hoạt động giữa người dùng và Web Server. Vậy nên ta sẽ cài đặt, ở đây ta demo Web Server Apache2.

- Cài đặt apache2 bằng **apt-get** với command:
```
$sudo apt-get install apache2
```
### Bước 3: Vô hiệu hoá quyền truy cập từ xa đến Web Server
- Apache cho phép quyền kết nối tới tất cả giao diện mạng theo mặc định. Bằng cách tạo cấu hình cho nó để cho phép quyền kết nối chỉ trên giao diện loopback, bạn có thể chắc chắn rằng người dùng từ xa sẽ không thể truy cập nó được.

- Mở **ports.conf** bằng **nano** để chỉnh sửa với command:
```
$sudo nano /etc/apache2/ports.conf
```
- Tìm dòng chứa **Listen 80** chỉ thị và đổi nó thành:
```
Listen 127.0.0.1:80
```
 Lưu file và thoát.

- Tiếp theo, mở **apache2.conf**.
```
$sudo nano /etc/apache2/apache2.conf
```
Lưu file và thoát.

- Khởi động lại Apache để áp dụng cấu hình vừa thay đổi bằng command sau:
```
$sudo service apache2 restart
```
### Bước 4: Tạo cấu hình cho Traffic Server như một Reverse Proxy
- Để Traffic Server như một Reverse Proxy, đầu tiên ta phải ánh xạ xem máy ATS sẽ làm Reverse Proxy cho server nào. Để làm điều đó ta sẽ cấu hình 
trong file [remap.config](https://docs.trafficserver.apache.org/en/latest/admin-guide/files/remap.config.en.html) của **trafficserver**.

- Dùng **nano** với command sau:
```
$sudo nano /etc/trafficserver/remap.config
```
- Hãy tạo một quy tắc ánh xạ với thông báo tất cả các request truy cập vào địa chỉ IP của máy chủ gốc 127.0.0.1:80 sẽ được địa chỉ IP
của máy ATS tiếp nhận.

- Thêm dòng sau vào cuối file **remap.config**.
```
map http://traffic_server_ip:8080/ http://127.0.0.1:80/
```
Lưu và thoát file.

- Khởi động lại ATS để áp dụng cấu hình với command sau:
```
$sudo traffic_ctl config reload
```
- Mở trình duyệt và truy cập đường dẫn **http://traffic_server_ip:8080/**. Nếu thấy trang bắt đàu của Apache tức là đã thành công trong việc tạo quy tắc ánh xạ.

### Bước 5: Cấu hình Traffic Server
- Trong Apache Traffic Server, file **records.config** chính là nơi cấu hình chính của ATS. Hãy đảm bảo rằng
các option sau đã có trong file **records.config**.
```
$sudo nano /etc/trafficserrver/records.config
```
```ssh
                                             records.config
                                  
                     
 CONFIG proxy.config.http.cache.http INT 1
 CONFIG proxy.config.reverse_proxy.enabled INT 1
 CONFIG proxy.config.url_remap.remap_required INT 1
 CONFIG proxy.config.url_remap.pristine_host_hdr INT 0
 CONFIG proxy.config.http.server_ports STRING 8080 443:ssl
```
- Thay đổi một chút mặc định với dòng **CONFIG proxy.config.http.server_ports STRING 8080 443:ssl**.

- Các ption trên cho phép bộ nhớ đệm(cache) và reverse proxy và các cổng sẽ lắng nghe cho lưu lượng
http(8080) và https(443:SSL).

**Cấu hình liên quan đến bộ nhớ đệm Cache:**

```ssh
                                       records.config
 
CONFIG proxy.config.http.cache.http INT 1
CONFIG proxy.config.http.cache.ignore_client_cc_max_age INT 1
CONFIG proxy.config.http.normalize_ae_gzip INT 1
CONFIG proxy.config.http.cache.cache_responses_to_cookies INT 1
CONFIG proxy.config.http.cache.cache_urls_that_look_dynamic INT 1
CONFIG proxy.config.http.cache.when_to_revalidate INT 0
CONFIG proxy.config.http.cache.required_headers INT 2
CONFIG proxy.config.http.cache.ignore_client_no_cache INT 1
```
```CONFIG proxy.config.http.cache.ignore_client_no_cache INT 1``` là thiết lập cho phép chúng ta bỏ qua các clients yêu cầu no-cache và cung cấp nội dung từ bộ nhớ cache nếu có.
```proxy.config.http.cache.ignore_client_cc_max_age``` là thiết lập cho phép Traffic Server bỏ qua bất kỳ headers **Cache-Control: max-age** từ clients.

**Cấu hình lưu trữ Storage**
- Trong Traffic Server, file ```storage.config``` là file định dạng lưu trữ cho cache.
- Dùng nano mở file storage.config:
```
$sudo nano /etc/trafficserver/storage.config
 ```                            
```

/var/cache/trafficserver 256M

```
- Chúng ta sẽ thấy 1 dòng mặc định như ở trên, ở đây bộ nhớ lưu trữ của cache mặc định ban đầu sẽ là 256Mb.
### Một số tools
- Như vậy, các cấu hình ở trên và những cấu hình mặc định bên trong các file config của TrafficServer đã giúp ta cấu hình được cơ bản về Apache Traffic Server.
- Sau khi cấu hình, ta cần reload lại Traffic Server. Để làm điều này ta có thể sử dụng câu lệnh: 
```
$sudo /etc/init.d/trafficserver restart
```
Hoặc cũng có thể dùng ```traffic_ctl``` có sẵn của ATS
```
traffic_ctl config reload
```
- Chúng ta cũng có thể sử dụng tool ```traffic_top``` để xem tổng quan về bộ đệm ẩn.
Và một số công cụ khác cần tham khảo qua [Command Line Utilities](https://docs.trafficserver.apache.org/en/latest/appendices/command-line/index.en.html) trên Docs của Apache Traffic Server.

## Kết luận
  Apache Traffic Server là một nền tảng Proxy lưu trữ ổn định, nhanh và có thể mở rộng. Bằng chứng chính là nó đã được Yahoo sử dụng để giải quyết rất nhiều request vào thời điểm Yahoo còn vận hành.
  
  Và việc cài đặt cơ bản về ATS cũng khá dễ hiểu, nhưng để có thể nắm vững về ATS và các options của nó thì sẽ cần nhiều thời gian để giải quyết.
  
  Bài viết chỉ là bước đệm đơn giản để nghiên cứu về ATS  và còn thiếu rất nhiều thứ cũng như c












