Khai thác
Để bắt đầu quá trình xây dựng và thử nghiệm, chúng tôi tạo ra một trang web có lỗ hổng và xây dựng chúng bằng Docker. 
Việc xây dựng image được thực hiện qua việc sử dụng Dockerfile và Docker compose. Sau khi chạy lệnh “docker compose up”, sẽ có tập lệnh tự động đọc Dockerfile để tạo ra image, tải các image có sẵn trên DockerHub và tự động tạo ra các container. Việc xây dựng image trên Docker có thể đóng gói các ứng dụng và phụ thuộc của chúng vào một hệ thống duy nhất và có thể chạy trên nhiều máy chủ khác nhau mà không cần phải cài đặt các phần mềm và thư viện tương tự trên từng máy chủ. Điều này giúp cho việc triển khai ứng dụng trên môi trường sản xuất trở nên dễ dàng và nhanh chóng hơn.
Trong file “docker-compose.yml”, chúng ta đã thực hiện việc mount /var/run/docker.sock lên hệ thống container. Container cũng có thể truy cập vào Docker socket trên host thông qua đường dẫn “/custom/docker/docker.sock” trong container. 
Để bắt đầu, ta tiến hành khai thác lỗ hổng bảo mật có trên web.
Thu tập thông tin từ trang web, ta biết được những thông tin của trang web như sau:


Hình 3. 1 Các công nghệ sử dụng
Ở đây, ta có thể biết được một số thông tin như sau: 
	Trang web được xây dựng bằng ngôn ngữ PHP. 
	Trang web được chạy trên trên server Apache2. 3. Hệ điều hành của máy chủ là Debian
Khi vào trang website mà container tạo ra, ta được giao diện sau:
 
Hình 3. 2 Giao diện trang web
Vì khi vào trang web, trang web yêu cầu chúng ta đăng nhập, tuy nhiên, chúng ta lại không có tài khoản. Do đó, ta vào trang đăng kí tài khoản để tạo tài khoản đăng nhập vào trang web.
 
Hình 3. 3 Giao diện đăng kí tài khoản
Ở trang đăng kí tài khoản, ta thấy có một chức năng giúp ta có thể upload file lên để làm ảnh đại diện.
 	Ở đây, chúng ta chọn một file có tên là shell.php (Chúng ta chọn file có đuôi là “php” vì trang web này được xây dựng bằng PHP và chạy trên máy chủ Apache2) để tải lên.
Nội dung file shell.php như sau:
 
Hình 3. 4 Nội dung file shell.php

Giải thích nội dung của file shell.php:
1.	system():
	Hàm system() trong PHP được sử dụng để chạy một lệnh hệ thống (command) trên máy chủ nơi mã PHP được thực thi.
	Kết quả (output) của lệnh này sẽ được gửi trực tiếp tới trình duyệt của người dùng.
2.	$_GET['cmd']:
	$_GET là một mảng chứa các tham số được truyền qua URL bằng phương pháp GET.
Ở đây, đoạn mã đang lấy giá trị của tham số cmd từ URL, sau đó truyền nó làm đối số cho hàm system().

 
Hình 3. 5 Các thông tin from đăng kí

Khi nhấn Register, có xuất hiện thông báo “Registration successful! You can login.” . Điều này chứng tỏ ta có thể đăng kí thành công tài khoản.
 
Hình 3. 6 Thông báo đăng kí thành công
Sau đó chúng ta nhấn OK để tiếp tục đăng nhập.
Sau khi đăng nhập với tài khoản đăng kí, ta được giao diện như sau:
 
Hình 3. 7 Trang web sau khi đăng nhập

Tiếp theo, ta mở inspect để xác định được đường dẫn của file ảnh được lưu. Kết quả ta được như sau:
 
Hình 3.8 Nội dung HTML ở client
Ở đây, ta thấy đuợc khi upload file lên server thì tên file chỉ thêm một dãy số ngẫu nhiên mà không có phát hiện ra các file có thể thực thi trực tiếp qua Apache2. Cho nên, ta có thể khai thác lỗi bảo mật Upload file để có thể thực thi câu lệnh  và điều khiển server từ xa.
Khi chọn vào đường dẫn file ta tải lên, kết quả như sau:
 
Hình 3.9 Truy cập file chứa mã khai thác
Ở đây, khi thêm một parameter cmd vào đường dẫn URL, ta có thê thực thi lệnh trên server. Khi ta thêm vào thanh URL : ?cmd=id, ta được kết quả như sau:
 
