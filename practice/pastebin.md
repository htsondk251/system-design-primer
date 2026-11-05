# Mô phỏng phỏng vấn: Thiết kế Pastebin

**Trạng thái:** đang ở vòng hỏi sâu, chưa kết thúc và chưa chấm điểm.

## 1. Yêu cầu

Đề bài: thiết kế một dịch vụ giống Pastebin. Người dùng dán một đoạn text vào, hệ thống trả về một link ngắn. Ai có link đều xem được nội dung.

### Yêu cầu chức năng

- Tạo và xem paste không cần đăng ký hay đăng nhập.
- Mặc định paste không hết hạn. Người dùng có thể đặt thời hạn khi tạo (ví dụ 10 phút, 1 ngày, 1 tháng).
- Paste đã hết hạn thì không xem được nữa.
- Paste không sửa được và không xóa thủ công được. Paste chỉ biến mất khi hết hạn.
- Link phải ngắn và không trùng nhau. Độ dài do ứng viên quyết định.

### Yêu cầu phi chức năng

| Tiêu chí | Yêu cầu |
|---|---|
| Kích thước paste | Trung bình khoảng 1 KB, tối đa 10 MB |
| Người dùng | Khoảng 10 triệu |
| Lượt ghi | Khoảng 10 triệu/tháng |
| Lượt đọc | Khoảng 100 triệu/tháng |
| Thời gian lưu trữ | Chạy ổn định trong 3 năm |
| Phân bố lưu lượng | Không đều, có giờ cao điểm |
| Latency đọc | p99 dưới khoảng 100 ms, tính phía server |
| Latency ghi | p99 dưới khoảng 500 ms |
| Tính sẵn sàng | Cao (high availability) |
| Tính nhất quán | Read-after-write: nhận link xong phải mở được ngay, không bị 404 |

## 2. Ước lượng

| Đại lượng | Cách tính | Kết quả |
|---|---|---|
| Số paste sau 3 năm | 10 triệu/tháng × 36 tháng | 360 triệu |
| Kích thước mỗi bản ghi | 1 KB text + metadata | Khoảng 1,2 KB |
| Tổng dung lượng sau 3 năm | 360 triệu × 1,2 KB | Khoảng 440 GB |
| Ghi trung bình | 10 triệu/tháng | Khoảng 4 lượt/s |
| Đọc trung bình | 100 triệu/tháng | Khoảng 40 lượt/s |

Giả định cao điểm do interviewer đưa ra: gấp 10 lần trung bình, tức khoảng 40 lượt ghi/s và 400 lượt đọc/s.

## 3. Thiết kế đến thời điểm hiện tại

Đây là thiết kế của ứng viên, gồm cả phần cập nhật ở vòng hỏi sâu.

### API

- `/add`: request gồm `text` và thời hạn (không bắt buộc), response trả về `short_link`.
- `/get`: request gồm `short_link`, response trả về URL của nội dung trên object store. Client tự gọi URL đó để lấy text.

### Kiến trúc

```
Client ──> CDN (nếu có người dùng toàn cầu)
Client ──> Load Balancer ──> Web service (2 instance trở lên)
                               ├──> Cache
                               ├──> Database (metadata, replicate qua 3 AZ)
                               └──> Object store như S3 (nội dung paste)
Client ──> Object store (client tự tải nội dung bằng URL)
```

### Dữ liệu

Bảng `paste` trong database:

| Cột | Ý nghĩa |
|---|---|
| `id` | Khóa chính |
| `short_link` | Link ngắn, có index để đọc nhanh |
| `url` | URL của nội dung trên object store |
| `created_at` | Thời điểm tạo |
| `expire_at` | Thời điểm hết hạn, được đặt nếu request có thời hạn |

### Luồng xử lý

- **Tạo paste (`/add`):** web service sinh `short_link`, lưu nội dung lên object store, lưu metadata vào database, rồi trả `short_link` về client.
- **Đọc paste (`/get`):** web service tìm theo `short_link`, kiểm tra `expire_at`, rồi trả URL nếu paste chưa hết hạn.

### Các quyết định khác

- **Tính sẵn sàng:** đặt web service sau load balancer và replicate database qua 3 availability zone.
- **Cao điểm dự đoán được:** lập lịch tự scale service.
- **Cao điểm không dự đoán được:** thêm cache, vì tải đọc nhiều, ghi ít, và paste thường chỉ được đọc trong vài giờ đầu.
- **Read replica:** ban đầu dùng để tăng thông lượng đọc, sau đó ứng viên đề xuất có thể bỏ và chuyển sang cache.
- **Người dùng toàn cầu:** thêm CDN để giảm latency.
- **Cập nhật sau vòng 2:** một instance SQL (Postgres/MySQL) cùng một standby để HA, không có read replica, không tự scale ban đầu. Cache-aside với TTL = min(hạn paste, 1 giờ). `short_link` = hash có khóa bí mật của `id` (AUTO_INCREMENT), cắt 7 ký tự base62, trùng thì hash lại, bắt va chạm bằng ràng buộc UNIQUE.

