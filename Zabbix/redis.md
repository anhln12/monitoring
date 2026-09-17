# Giám sát Redis bằng Zabbix Agent2: Hướng dẫn chi tiết từ A đến Z

## Giới thiệu

Redis là một trong những thành phần hạ tầng quan trọng nhất trong các hệ thống backend hiện đại — từ cache, session store, message queue cho đến dữ liệu real-time. Khi Redis gặp sự cố (hết bộ nhớ, mất kết nối, replication lag, hay đơn giản là chậm bất thường), hậu quả thường lan rất nhanh sang các dịch vụ phụ thuộc phía trên.

Từ Zabbix Agent2 (viết bằng Go), Zabbix đã tích hợp sẵn **plugin Redis native**, cho phép giám sát Redis mà không cần cài thêm script Python/Perl như thời Zabbix Agent 1. Tuy nhiên, trên thực tế triển khai — đặc biệt là trong môi trường có nhiều host với **nhiều phiên bản Zabbix Agent2 khác nhau** — quá trình cấu hình lại phát sinh khá nhiều lỗi khó lường mà tài liệu chính thức không nói rõ. Bài viết này tổng hợp toàn bộ quy trình chuẩn, kèm theo các lỗi thực tế thường gặp và cách xử lý dứt điểm.

---

## 1. Kiến trúc tổng quan

Việc giám sát Redis qua Zabbix Agent2 gồm 3 lớp:

1. **Redis server** — cần biết địa chỉ, port, và có yêu cầu xác thực (`requirepass` hoặc ACL) hay không.
2. **Zabbix Agent2** (cài trên chính host Redis hoặc host có thể reach tới Redis) — chứa plugin Redis, chịu trách nhiệm kết nối và thu thập dữ liệu.
3. **Zabbix Server/Frontend** — poll dữ liệu từ Agent2 qua các item key, hiển thị qua template có sẵn.

Điểm mấu chốt gây nhầm lẫn nhiều nhất: **cú pháp cấu hình plugin Redis khác nhau giữa các phiên bản Zabbix Agent2**. Đây là nguyên nhân của phần lớn lỗi sẽ trình bày ở Mục 4.

---

## 2. Cài đặt Zabbix Agent2

```bash
# Debian/Ubuntu
sudo apt install zabbix-agent2

# RHEL/CentOS/Oracle Linux
sudo yum install zabbix-agent2
```

Kiểm tra plugin Redis đã có sẵn trong bản build hay chưa:

```bash
zabbix_agent2 -p | grep -i redis
```

Kiểm tra phiên bản đang cài — **bước này cực kỳ quan trọng**, vì nó quyết định cú pháp cấu hình ở bước sau:

```bash
zabbix_agent2 -V
```

---

## 3. Cấu hình plugin Redis — đúng theo từng phiên bản Agent2

Khuyến nghị tạo file cấu hình riêng cho plugin thay vì sửa trực tiếp file chính, để dễ quản lý và tách biệt theo host:

```bash
sudo nano /etc/zabbix/zabbix_agent2.d/plugins.d/redis.conf
```

Đảm bảo file chính `/etc/zabbix/zabbix_agent2.conf` có dòng include:

```
Include=/etc/zabbix/zabbix_agent2.d/plugins.d/*.conf
```

### 3.1. Zabbix Agent2 phiên bản cũ (≤ 6.2.x) — chỉ hỗ trợ `Sessions`

Trên các bản như **6.2.9**, plugin Redis **chưa có khái niệm `Default`** — chỉ hỗ trợ khai báo theo tên session (`Plugins.Redis.Sessions.<TênSession>.*`). Nếu bạn dùng cú pháp `Plugins.Redis.Default.*` trên bản này, Agent2 sẽ báo lỗi và không khởi động được.

Cấu hình đúng:

```ini
Plugins.Redis.Sessions.RedisServer.Uri=tcp://localhost:6379
Plugins.Redis.Sessions.RedisServer.Password=your_redis_password
```

Test bằng key có kèm tên session:

```bash
zabbix_agent2 -t "redis.ping[RedisServer]"
zabbix_agent2 -t "redis.info[RedisServer]"
```

### 3.2. Zabbix Agent2 phiên bản mới hơn (6.4.x trở lên) — hỗ trợ `Default`

```ini
Plugins.Redis.Default.Uri=tcp://localhost:6379
Plugins.Redis.Default.Password=your_redis_password
```

Test bằng key trần, không cần chỉ định session:

```bash
zabbix_agent2 -t redis.ping
zabbix_agent2 -t redis.info
```

