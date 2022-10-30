---
title: "Tailscale for your friendly work VPN"
date: 2022-10-22T07:33:29+00:00
lastmod: 2022-10-23T07:33:29+00:00
description: "This article shows how to use Tailscale and what can we achieve in Tailscale platform."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Tailscale", "VPN", "Work Tool", "Bastion Host", "Cloud Security", "Subnet Router"]
categories: ["VPN"]


---

Ưu tiên quan trọng nhất trong bảo mật đám mây là private access và admin control để quản lý người dùng và giới hạn dịch vụ họ có thể truy cập. Tuy nhiên, việc kiểm soát quá nhiều quyền truy cập và quản trị gây cản trở và khó khăn cho việc debug của team members. Đặc biệt là càng nhiều services thì yêu cầu bảo trì nhiều hơn từ cả maintainer và user, nguồn lực để giữ mọi thứ theo tiêu chuẩn bảo mật đòi hỏi nhiều về chi phí con người lẫn tiền bạc. Tailscale là dịch vụ VPN mới cung cấp giải pháp cho những vấn đề trên với chi phí rất cạnh tranh.

<!--more-->

## My goal

Hiện tại, đa số môi trường làm việc của công ty yêu cầu truy cập máy chủ và database service từ Bastion Host và VPN. Việc thiết lập ban đầu vốn rất đơn giản, nhưng khi đội ngũ và service của team phát triển, độ phức tạp của việc thiết lập sẽ tăng lên rất nhiều lần. Onboard người mới không bao giờ dễ dàng và mặc dù mình đã thiết lập [Dev microVM](https://haicheviet.com/cluster-vm-with-firecracker/) với tất cả các environment và package cần thiết, cấu hình mạng vẫn là những bước khó khăn để junior dev có thể debug local. Các vấn đề của quy trình hiện tại có thể được liệt kê như sau:

- Một số [HPC servers](https://en.wikipedia.org/wiki/High-performance_computing) được thiết lập ở mạng nội bộ công ty, team members chỉ có thể truy server từ mạng nội bộ và hiện tại chưa support VPN nội bộ. Email request cho admin network rất tốn thời gian và cồng kềnh vì có cả đống thủ tục mà mình phải follow để yêu cầu VPN. Dù vậy mình cũng đã cố gắng cung cấp các thông tin cần thiết nhưng bị từ chối request vì lý do teamsize quá nhỏ nên không cần thiết :disappointed_relieved:.
- Mình đã thử Cloudflare Tunnel và Ngrok nhưng để setup được chuẩn chỉnh rất tốn công, đặc biệt mỗi service đều cần domain host riêng để public access nhưng cái mình muốn chỉ cần truy cập qua email thôi là đủ. Như vậy thì tốn công và khó sử dụng quá.
- Hiện tại cloud platform của team đang được đặt chính ở private subnet để giảm thiểu cost network lẫn đảm bảo bảo mật. Để team members có thể access vào cloud database thì cần Bastion Host và VPN.
- Bastion host xài một hai services thì ok nhưng càng nhiều service thì config ssh rất phức tạp và khó transfer cho từng member hiểu được họ đang thiếu foward service nào.
- VPN xài ổn nhưng chỉ có thể xài một VPN ở một thời điểm duy nhất. Như vậy để switch môi trường debug cần phải tắt bật VPN nhưng như vậy khi vào đang xài VPN staging thì họ không thể vào mạng nội bộ hoặc production được. Rất tốn thời gian và khó khăn nếu member cần access nhiều mạng một lúc.

Mình đã cằn nhằn về cách cấu hình mạng hiện tại rất lâu và việc chứng kiến các junior member khi debug local gặp nhiều khó khăn không phải là một cảnh dễ chịu. Tailscale là giải pháp mới mình tìm được để giải quyết tất cả các vấn đề đã liệt kê. Trải nghiệm của team khi sử dụng rất là tốt và mọi thứ đều có thể được truy cập và cấu hình dễ dàng.

## Tailscale hands on

Tạo tài khoản Tailscale rất dễ, mọi người có thể vào [trang này](https://login.tailscale.com/start) để đăng ký. Tài khoản đăng ký hiện tại chỉ support đang ký qua email của những SSO identity providers lớn (như Gooogle hoặc Microsoft). Nếu bạn cần đăng ký email của các provider khác thì tham khảo thêm ở [đây](https://tailscale.com/kb/1013/sso-providers/) để thiết lập.

![Login page](tailscale-login.webp "Login page")

Khi tạo account thành công và login, bạn sẽ thấy được admin page với những device bạn đã register.

![Admin page](admin-page.webp "Admin page")

Hình trên là những devices mà mình đang sử dụng, mỗi device được cấp riêng IPv4 address mà có thể access ngay được khi đăng nhập vào Tailscale, không cần thiết lập domain hay public access gì.

## Tailscale device register

Tailscale giúp bạn kết nối các thiết bị của mình với nhau. Để đăng ký thiết bị lên Tailscale cluster, tải Tailscale ở máy client của bạn và device bạn muốn register. Tailscale hiện tại support rất nhiều hdh như Linux, Windows, macOS, Raspberry Pi, Android, Synology, v.v. [Tải xuống Tailscale](https://tailscale.com/download) và đăng nhập trên thiết bị dựa trên auto-script.

Còn đây là cách mình thiết lập ở máy Fedora:

### Step 1: Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Step 2: Connect my device to Tailscale

Đăng ký từ browser

```bash
$ sudo tailscale up

To authenticate, visit:

      https://login.tailscale.com/a/abcde
```

Đăng ký từ key ở [your setting keys](https://login.tailscale.com/admin/settings/keys)

```bash
sudo tailscale up --authkey tskey-abcdef1432341818
```

### Step 3: Verify your connection

Kiểm tra xem có thể ping tới thiết bị mới register từ máy client được không. IP của thiết bị mới có thể tìm ở [admin console](https://login.tailscale.com/admin/machines), hoặc bạn có thể run câu lệnh sau ở thiết bị mới để show IP.

```bash
$ tailscale ip -4
100.83.201.24 # Your IPv4 in tailscale
```

Ping thử IPv4 của thiết bị mới từ máy của mình

```bash
$ ping 100.83.201.24

Pinging 100.83.201.24 with 32 bytes of data:
Reply from 100.83.201.24: bytes=32 time=77ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
```

Vậy là mình đã hoàn thành đăng ký thiết bị mới lên Tailscale cluster, từ giờ member nào trong cluster cũng có thể access ssh hoặc port tới thiết bị mới từ IPv4 mà mình đã đăng ký ở Tailscale Admin.

Thêm nhiều thiết bị vào cluster bằng cách tiếp tục từ step 2 hoặc [mời nhưng member truy cập vào network của bạn](https://tailscale.com/kb/1064/invite-team-members/). Và hơn thế nữa, Tailscale support chia sẻ files từ các devices và user với nhau khi bạn enable [TailDrop](https://tailscale.com/kb/1106/taildrop/#enabling-taildrop-for-your-network)

{{< video src="https://tailscale.com/kb/1106/taildrop/taildrop.mp4" loop=true autoplay=true >}}

## Tailscale as a network layer

Taiscale hổ trợ cả `subnet router` (trước đây gọi là relay node hoặc relaynode) để access tới thiết bị mạng private từ TailScale. **Bộ định tuyến hoạt động như một cổng chuyển tiếp**, chuyển tiếp lưu lượng truy cập từ mạng Tailscale của bạn vào private network vật lý. Bộ định tuyến có các tính năng như chính sách kiểm soát truy cập, giúp dễ dàng di chuyển một mạng lớn sang Tailscale mà không cần cài đặt ứng dụng trên mọi thiết bị.

![Subnet router](subnets.webp "Subnet router")

Việc thiết lập subnet router tương đối dễ dàng và bạn có thể đọc [hướng dẫn này](https://tailscale.com/kb/1019/subnets/#setting-up-a-subnet-router) để biết thêm chi tiết. Dưới đây là quy trình chi tiết mà tôi đã thiết lập trên đám mây AWS.

### Step 1: Create an EC2 instance router

Đầu tiên, tạo một instance EC2 chạy Amazon Linux trên x86 hoặc ARM. Tailscale hỗ trợ cài đặt cho cả hai kiến trúc và bạn nên xài các phiên bản AWS ARM vì rất hiệu quả về chi phí. Khi đặt security policy, `cho phép cổng UDP 41641` xâm nhập từ bất kỳ nguồn nào. Điều này sẽ cho phép kết nối trực tiếp và giảm thiểu độ trễ.

![Security Group](security-group.webp "Security Group")

### Step 2: Configure tailscale subnet router

Sau đó, ssh vào instance và làm theo các bước [cài đặt Tailscale trên Amazon Linux 2](https://tailscale.com/kb/1052/install-amazon-linux-2/) và thiết lập subnet router. Khi chạy `tailscale up`, hãy chuyển dải mạng VPC của instance đến `--advertise-route`.

![VPC subnet address](subnet-address.webp "VPC subnet address")

Như hình trên ta có thể thấy dải mạng của VPC là 10.0.0.0/16:

```bash
tailscale up --advertise-routes=10.0.0.0/16 --accept-dns=false
```

{{< admonition tip >}}

Đối với các EC2 instances, tốt nhất là để Amazon xử lý cấu hình DNS, không để Tailscale ghi đè nó, vì vậy mình đã thêm --accept-dns=false.

{{< /admonition >}}

### Step 3: Enable subnet router in admin

Tiếp theo ta sẽ kiểm tra xem subnet router đã được config thành công chưa.

```bash
$ tailscale ip -4
100.83.201.24
```

Các step tiếp theo không cần thiết nếu bạn đã enable lựa chọn *autoApprovers* ở admin.

Truy cập vào trang [admin console](https://login.tailscale.com/admin/machines), và tìm kiếm thiết bị mới mà bạn đã đăng ký subnet routes. Bạn có thể tìm kiếm ở Subnets badge trên các thiết bị được liệt kê. Sử dụng `...` icon ở cuối hàng, bấm vào `Edit route settings`. Từ đây sẽ có popup về Edit route settings panel như hình dưới.

![Edit route settings](subnet-route-setting.webp "Edit route settings")

Bạn đồng ý các route được đăng ký bằng cách kéo thanh bar từ trái sang phải.

{{< admonition tip >}}

Tốt nhất bạn nên disable key expiry ở con server subnet route để tránh việc phải reauthenticate lại nhiều lần.

{{< / admonition >}}

### Step 4: Use your subnet routes from other machines

Bạn có thể traceroute và SSH private instances (10.0.26.12) từ subnet routes bạn mới đăng ký (100.83.201.24).

```bash

$ traceroute 10.0.26.12
traceroute to 10.0.26.12 (10.0.26.12), 64 hops max, 52 byte packets
 1  100.83.201.24 (100.83.201.24)  27.135 ms  17.935 ms  18.342 ms
 2  10.0.26.12 (10.0.26.12)  19.396 ms  18.852 ms  18.364 ms

$ ssh /path/to/private_key ec2-user@10.0.26.12
```

Nếu bạn cần truy cập các dịch vụ khác, chẳng hạn như các AWS RDS database được tạo mà không có khả năng truy cập công khai. Tên miền của private service này sẽ được access qua các advertise route đã được thiết lập qua ở trên.

Bạn có thể access RDB database từ mạng Tailscale y hệt cách các instance private truy cập.

```bash
$ dig +short database.xxx.ap-southeast-1.rds.amazonaws.com
10.0.3.176

$ mysqlsh --uri=admin@database.xxx.ap-southeast-1.rds.amazonaws.com:3306
...
 MySQL  database-1.c75dvgjlkohb.ap-southeast-1.rds.amazonaws.com:3306 ssl  JS >
 
```

{{< admonition tip >}}

Các client như Windows, macOS, Android, iOS, v.v. đều chấp nhận các advertised routes theo mặc định, nhưng các client Linux cần thiết lập thêm  `tailscale up --accept-route = true` để sử dụng các subnet router trong AWS.

{{< /admonition >}}

## Some afterthought

- Sử dụng Tailscale thực sự đã đến mức mình không thể quay lại sử dụng cách truyền thống nữa. Mọi dịch vụ đều có thể được truy cập bởi tên miền riêng của nó, không còn bị xáo trộn môi trường vì chuyển mạng hoặc lỗi do cổng đang được sử dụng.
- Tailscale là một dịch vụ trả phí và users phải phụ thuộc vào Tailscale để quản lý máy chủ. Rất may, Taiscale cũng cung cấp giải pháp cho open-source hosting [Headscale](<https://github.com/juanfont/headscale>) và bạn có thể tự cấu hình quản trị tất cả máy chủ của mình.
- Và cuối cùng bạn có thể sử dụng [Terraform](<https://registry.terraform.io/modules/hardfinhq/tailscale-subnet-router/aws/latest?tab=resources>) để quản lý Tailscale và tự động hóa tất cả các quy trình để thiết lập subnet router mà mình đã liệt kê ở trên.
