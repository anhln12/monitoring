# Install Ubuntu 20.04
```
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu20.04_all.deb
sudo dpkg -i zabbix-release_6.0-4+ubuntu20.04_all.deb
sudo apt update
sudo apt install zabbix-agent -y
sudo nano /etc/zabbix/zabbix_agentd.conf

Cập nhật các tham số sau:
- Server: IP của Zabbix Server.
- ServerActive: IP của Zabbix Server (cho các item dạng active check).
- Hostname: Tên định danh của host này trên Zabbix Web UI.

sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
sudo ufw allow 10050/tcp
```


# Uninstall Zabbix Agent * Install Zabbix Agent 2
Bước 1: Dừng và gỡ bỏ Zabbix Agent cũ
```
sudo systemctl stop zabbix-agent
sudo systemctl disable zabbix-agent
sudo apt-get purge zabbix-agent -y
sudo apt-get autoremove -y
```
Bước 2: Cập nhật Zabbix Repository (Nếu cần)
```
# Ví dụ cấu hình cho Zabbix Repo 6.0 LTS trên Ubuntu 22.04 (Jammy)
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb
sudo apt update
```

Ubuntu 24.04 (Noble)
```
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
```


Bước 3: Cài đặt Zabbix Agent 2
```
sudo apt install zabbix-agent2 zabbix-agent2-plugin-* -y
```

Bước 4: Khôi phục cấu hình (Migration)
```
vi /etc/zabbix/zabbix_agent2.conf
```
Tìm và sửa đổi các tham số quen thuộc sau:
- Server: Điền IP của Zabbix Server hoặc Zabbix Proxy của bạn.
- ServerActive: Điền IP của Zabbix Server/Proxy phục vụ cho các item Active.
- Hostname: Điền chính xác tên Hostname của Server này (phải khớp 100% với tên Host name đặt trên Zabbix Web UI, ví dụ: Parrot-Server01).