> **Lưu ý:** Ngay trong nội bộ dòng 6.4.x, tham số `Plugins.Redis.Default.User` (dùng cho Redis ACL user, Redis 6+) **không có ở mọi bản build** — kể cả bản 6.4.14. Nếu bạn gặp lỗi `invalid parameter Plugins.Redis.Default.User: unknown parameter`, hãy bỏ hẳn dòng `User` và chỉ dùng `Uri` + `Password`. Nếu bắt buộc cần xác thực bằng username ACL, truyền trực tiếp trong item key theo cú pháp `redis.ping[uri,user,password]` thay vì khai báo trong file cấu hình.

### 3.3. Bảng tóm tắt tương thích

| Zabbix Agent2 version | `Plugins.Redis.Default.*` | `Plugins.Redis.Sessions.*` | `Default.User` (ACL) |
|---|---|---|---|
| ≤ 6.2.x | ❌ Không hỗ trợ | ✅ Có | ❌ |
| 6.4.x | ✅ Có (Uri, Password) | ✅ Có | ⚠️ Tùy build, nhiều bản chưa có |
| 7.0+ | ✅ Có | ✅ Có | ✅ (từ các bản mới) |

Vì vậy, **luôn chạy `zabbix_agent2 -V` trước khi cấu hình**, đừng copy nguyên cấu hình từ tài liệu chính thức (thường viết cho bản mới nhất) sang host đang chạy bản cũ.

---

## 4. Các lỗi thực tế thường gặp và cách xử lý

### Lỗi 1: `flag provided but not defined: -s`

```
zabbix_agent2 -s 127.0.0.1 -k redis.ping
```

**Nguyên nhân:** Nhầm cú pháp giữa `zabbix_agent2` và `zabbix_get`. Flag `-s` (chỉ định host/IP) chỉ tồn tại ở `zabbix_get`, không có ở `zabbix_agent2`.

**Cách đúng:**

```bash
# Test trực tiếp bằng agent2 (không cần daemon đang chạy)
zabbix_agent2 -t redis.ping

# Hoặc dùng zabbix_get để test qua network (cần cài gói zabbix-get riêng)
zabbix_get -s 127.0.0.1 -k redis.ping
```

### Lỗi 2: `invalid parameter Plugins.Redis.Default.User: unknown parameter`

**Nguyên nhân:** Bản Agent2 đang chạy chưa hỗ trợ tham số `User` cho Redis plugin (phổ biến ở nhiều bản 6.x, kể cả một số bản 6.4).

**Cách xử lý:** Xóa dòng `Plugins.Redis.Default.User` khỏi file cấu hình. Nếu cần xác thực ACL username, truyền trực tiếp qua item key:

```bash
zabbix_agent2 -t "redis.ping[tcp://localhost:6379,acl_username,acl_password]"
```

### Lỗi 3: `invalid parameter Plugins.Redis.Default: unknown parameter`

**Nguyên nhân:** Khác lỗi 2 ở chỗ đây là tham số **cụt** (`Plugins.Redis.Default` không có hậu tố `.Uri`/`.Password`) — thường do:
- Bản Agent2 quá cũ (≤ 6.2.x) hoàn toàn chưa biết đến khái niệm `Default`, nên parser hiểu nhầm toàn bộ nhánh `Default.*` là một tham số không xác định.
- Hoặc file cấu hình bị lỗi cú pháp (dòng bị cắt, thiếu dấu `=`, ký tự ẩn do copy-paste từ nguồn khác).

**Cách xử lý:** Kiểm tra version bằng `zabbix_agent2 -V`. Nếu là bản ≤ 6.2.x, chuyển sang cú pháp `Sessions.<TênSession>.*` như Mục 3.1. Nếu là bản mới nhưng vẫn lỗi, kiểm tra ký tự ẩn trong file bằng:

```bash
cat -A /etc/zabbix/zabbix_agent2.d/plugins.d/redis.conf
```

### Lỗi 4: `Connection failed: NOAUTH Authentication required`

**Nguyên nhân:** Agent2 kết nối tới Redis **không kèm password** — do file cấu hình plugin thiếu dòng `Password`, dùng sai tên session, hoặc test bằng key trần (`redis.info`) trong khi cấu hình lại đặt theo tên session (`Sessions.RedisServer.*`) thay vì `Default.*`.

**Cách xử lý:**
1. Xác nhận đúng password bằng `redis-cli`:
   ```bash
   redis-cli -h 127.0.0.1 -p 6379 -a 'your_password' ping
   ```
2. Nếu dùng session, nhớ truyền tên session vào key khi test:
   ```bash
   zabbix_agent2 -t "redis.info[RedisServer]"
   ```

### Mẹo debug chung

Khi gặp bất kỳ lỗi parse config nào, luôn thực hiện song song 2 bước:
1. `zabbix_agent2 -V` — xác định chính xác version đang chạy trên host đó (đừng giả định tất cả host đều cùng version).
2. `cat -n <file_config>` — xem rõ số dòng, đối chiếu với thông báo lỗi `at line X` để khoanh vùng chính xác.

---

