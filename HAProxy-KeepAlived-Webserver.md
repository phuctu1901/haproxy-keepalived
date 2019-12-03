# Yêu cầu:
Triển khai trên Virtual Box gồm:
- 2 load-balancer (gọi tắt là lb: lb1, lb2)
- 2 webserver (gọi tắt là ws: ws1, ws2)

# A. Chuẩn bị:
## 1. Đặt ip tĩnh cho các máy:
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yml
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
     dhcp4: no
     addresses: [192.168.8.110/24]
     gateway4: 192.168.8.1
     nameservers:
       addresses: [8.8.8.8,8.8.4.4]
```
```bash
sudo netplan apply
sudo reboot
```

- lb1: 192.168.8.110
- lb2: 192.168.8.111
- ws1: 192.168.8.120
- ws2: 192.168.8.121


## 2. Cài đặt openssh-server (tùy chọn):
```bash
sudo apt-get install openssh-server
```

## 3. Đổi tên máy ảo ubuntu sau khi clone:
  Chúng ta chỉ tạo 1 máy ảo để làm load-balancer và 1 máy ảo làm webserver sao đó clone ra thành 2 lb và 2 ws. Sau khi clone thì đổi ip tĩnh của máy đó.

  Đồng thời để dễ phân biệt trong quá trình thao tác thì đổi luôn hostname của Ubuntu bằng cách sau:
https://linuxize.com/post/how-to-change-hostname-on-ubuntu-18-04/

```bash
sudo hostnamectl set-hostname tên-bạn-muốn đổi
sudo nano /etc/cloud/cloud.cfg
```
Tìm dòng:
 `preserve_hostname: false` và sửa thành `preserve_hostname: true`
Sau đó kiểm tra lại và khởi động lại máy
```bash
hostnamectl
sudo reboot
```

# B. Thực hiện:

## 1. Cài đặt máy chủ web (webservers)
### 1.1. Cài đặt Apache2 để làm webserver
```bash
sudo apt-get -y install apache2
sudo service apache2 start
sudo service apache2 status
```
Đối với ai làm việc với php thì có thể cài đặt thêm PHP
```bash
sudo apt-get -y install php
sudo service apache2 restart
```

### 1.2. Tiến hành clone máy ảo và đổi ip tĩnh cũng như hostname như hướng dẫn ở trên

### 1.3. Thay đổi nội dung trang `index`:
Thay đổi nội dung trang index ở địa chỉ `/var/www/html/index.html`

## 2. Cài đặt cân bằng tải (load-balancers):
### 2.1. Cài đặt KeepAlived để tạo IP ảo phục vụ cho việc sử dụng nhiều cân bằng tải:
**Đối với lb1: 192.168.8.110**
  
Cài đặt:
```bash
sudo apt-get update
sudo apt-get install -y keepalived
```

Cấu hình:
```bash
nano /etc/keepalived/keepalived.conf
```

```bash
vrrp_script chk_haproxy {           # Requires keepalived-1.1.13
        script "killall -0 haproxy"     # cheaper than pidof
        interval 2                      # check every 2 seconds
        weight 2                        # add 2 points of prio if OK
}

vrrp_instance VI_1 {
        interface enp0s3
        state MASTER
        virtual_router_id 51
        priority 101   # 101 on master, 100 on backup
        virtual_ipaddress {
            192.168.8.123 # địa chỉ IP ảo dùng để truy cập
        }
        track_script {
            chk_haproxy
        }
}
```

```bash
sudo echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
sudo sysctl -p
sudo apt install -y psmisc
sudo /etc/init.d/keepalived start
sudo service keepalived status
```

```bash
# Kiểm tra IP ảo đã ổn chưa:
ip addr sh enp0s3
```

### 2.2. Cài đặt HAProxy:

```bash
sudo apt install -y haproxy
sudo nano /etc/haproxy/haproxy.cfg
```

```config
global
        daemon
        maxconn 256

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms
    frontend http-in
        bind *:80
        default_backend app
    frontend stats
        bind *:8084
        stats enable
        stats uri /stats
        stats refresh 10s
    backend static
        balance roundrobin
        server static 192.168.8.123:80
    backend app
        balance roundrobin
        server maychu-001 192.168.8.120 check
        server maychu-002 192.168.8.121 check
```
```nano
sudo service haproxy start
sudo service haproxy status
```

### 2.3. Kiểm tra tình trạng hoạt động:
Truy cập: [Trang web thông qua server ảo này](192.168.8.123) và tải lại trang (refresh) xem nội dung có thay đổi không?

Truy cập: [Trang trực tiếp thông qua server1](192.168.8.120)

Truy cập: [Trang trực tiếp thông qua server2](192.168.8.121)

Truy cập: [Trang trại thái](192.168.8.123:8084/stats) để kiểm tra trạng thái server

### 2.4. Triển khai server cân bằng tải dự phòng (lb2):
Clone máy ảo lb1.

Đổi hostname

Đổi địa chỉ ip thành 192.168.8.111.

Cấu hình lại KeepAlived của lb2.
```bash
nano /etc/keepalived/keepalived.conf
```
```bash
vrrp_script chk_haproxy {           # Requires keepalived-1.1.13
        script "killall -0 haproxy"     # cheaper than pidof
        interval 2                      # check every 2 seconds
        weight 2                        # add 2 points of prio if OK
}

vrrp_instance VI_1 {
        interface enp0s3
        state MASTER
        virtual_router_id 51
        priority 100   # 101 on master, 100 on backup
        virtual_ipaddress {
            192.168.8.123 # địa chỉ IP ảo dùng để truy cập
        }
        track_script {
            chk_haproxy
        }
}
```
# C. Kết luận:
Như vậy mô hình của bài toán này sẽ như sau:

![alt text][partent]

[partent]: ./images/mô-hình.png "Mô hình bài toán"

Trang trạng thái:

![alt text][stat-page]

[stat-page]: ./images/trạng-thái.PNG " "