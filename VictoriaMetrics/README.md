VictoriaMetrics Cluster (Một giải pháp Monitor/Time Series Database có hiệu năng cực cao, thay thế hoàn hảo cho Prometheus khi cần Scale lớn).

Theo chuẩn DevOps production, cụm này được chi thành 3 thành phần core riêng biệt để dễ scale độc lập:
* vmstorage: Lưu dữ liệu (Stateful) mở port 8482, 8400 (Inbound từ vminsert), 8401 (Inbound từ vmselect)
* vminsert: Ghi dữ liệu vào (Stateless) mở port 8480 (Ingrestion)
* vmselect: Đọc/Truy vấn dữ liệu ra (Stateless) mở port 8481 (Query)

1. Kiến trúc tổng quan của cụm

<img width="938" height="432" alt="image" src="https://github.com/user-attachments/assets/fa33113f-8d61-4a80-910b-db3b3591e81a" />

Trước khi cấu hình, bạn cần nắm được luồng đi của dữ liệu trong cụm VictoriaMetrics Cluster:
* Luồng ghi (ingrestion): Prometheus/Agent -> Load Balancer -> vminsert -> vmstorage
* Luồng đọc (Query): Grafana/User -> Load Balancer -> vmselect -> vmstorage


2. Các bước chuẩn bị (all node)

Bước 2.1: Cấu hình file hosts

/etc/hosts
```
10.0.0.11 dc-prod-devops-vmstorage01
10.0.0.12 dc-prod-devops-vmstorage02
10.0.0.13 dc-prod-devops-vmstorage03
10.0.0.21 dc-prod-devops-vminsert01
10.0.0.22 dc-prod-devops-vminsert02
10.0.0.23 dc-prod-devops-vminsert03
10.0.0.31 dc-prod-devops-vmselect01
10.0.0.32 dc-prod-devops-vmselect02
```

Bước 2.2: Tạo User hệ thống (không có quyền login, ko có home directory)
```
groupadd -r victoriametrics
useradd -r -g victoriametrics -s /sbin/nologin -d /nonexistent victoriametrics
```

Bước 2.3: Tải và cài đặt Binany

Tải package cluster (dạng victoria-metrics-vX.XX.X-cluster.tar.gz), giải nén và di chuyển file binary tương ứng vào thư mục hệ thống:
* Trên 3 node Storage: Move vmstorage-prod vào /usr/local/bin/
* Trên 3 node Insert: Move vminsert-prod vào /usr/local/bin/
* Trên 2 node Select: Move vmselect-prod vào /usr/local/bin/

Đặt binary tại /usr/local/bin/ và cấp quyền chạy: chmod +x /usr/local/bin/vm*-prod

3. Cấu hình chi tiết từng thành phần

Chúng ta sẽ dùng Systemd để quản lý các service này cho chuẩn production

Bước 3.1: Dựng các node vmstorage (01, 02, 03)

Tạo thư mục chứa data và cấu hình

```
mkdir -p /opt/victoriametrics-vmstorage /var/lib/vmstorage
chown -R victoriametrics:victoriametrics /opt/victoriametrics-vmstorage /var/lib/vmstorage
```

Tạo file cấu hình môi trường /opt/victoriametrics-vmstorage/vmstorage.conf

```
# VictoriaMetrics vmstorage environment configuration
retentionPeriod=1m
storageDataPath=/var/lib/vmstorage
httpListenAddr=:8482
vminsertAddr=:8400
vmselectAddr=:8401
loggerFormat=json
```

Tạo file dịch vụ systemd ```/etc/systemd/system/vmstorage.service```
```
[Unit]
Description=VictoriaMetrics vmstorage service
After=network.target

[Service]
Type=simple
User=victoriametrics
Group=victoriametrics
Restart=always
EnvironmentFile=/opt/victoriametrics-vmstorage/vmstorage.conf
ExecStart=/usr/local/bin/vmstorage-prod -envflag.enable

# Security Hardening Isolation
PrivateTmp=yes
ProtectHome=yes
NoNewPrivileges=yes
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

Ghi chú:
* User=victoriametrics / Group=victoriametrics: Chạy service bằng user thường có quyền hạn hạn chế, không chạy bằng root. Nếu hacker chiếm được quyền điều khiển service, họ cũng không phá được hệ điều hành.
* PrivateTmp=yes: Tạo một thư mục /tmp độc lập cho service, không dùng chung /tmp của hệ thống để tránh rò rỉ dữ liệu hoặc bị chèn file độc hại.
* ProtectHome=yes: Thư mục cá nhân của các user khác (/home, /root) sẽ trở thành "bất khả xâm phạm" (hoàn toàn trống rỗng) đối với service này.
* ProtectSystem=full: Biến các thư mục hệ thống như /usr, /boot, /etc thành chế độ Chỉ đọc (Read-only) đối với service. Service chỉ được quyền ghi vào đúng thư mục data được chỉ định.
* NoNewPrivileges=yes: Ngăn chặn tiến trình (process) tự động leo thang đặc quyền (Privilege Escalation) thành root.

Kích hoạt và chạy service:
```
systemctl daemon-reload && systemctl enable --now vmstorage
```

Bước 3.2: Dựng các node vminsert (01, 02, 03)

Tạo thư mục cấu hình
```
mkdir -p /opt/victoriametrics-vminsert
chown -R victoriametrics:victoriametrics /opt/victoriametrics-vminsert
```

File cấu hình môi trường: /opt/victoriametrics-vminsert/vminsert.conf
```
# VictoriaMetrics vminsert environment configuration
httpListenAddr=:8480
storageNode=dc-prod-devops-vmstorage01:8400,dc-prod-devops-vmstorage02:8400,dc-prod-devops-vmstorage03:8400
replicationFactor=2
loggerFormat=json
```
(Sử dụng tham số -replicationFactor=2 để nhân bản chéo dữ liệu, đảm bảo chết 1 node storage cụm vẫn sống nguyên vẹn).

File dịch vụ Systemd /etc/systemd/system/vminsert.service
```
[Unit]
Description=VictoriaMetrics vminsert service
After=network.target

[Service]
Type=simple
User=victoriametrics
Group=victoriametrics
Restart=always
EnvironmentFile=/opt/victoriametrics-vminsert/vminsert.conf
ExecStart=/usr/local/bin/vminsert-prod -envflag.enable

# Security Hardening Isolation
PrivateTmp=yes
ProtectHome=yes
NoNewPrivileges=yes
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

Kích hoạt và chạy service:
```
systemctl daemon-reload && systemctl enable --now vminsert
```