## 5. Cấu hình phía Zabbix Server / Frontend

### Bước 1: Import và gán template

Zabbix cung cấp sẵn template **"Redis by Zabbix agent 2"**. Vào **Data collection → Hosts** → chọn host → tab **Templates** → **Add** → chọn template này → **Update**.

### Bước 2: Cấu hình Macro theo đúng kiểu cấu hình plugin trên từng host

Đây là phần dễ nhầm nhất khi hạ tầng có nhiều version Agent2 khác nhau. Item mặc định trong template gọi theo dạng:

```
redis.ping[{$REDIS.CONN.URI},{$REDIS.PASSWORD},{$REDIS.USERNAME}]
```

**Trường hợp A — Host dùng `Plugins.Redis.Default.*` (Agent2 6.4+):**

| Macro | Giá trị |
|---|---|
| `{$REDIS.CONN.URI}` | `tcp://localhost:6379` |
| `{$REDIS.PASSWORD}` | để trống hoặc điền password tường minh |
| `{$REDIS.USERNAME}` | để trống nếu không dùng ACL |

**Trường hợp B — Host dùng `Plugins.Redis.Sessions.*` (Agent2 ≤ 6.2.x):**

| Macro | Giá trị |
|---|---|
| `{$REDIS.CONN.URI}` | **tên session** đã khai báo (ví dụ `RedisServer`), không phải URI thật |
| `{$REDIS.PASSWORD}` | để trống — agent tự lấy từ session đã cấu hình |
| `{$REDIS.USERNAME}` | để trống |

Macro nên được set **ở cấp Host** (không sửa ở Template) khi các host có cấu hình khác nhau, để tránh ảnh hưởng dây chuyền sang các host khác dùng chung template.

### Bước 3: Kiểm tra dữ liệu

Vào **Monitoring → Latest data**, lọc theo host. Nếu thấy các item như `Redis: Uptime`, `Redis: Ping`, `Redis: Number of clients connected` có giá trị (không phải `Not supported`) là đã cấu hình thành công.

Nếu item vẫn lỗi, dùng **Execute now** trực tiếp trên item đó (trong Configuration → Items) để xem thông báo lỗi chi tiết trả về, thường sẽ chỉ rõ nguyên nhân (auth, connection refused, sai tham số key...).

---

## 6. Khuyến nghị vận hành lâu dài

1. **Chuẩn hóa version Zabbix Agent2** trên toàn bộ fleet Redis nếu có thể (khuyến nghị 6.4+ hoặc mới hơn), để thống nhất dùng cú pháp `Default.*` — giúp macro `{$REDIS.CONN.URI}` luôn ở giá trị mặc định `tcp://localhost:6379` cho mọi host, giảm rủi ro cấu hình sai khi mở rộng.
2. **Lập bảng mapping vận hành** (host, version Agent2, tên session nếu có, giá trị macro cần set) để đội vận hành tránh nhầm lẫn khi thêm host Redis mới, đặc biệt trong môi trường có nhiều thế hệ hạ tầng khác nhau.
3. **Tránh dùng ký tự đặc biệt nhạy cảm với parser** (như `#`) trong password nếu có thể, hoặc luôn kiểm tra kỹ bằng `cat -A` sau khi chỉnh sửa để phát hiện sớm lỗi cú pháp/ký tự ẩn.
4. **Luôn test bằng `zabbix_agent2 -t <key>` trực tiếp trên host trước**, sau đó mới test qua `zabbix_get` từ Zabbix Server, để nhanh chóng xác định lỗi nằm ở tầng cấu hình plugin hay tầng network/firewall giữa Server và Agent.
5. Đảm bảo user Redis dùng để giám sát (mặc định hoặc ACL) có đủ quyền chạy các lệnh `PING`, `INFO`, `CONFIG`, `CLIENT`, `SLOWLOG` — hoặc thuộc các category ACL `@admin`, `@slow`, `@dangerous`, `@fast`, `@connection` — nếu không, item sẽ báo lỗi permission dù kết nối và xác thực đã đúng.

---

## Tổng kết

Giám sát Redis bằng Zabbix Agent2 về bản chất khá đơn giản nhờ plugin native tích hợp sẵn, nhưng thực tế triển khai trên hạ tầng có nhiều phiên bản Agent2 khác nhau lại tiềm ẩn nhiều lỗi nhỏ liên quan đến cú pháp cấu hình (`Default` vs `Sessions`), xác thực (password, ACL user), và cách khai báo macro tương ứng trên Zabbix Frontend. Nắm rõ đúng version đang chạy trên từng host, kiểm tra kỹ file cấu hình bằng các lệnh debug (`-t`, `cat -A`, `cat -n`), và chuẩn hóa cách đặt macro theo từng nhóm host là chìa khóa để triển khai giám sát Redis ổn định, dễ mở rộng về sau.