## 4. Các câu hỏi của ứng viên đã được giải đáp

- **SQL có đạt latency 100 ms không?** Có. Tra cứu theo index B-tree thường mất khoảng 1–10 ms, kể cả với bảng hàng trăm triệu dòng, miễn là index nằm gọn trong RAM. Muốn chắc chắn thì đo bằng benchmark.
- **Một instance SQL có chứa được 440 GB không?** Được. Một instance có thể chứa vài TB, và RDS trên AWS lưu được tới khoảng 64 TB.
- **Redis có phù hợp để lưu 440 GB không?** Redis giữ toàn bộ dữ liệu trong RAM, nên lưu lượng dữ liệu này vào đó tốn kém và không cần thiết.
- **Best practice xử lý fail-over database?** Gồm: (1) phát hiện lỗi bằng health check với nhiều lần thất bại liên tiếp để tránh flapping; (2) dùng dịch vụ quản lý (RDS Multi-AZ) hoặc công cụ như Patroni có quorum (etcd/Consul) hoặc witness để tránh split-brain, và fence primary cũ; (3) trỏ ứng dụng vào endpoint ổn định (DNS TTL thấp hoặc proxy), connection pool phải kiểm tra và tạo lại kết nối; (4) ứng dụng retry có exponential backoff và jitter, `/add` trả 503 kèm `Retry-After` trong lúc fail-over; (5) với replicate đồng bộ chỉ một standby, quyết định trước khi standby chết thì chặn ghi hay hạ xuống bất đồng bộ; (6) diễn tập fail-over định kỳ và theo dõi replication lag, đo RTO/RPO. RTO điển hình: khoảng 60–120 giây với RDS Multi-AZ, khoảng 10–30 giây với Patroni (con số thay đổi theo cấu hình, cần đo).

## 5. Các câu hỏi chưa được giải đáp

### Tải cao điểm

- [x] Với khoảng 400 lượt đọc/s và 40 lượt ghi/s, một instance SQL có index có chịu nổi không?
  - Trả lời: chịu được. Ứng viên nhớ rằng 2 vCPU/8 GB đạt khoảng 5000 đọc/s và 500 ghi/s, tức dư khoảng 12 lần. Đây là con số ước lượng theo trí nhớ, cần benchmark. Lưu ý: ở năm thứ 3 index `short_link` có thể lớn hơn 8 GB RAM (xem mục 5, nhóm Short link), nên hiệu năng này chỉ đúng nếu tập paste được đọc thường xuyên nằm gọn trong RAM.
- [x] Theo số liệu đó, có thực sự cần lịch tự scale, read replica và cache ngay từ đầu không?
  - Trả lời: không cần tự scale, không cần read replica. Giữ cache. Lý do giữ cache "giảm tải DB" mâu thuẫn với kết luận DB còn dư 12 lần, cần nêu lại lý do và dấu hiệu để thêm cache (chưa trả lời).

### Short link

- [ ] Snowflake ID dài bao nhiêu bit, và đổi sang base62 thì được bao nhiêu ký tự?
- [ ] Snowflake có khớp với lựa chọn chuỗi 7 ký tự ban đầu không? Nếu không thì chọn cách nào?
- [ ] Snowflake ID tăng theo thời gian. Việc này có vấn đề gì với dịch vụ mà *ai có link mới xem được*?
- [x] Nếu hai request tạo paste cùng lúc sinh ra trùng `short_link` thì xử lý ra sao? Câu này còn phụ thuộc vào cách sinh link được chọn.
  - Trả lời: `short_link = hash(id)` cắt còn 7 ký tự base62. Nếu trùng thì nối thêm một chuỗi cố định rồi hash lại, lặp đến khi duy nhất. Với 360 triệu paste, số va chạm kỳ vọng khoảng 18.000 (n²/2N, N = 62^7 ≈ 3,5 nghìn tỷ), tức khoảng 0,005% lượt ghi. Xác suất trùng hai lần liên tiếp cỡ 10⁻⁸ nên gần như không ảnh hưởng p99 ghi.
  - Còn phải làm đúng: (1) phát hiện trùng bằng ràng buộc UNIQUE trên `short_link` và bắt lỗi khi INSERT, không kiểm tra trước rồi mới INSERT vì sẽ bị race. (2) Hash phải có khóa bí mật (HMAC), cả chuỗi nối thêm cũng phải bí mật. Nếu thuật toán công khai, kẻ tấn công tự tính `hash(1..N)` và duyệt được mọi paste, không khác gì link tuần tự. (3) Với khóa bí mật, mật độ link hợp lệ khoảng 360 triệu / 3,5 nghìn tỷ ≈ 10⁻⁴, tức đoán ngẫu nhiên 1 trong khoảng 10.000 lần trúng một paste, nên cần rate limit theo IP cho lượt 404.
  - Việc cần làm rõ khi cài đặt: `short_link` phụ thuộc `id` nên phải biết `id` trước khi INSERT (dùng sequence lấy `id` trước) hoặc INSERT rồi UPDATE.