Hình 3.10 Kết quả khi thực thi lệnh id
Kết quả thu được ta thấy:
	Ngươi dùng hiện tại là www-data
	Người dùng www-data thuộc nhóm docker.
Điều này chứng tỏ trên server có cài đặt docker và người dùng “www-data” có thể thực thi lệnh docker mà không cần sudo.
Khi thay lệnh “id” bằng lệnh “mount” ta thu được kết quả sau:
 
Hình 3.11 Kết quả khi thực hiện lệnh mount
Ở đây, ta thây có một socket của docker được mount vào server. Từ những dữ liệu trên, ta có thể suy đoán được rằng: 
	Server được xây dựng trên Docker
	Có thể rằng /var/run/docker.sock được mount vào container chạy website với đường dẫn /custom/docker/docker/sock
Từ những thông tin trên, ta có thể suy đoán được rằng ta khai thác lỗi Docker-In-Docker qua việc sử dụng socket được mount vào để run một docker bên ngoài máy host. 
Để chứng thực cho việc này, ta thay lệnh “mount” bằng lệnh “ docker -H unix:///custom/docker/docker.sock images” để kiểm tra xem liệu rằng có thực hiện được hay không. Kết quả thu dược là:
 
Hình 3.12 Kết quả khi thực thi lệnh docker -H
Ở đây, ta thu được danh sách các images đang có trên socket này. Điều này chứng tỏ suy đoán của chúng ta đã đúng.
Ở đây ta tiếp tục run một image bất kì, ta chọn image hello-world, kết quả như sau:
 
Hình 3.13 Kết quả khi run image hello-world
Tiếp theo, ta sẽ tiếp tục run một images với port 8085. Đây là một images mà tôi đã tạo với việc hiển thị dòng chữ “Hacked Detected” và đã được đẩy lên DockerHub với tên là: hacked_images_docker. Dòng lên được sử dụng là: docker -H unix:///custom/docker/docker.sock run -p 8085:80 hacked_images_docker. Kết quả thu đươc:
 
Hình 3.14 Kết quả khi run một images tên ph4n10m1808/hacked_images_docker
Ở đây, ta có thể chạy được một images khác trên máy host. Hacker có thể lợi dụng việc này để có thể tải lên các images độc hại lên máy host và gây hại cho máy host chạy các Docker Container này.
Đây là đoạn cấu hình bị sai gây lỗi trên:
 
Hình 3.15 Cấu hình lỗi ở file docker-compose.yaml
 
Hình 3.16 Cấu hình lỗi ở file Dockerfile
Ở đây, ta thấy được người quản trị hệ thống đã mount socket tử máy host sang container, sau đó đưa người dùng thường vào group docker khiến cho bất kì người dùng nào cũng có thể thực thi được docker không qua root.
Từ đó, dẫn đến việc các hacker có thể xâm nhập vào và chạy nhiều container trên máy host. Điều này dẫn đến việc gây tiêu tốn nhiều tài nguyên trên máy host, thậm chí có thể gây sập máy host. Nếu máy host ở trên các dịch vụ cloud như AWS, Google Cloud, … , việc sử dụng nhiều tài nguyên sẽ gây ra sự thất thoát tài chính vô cùng lớn. Do đó, lập trình viên và những người triển khai hệ thống máy chủ chúng ta nên lưu ý việc này để tránh gây ra những tổn thất lớn.
Ngoài ra, việc bảo mật những ứng dụng chạy trên container ở môi trường sản xuất cũng nên được chú trọng. Ở đây, trang web chạy trên container này bị lỗ hổng Upload file. Lỗ hổng này do lập trình viên không kiểm tra các file được tải lên trên server, từ đó dẫn đến việc server dễ bị tấn công