### Cache và tính sẵn sàng

- [x] Chọn chiến lược cache nào và đặt TTL ra sao?
  - Trả lời: cache-aside (lazy), TTL = min(thời gian còn lại đến `expire_at`, 1 giờ). Paste không đặt hạn thì TTL là 1 giờ. Paste bất biến và mỗi paste có key mới, nên "invalidate khi có bản ghi mới" chỉ cần nếu có cache kết quả 404 (negative caching).
- [x] Một paste còn 5 phút nữa hết hạn được đưa vào cache với TTL 1 giờ. Làm sao để không trả về paste đó sau khi đã hết hạn?
  - Trả lời: đặt TTL = min(hạn paste, 1 giờ) nên cache tự xóa đúng lúc paste hết hạn. Có thể thêm `expire_at` vào giá trị cache và kiểm tra lại khi đọc, phòng lệch đồng hồ.
- [x] Nếu bỏ read replica, điều gì xảy ra khi primary DB gặp sự cố? Dữ liệu vừa ghi có thể bị mất không?
  - Trả lời: một primary và một standby (active-passive) khác AZ, replicate đồng bộ (RTT dưới 2 ms, thêm khoảng vài ms cho mỗi lần ghi), nên không mất dữ liệu đã xác nhận và giữ read-after-write. Mọi request đi qua primary. Cache giúp phục vụ đọc trong lúc fail-over, và nếu cache chết thì DB vẫn chịu được 400 lượt đọc/s.
  - Còn phải xử lý: nếu standby chết thì primary đồng bộ bị treo ghi (chọn chặn ghi hoặc hạ xuống bất đồng bộ), cần cơ chế chống split-brain, và ứng dụng cần retry khi fail-over (xem mục 4).

### Hết hạn

- [ ] Kiểm tra `expire_at` khi `/get` chỉ chặn đọc. Dữ liệu đã hết hạn trong database và S3 được dọn như thế nào?
  - Ứng viên trả lời: database dựa vào `expire_at` (coi như xóa mềm), thêm logic xóa text trên S3 và CDN. Chưa rõ: ai chạy logic đó, chạy khi nào, theo thứ tự nào. Cần chốt có xóa hẳn dòng trong database không (giữ lại thì không tái sử dụng được `short_link` cũ, nhưng bảng phình mãi), và cần index trên `expire_at` cho job dọn.
  - Ứng viên bổ sung: job nền chạy định kỳ 5–10 phút, và nhận ra điều này dẫn đến câu hỏi tiếp theo: giữa lúc paste hết hạn và lúc job chạy, URL S3 vẫn đọc được (cửa sổ tối đa bằng chu kỳ job cộng thời gian chạy). Còn thiếu: thứ tự xóa, giữ hay xóa dòng database, cách đóng cửa sổ này.
- [ ] Paste đã hết hạn mà ai đó vẫn giữ URL S3 thì sao?
  - Chưa trả lời. Vấn đề: `/get` trả thẳng URL S3 nên người giữ URL vẫn đọc được đến khi dọn xong. Hướng cần cân nhắc: bucket private và pre-signed URL có thời hạn = min(thời gian còn lại, vài phút).
- [ ] Làm sao để CDN không tiếp tục phục vụ paste đã hết hạn?
  - Ứng viên trả lời: xóa trên CDN khi hết hạn. Cách này đắt và chậm (invalidation có giới hạn, mất vài phút). Hướng cần cân nhắc: đặt `Cache-Control: max-age/s-maxage` = thời gian còn lại của paste để CDN tự hết hạn đúng lúc.

### Lưu trữ trên S3

- [ ] Lưu mọi paste trên S3 hay chỉ paste lớn?
- [ ] Với yêu cầu p99 dưới 100 ms, luồng `/get` trả URL rồi client gọi tiếp có ảnh hưởng gì?
- [ ] Khi tạo paste, ghi vào S3 và database theo thứ tự nào? Nếu ghi được một bên rồi bên kia lỗi thì xử lý ra sao?

## 6. Quyết định còn bỏ ngỏ trong thiết kế

- **Loại database:** đã chốt dùng SQL (Postgres hoặc MySQL), một instance, không chuyển sang DynamoDB.
- **Tính nhất quán:** ứng viên cho rằng một database là đủ để có strong consistency. Điều này đúng nếu mọi lần đọc đi qua primary. Còn một điểm chưa khớp: yêu cầu high availability và "replicate qua 3 AZ" (mục 3) mâu thuẫn với "chỉ có 1 database". Đã chốt: một primary và một standby khác AZ, replicate đồng bộ (mục 5).