3.2. Một số biện pháp phòng chống.
Để phòng chống các lỗ hổng bảo mật Misconfiguration trên Docker, cần áp dụng một loạt các biện pháp bảo mật và cấu hình phù hợp nhằm đảm bảo an toàn cho hệ thống.
1. Hạn chế quyền --privileged và các quyền truy cập khác
Tránh sử dụng quyền --privileged khi khởi chạy container, vì quyền này cấp quá nhiều đặc quyền và dễ dẫn đến container breakout.
Chỉ cấp các quyền cần thiết: Sử dụng tùy chọn --cap-drop để loại bỏ các quyền không cần thiết và chỉ giữ lại những quyền tối thiểu cho container hoạt động.
2. Bảo mật Docker Socket
Không mount Docker socket (/var/run/docker.sock) vào container, vì điều này cho phép container kiểm soát Docker daemon và thao tác với các container khác trên host.
Sử dụng Docker API với xác thực: Nếu cần truy cập từ xa vào Docker daemon, cấu hình bảo mật bằng TLS và xác thực người dùng để ngăn chặn truy cập trái phép vào API.
3. Giới hạn quyền trong container
Chạy container với non-root user: Sử dụng --user để chỉ định một người dùng không có quyền root trong container, nhằm giảm nguy cơ tấn công nếu container bị xâm nhập.
Áp dụng chính sách AppArmor hoặc SELinux: Sử dụng các chính sách này để giới hạn quyền truy cập của container vào các tài nguyên hệ thống của host.
4. Quản lý cấu hình mạng của Docker
Phân tách mạng cho từng container hoặc nhóm container: Sử dụng các mạng riêng biệt (bridge networks) để cô lập các container khác nhau, giúp ngăn ngừa tấn công lateral movement.
Áp dụng firewall rules: Cấu hình firewall để kiểm soát và giới hạn các cổng mở mà Docker daemon sử dụng, chỉ cho phép truy cập từ các nguồn tin cậy.
5. Kiểm soát Volume và quản lý dữ liệu
Không mount volume nhạy cảm từ host: Tránh mount các volume từ hệ thống file host vào container trừ khi cần thiết. Khi mount, nên giới hạn quyền chỉ đọc để ngăn ngừa ghi dữ liệu vào các file nhạy cảm.
Sử dụng Docker Secrets và Configs: Thay vì lưu trữ thông tin nhạy cảm trong volume hoặc biến môi trường, Docker Secrets và Configs giúp quản lý và bảo vệ dữ liệu nhạy cảm một cách an toàn.
6. Cập nhật hình ảnh Docker và kiểm tra tính bảo mật
Sử dụng các image chính thức và được cập nhật thường xuyên: Chỉ sử dụng các image từ các nguồn uy tín hoặc từ các registry chính thức để giảm thiểu nguy cơ từ các image chứa mã độc.
Kiểm tra các lỗ hổng trong Docker image: Sử dụng các công cụ như Clair, Anchore, hoặc Trivy để quét và phát hiện các lỗ hổng trong image trước khi triển khai.
7. Bảo mật Docker API Daemon
Không mở cổng Docker API trên mạng công cộng: Hạn chế Docker API daemon chỉ có thể truy cập qua localhost hoặc thiết lập các whitelist IP để kiểm soát truy cập.
Sử dụng mã hóa TLS cho Docker API: Cấu hình bảo mật TLS cho Docker daemon khi cần truy cập từ xa để đảm bảo dữ liệu được truyền tải an toàn.
8. Sử dụng chính sách bảo mật Seccomp và cấu hình giới hạn tài nguyên
Kích hoạt Seccomp profile: Docker cung cấp một Seccomp profile mặc định nhằm giới hạn các syscall có thể gọi từ container, giúp ngăn chặn một số cuộc tấn công nhắm vào kernel.
Giới hạn tài nguyên cho container: Sử dụng các tùy chọn --memory và --cpus để giới hạn tài nguyên, ngăn chặn container tiêu thụ quá nhiều tài nguyên và gây ảnh hưởng đến hệ thống.
9. Tự động hóa kiểm tra bảo mật và cấu hình
Sử dụng công cụ tự động kiểm tra cấu hình Docker: Các công cụ như Docker Bench for Security có thể tự động kiểm tra cấu hình của Docker và đưa ra các đề xuất cải thiện bảo mật.
Áp dụng Continuous Integration/Continuous Deployment (CI/CD) với kiểm tra bảo mật: Thiết lập quy trình CI/CD với các bước kiểm tra bảo mật và phát hiện lỗ hổng để đảm bảo mọi thay đổi mới được kiểm duyệt trước khi triển khai.
10. Giám sát và phát hiện xâm nhập
Sử dụng các công cụ giám sát: Kết hợp các công cụ giám sát như Sysdig, Falco, hoặc Auditd để theo dõi hoạt động của container và phát hiện các hành vi bất thường.
Xây dựng hệ thống cảnh báo: Thiết lập cảnh báo cho các sự kiện đáng ngờ như tạo container mới không xác định hoặc truy cập không hợp lệ vào Docker daemon.

