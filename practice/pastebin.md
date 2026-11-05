# Mô phỏng phỏng vấn: Thiết kế Pastebin

**Trạng thái:** đã kết thúc sau vòng 4 (ứng viên yêu cầu kết thúc). Đã chấm điểm ở mục 9. Các điểm còn bỏ ngỏ được giải đáp theo best practice ở mục 8.

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

## 3. Thiết kế đến hết vòng 2 (lịch sử)

Đây là thiết kế của ứng viên đến hết vòng 2, giữ nguyên để đối chiếu. Thiết kế cuối cùng sau vòng 4 nằm ở mục 7.

### API

- `/add`: request gồm `text` và thời hạn (không bắt buộc), response trả về `short_link`.
- `/get`: request gồm `short_link`, response trả về URL của nội dung trên object store. Client tự gọi URL đó để lấy text.

### Kiến trúc

```
Client ──> CDN (nếu có người dùng toàn cầu)
Client ──> Load Balancer ──> Web service (2 instance trở lên)
                               ├──> Cache
                               ├──> Database (metadata, primary + standby đồng bộ khác AZ, xem mục 5)
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
  - Trả lời: không cần tự scale, không cần read replica. Giữ cache. Lý do giữ cache "giảm tải DB" mâu thuẫn với kết luận DB còn dư 12 lần, cần nêu lại lý do và dấu hiệu để thêm cache.
  - Trả lời vòng 3: bỏ cache ngay từ đầu. Chỉ thêm khi tải đọc làm DB duy trì khoảng 60% CPU và RAM liên tục.
  - Nhận xét: quyết định bỏ cache là hợp lý với tải này. Hai điểm cần sửa: (1) "60% RAM" không phải dấu hiệu tốt, vì buffer pool của DB luôn dùng gần hết RAM được cấp; dấu hiệu nên dùng là p99 đọc tiến gần SLO 100 ms, CPU DB duy trì cao, hoặc buffer cache hit ratio tụt. (2) Ở vòng 2 bạn dựa vào cache để phục vụ đọc trong lúc fail-over (60–120 giây). Bỏ cache thì lập luận đó không còn, cần chốt lại là chấp nhận mất đọc trong fail-over hay không (xem câu hỏi vòng 4).

### Short link

- [x] Snowflake ID dài bao nhiêu bit, và đổi sang base62 thì được bao nhiêu ký tự?
- [x] Snowflake có khớp với lựa chọn chuỗi 7 ký tự ban đầu không? Nếu không thì chọn cách nào?
- [x] Snowflake ID tăng theo thời gian. Việc này có vấn đề gì với dịch vụ mà *ai có link mới xem được*?
  - Ba câu trên đã được thay thế bởi quyết định ở vòng 2: không dùng Snowflake, dùng HMAC của `id` AUTO_INCREMENT cắt 7 ký tự base62. Điểm mấu chốt của câu thứ ba (ID tăng dần thì đoán được link) đã được xử lý bằng khóa bí mật, xem câu trả lời bên dưới. Ghi nhớ: Snowflake 64 bit đổi sang base62 cần 11 ký tự, không khớp 7 ký tự.
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

- [x] Kiểm tra `expire_at` khi `/get` chỉ chặn đọc. Dữ liệu đã hết hạn trong database và S3 được dọn như thế nào? (chốt ở mục 8.2)
  - Ứng viên trả lời: database dựa vào `expire_at` (coi như xóa mềm), thêm logic xóa text trên S3 và CDN. Chưa rõ: ai chạy logic đó, chạy khi nào, theo thứ tự nào. Cần chốt có xóa hẳn dòng trong database không (giữ lại thì không tái sử dụng được `short_link` cũ, nhưng bảng phình mãi), và cần index trên `expire_at` cho job dọn.
  - Ứng viên bổ sung: job nền chạy định kỳ 5–10 phút, và nhận ra điều này dẫn đến câu hỏi tiếp theo: giữa lúc paste hết hạn và lúc job chạy, URL S3 vẫn đọc được (cửa sổ tối đa bằng chu kỳ job cộng thời gian chạy). Còn thiếu: thứ tự xóa, giữ hay xóa dòng database, cách đóng cửa sổ này.
  - Trả lời vòng 3 (thứ tự xóa): xóa object S3 trước, rồi mới xóa dòng database. Lý do: nếu xóa database trước và job chết giữa chừng thì object S3 thành mồ côi, không còn ai biết để xóa.
  - Nhận xét: đúng và tự sửa được. Nếu job chết sau khi xóa S3, dòng database vẫn có `expire_at < now` nên lần chạy sau tìm lại được; lệnh xóa S3 trên key không tồn tại là idempotent (trả 204), rồi xóa dòng. `/get` kiểm tra `expire_at` trước nên trạng thái "dòng còn, object mất" không lộ ra 404 từ S3. Còn thiếu: chưa trả lời giữ hay xóa dòng database; chưa nói index trên `expire_at` và cách job quét theo lô (`WHERE expire_at < now() LIMIT N`) để không khóa bảng lâu.
- [x] Paste đã hết hạn mà ai đó vẫn giữ URL S3 thì sao?
  - Trả lời vòng 3: bucket private, `/get` trả pre-signed URL có thời hạn = min(`expire_at`, now + chu kỳ job dọn).
  - Nhận xét: đóng được cửa sổ, vì thành phần `expire_at` làm chữ ký hết hiệu lực đúng lúc paste hết hạn, không phụ thuộc job. Thành phần "now + chu kỳ job" chỉ có tác dụng với paste không đặt hạn, và nên nói rõ mục đích là giới hạn thời gian URL bị chia sẻ lại. Còn hở: nếu cache (hoặc sau này CDN) lưu chính pre-signed URL với TTL 1 giờ thì sẽ trả URL đã hết chữ ký; nên cache object key và ký ngay lúc trả lời (ký là HMAC cục bộ, không gọi mạng, dưới 1 ms). Bucket private cũng làm CDN không đọc thẳng S3 được, cần origin access (CloudFront OAC) hoặc bỏ CDN cho nội dung.
- [x] Làm sao để CDN không tiếp tục phục vụ paste đã hết hạn? (chốt ở mục 8.4)
  - Ứng viên trả lời: xóa trên CDN khi hết hạn. Cách này đắt và chậm (invalidation có giới hạn, mất vài phút). Hướng cần cân nhắc: đặt `Cache-Control: max-age/s-maxage` = thời gian còn lại của paste để CDN tự hết hạn đúng lúc.

### Lưu trữ trên S3

- [x] Lưu mọi paste trên S3 hay chỉ paste lớn?
- [x] Với yêu cầu p99 dưới 100 ms, luồng `/get` trả URL rồi client gọi tiếp có ảnh hưởng gì?
  - Trả lời vòng 3: tách hai luồng theo ngưỡng 64 KB. Dưới hoặc bằng 64 KB lưu thẳng trong database và `/get` trả text luôn; trên 64 KB lưu S3 và trả pre-signed URL. Trường hợp xấu nhất 400 đọc/s đều trúng text 64 KB, băng thông ứng viên tính "khoảng 26 Mb/s".
  - Nhận xét: hướng tách là đúng, với trung bình 1 KB thì gần như toàn bộ paste đi luồng một bước, bỏ được round trip thứ hai. Sai đơn vị: 400 × 64 KB = 25,6 MB/s, tức khoảng 205 Mbit/s, không phải 26 Mb/s; mức này vẫn ổn với một web tier nhưng là 8 lần con số đã nêu. Hệ quả chưa nêu: bảng `paste` cần thêm cột `content` (nullable) và `url` thành nullable, hoặc một cột `storage` để biết đọc ở đâu; ước lượng 440 GB vẫn đúng vì text nằm trong DB thay vì S3; Postgres tự TOAST cột text trên 2 KB nên không ảnh hưởng index. Response của `/get` giờ có hai dạng (text hoặc URL), API cần nói rõ.
- [x] Khi tạo paste, ghi vào S3 và database theo thứ tự nào? Nếu ghi được một bên rồi bên kia lỗi thì xử lý ra sao?
  - Trả lời vòng 3: ghi S3 trước rồi INSERT database. Nếu INSERT lỗi thì chỉ sinh object mồ côi, không sinh link 404 (giữ read-after-write). Thứ tự ngược lại thì INSERT xong, ghi S3 lỗi sẽ tạo link 404. Dọn mồ côi bằng job nền chạy mỗi giờ, chỉ quét object tạo trong 1 giờ gần nhất, so với database, thiếu thì xóa.
  - Nhận xét: phân tích hai thứ tự đúng và chọn đúng. Lưu ý: INSERT lỗi thì `/add` trả lỗi cho client, client không nhận link, nên "không 404" là đúng theo nghĩa client. Hai điểm chưa làm được như mô tả: (1) S3 ListObjects không lọc theo thời gian tạo; với key là `short_link` ngẫu nhiên thì "quét object trong 1 giờ gần nhất" phải liệt kê cả bucket (360 triệu object nếu lưu hết; ít hơn nhiều nếu chỉ paste trên 64 KB). Cần key có tiền tố thời gian, ví dụ `yyyy/mm/dd/hh/<short_link>`, để list theo prefix của giờ trước. (2) Race: object vừa upload mà INSERT chưa kịp chạy sẽ bị job coi là mồ côi; job phải bỏ qua object trẻ hơn vài phút, hoặc chỉ quét giờ đã khép (giờ trước, không quét giờ hiện tại).

## 5b. Vòng 4: câu hỏi và trả lời

- **Giữ hay xóa dòng database của paste hết hạn, và ước lượng index?**
  - Ứng viên không chọn, thay vào đó hỏi lại cách ước lượng index cho 360 triệu bản ghi. Phần hướng dẫn nằm ở mục 8.1. Câu "giữ hay xóa" được chốt theo best practice ở mục 8.2.
- **Fail-over khi không còn cache?**
  - Trả lời: giữ lại cache, với lý do dự phòng cho database lúc peak và khớp query pattern (đọc nhiều trong vài giờ đầu).
  - Nhận xét: đây là lần đổi ý thứ ba về cache (vòng 2 giữ, vòng 3 bỏ, vòng 4 giữ) và hai lý do nêu ra chính là hai lý do đã bị chỉ ra là mâu thuẫn với kết luận DB dư 12 lần. Lý do hợp lệ duy nhất mà câu hỏi gợi ra (phục vụ đọc trong 60–120 giây fail-over) không được bàn tới, và phép tính availability không được làm. Xem mục 8.3.
- **Key S3 để job dọn mồ côi chạy được?**
  - Trả lời: key = timestamp tới mili giây `yyyyMMddHHmmssSSS` ghép với `short_link`, lọc theo tên để tìm object tạo trong 1 giờ gần nhất. Bỏ cột `url`.
  - Nhận xét: hướng đúng, prefix theo giờ (`yyyyMMddHH`) cho phép list bằng một prefix. Hai điều kiện để bỏ được `url`: `created_at` phải do ứng dụng gán đúng tới mili giây và dùng chung giá trị cho cả key lẫn INSERT, không dùng default của database; và quy ước key phải cố định mãi mãi (đổi là mất dấu object cũ). Chưa trả lời phần race (object vừa upload, INSERT chưa commit). Cách đóng: job chỉ quét giờ đã khép (giờ trước), không quét giờ hiện tại, như vậy mọi object trong phạm vi quét đều đã quá vài phút tuổi.
- **Hợp đồng API `/get`?**
  - Trả lời: response có hai trường tách riêng `text` và `url`; có `url` thì client gọi S3, có `text` thì hiển thị. Mã lỗi riêng cho link không tồn tại và link hết hạn. Với khoảng 26 MB/s thì không cần giới hạn gì ở tầng web.
  - Nhận xét: hai trường tách riêng là đủ, nên có thêm `expire_at` để client hiển thị. 404 cho không tồn tại và 410 Gone cho hết hạn là chuẩn. "Không cần giới hạn gì" mâu thuẫn với kết luận ở mục 5 (nhóm Short link) rằng cần rate limit theo IP cho lượt 404 để chống đoán link; băng thông không phải lý do duy nhất để giới hạn. Thiếu các header bảo mật cho nội dung do người dùng tải lên (xem mục 8.5).

## 6. Quyết định còn bỏ ngỏ trong thiết kế

- **Loại database:** đã chốt dùng SQL (Postgres hoặc MySQL), một instance, không chuyển sang DynamoDB.
- **Tính nhất quán:** ứng viên cho rằng một database là đủ để có strong consistency. Điều này đúng nếu mọi lần đọc đi qua primary. Còn một điểm chưa khớp: yêu cầu high availability và "replicate qua 3 AZ" (mục 3) mâu thuẫn với "chỉ có 1 database". Đã chốt: một primary và một standby khác AZ, replicate đồng bộ (mục 5).

## 7. Thiết kế cuối cùng (sau vòng 4, đã chốt theo best practice ở mục 8)

### API

- `POST /add`: body `text` (tối đa 10 MB), `ttl` tùy chọn. Response `201` với `short_link`, `expire_at` (null nếu không hạn). Lỗi tạm thời (fail-over) trả `503` kèm `Retry-After`.
- `GET /get/{short_link}`: response `200` với `expire_at` và đúng một trong hai trường: `text` (paste tới 64 KB) hoặc `url` (pre-signed URL S3, paste trên 64 KB). `404` nếu không tồn tại, `410` nếu hết hạn. Rate limit theo IP cho lượt 404/410.

### Kiến trúc

```
Client ──> Load Balancer ──> Web service (2 instance trở lên, 2 AZ)
                               ├──> Cache (Redis, cache-aside, chỉ giữ paste tới 64 KB)
                               ├──> Postgres primary ──(sync)──> standby khác AZ
                               └──> S3 private bucket (paste trên 64 KB)
Client ──> S3 (pre-signed URL, chỉ với paste lớn)
Job dọn hết hạn (mỗi 5–10 phút) ──> S3 rồi DB, theo lô
Job dọn mồ côi (mỗi giờ, quét giờ trước) ──> list prefix S3, so DB, xóa
```

### Dữ liệu

| Cột | Ý nghĩa |
|---|---|
| `short_link` | `CHAR(7)`, khóa chính. Sinh bằng HMAC(`id`) cắt 7 ký tự base62 |
| `id` | `BIGINT` lấy từ sequence trước khi INSERT, chỉ làm đầu vào HMAC, không index |
| `content` | `TEXT`, nullable, có giá trị khi paste tới 64 KB |
| `created_at` | `TIMESTAMPTZ(3)`, ứng dụng gán tới mili giây, dùng để suy ra key S3 |
| `expire_at` | `TIMESTAMPTZ`, nullable, partial index `WHERE expire_at IS NOT NULL` |

Key S3 = `yyyyMMddHH/` + `yyyyMMddHHmmssSSS` + `_` + `short_link`. Không có cột `url`.

### Luồng

- **Tạo:** `nextval` lấy `id`, tính `short_link`, gán `created_at`. Nếu trên 64 KB thì PUT S3 trước. INSERT; va chạm UNIQUE thì hash lại với salt bí mật rồi INSERT lại. Trả `short_link`.
- **Đọc:** cache-aside theo `short_link`, TTL = min(thời gian còn lại, 1 giờ). Miss thì đọc DB, kiểm tra `expire_at`. Paste lớn: ký pre-signed URL ngay lúc trả lời, hạn = min(thời gian còn lại, 10 phút); cache chỉ giữ metadata, không giữ URL đã ký.
- **Hết hạn:** `/get` chặn theo `expire_at`. Job dọn chọn `WHERE expire_at < now() ORDER BY expire_at LIMIT 1000`, xóa S3 (nếu paste lớn) rồi DELETE dòng, lặp đến khi hết.

## 8. Giải đáp các điểm còn bỏ ngỏ theo best practice

### 8.1. Cách ước lượng kích thước index B-tree (Postgres)

Cách làm theo từng bước, áp dụng cho bất kỳ index nào:

1. **Tính kích thước một entry ở lá.** Entry = header `IndexTupleData` 8 byte + dữ liệu khóa, làm tròn lên bội số 8 (MAXALIGN), cộng thêm 4 byte line pointer trong trang.
   - `short_link` `CHAR(7)`: varlena ngắn 1 byte header + 7 byte = 8 byte. Entry = 8 + 8 + 4 = **20 byte**.
   - `id` `BIGINT`: 8 byte. Entry = 8 + 8 + 4 = **20 byte**.
   - `expire_at` `TIMESTAMPTZ`: 8 byte. Entry = **20 byte**.
   - Quy tắc nhanh: khóa 8 byte trở xuống thì mỗi entry khoảng 20 byte, khóa 16 byte (UUID) thì khoảng 28 byte.
2. **Tính số entry trên một trang.** Trang 8192 byte, trừ header 24 byte và vùng special 16 byte còn 8152 byte. Trang lá được lấp theo fillfactor mặc định 90% nên dùng khoảng 7.300 byte. 7.300 / 20 ≈ **365 entry/trang**.
3. **Tính số trang lá và cộng tầng trong.** 360 triệu / 365 ≈ 986.000 trang lá. Tầng trong chiếm thêm khoảng 0,3% (986.000 / 365 ≈ 2.700 trang, rồi 8 trang, rồi root). Tổng ≈ 990.000 trang × 8 KB ≈ **8,1 GB cho mỗi index**.
4. **Cộng lại và so với RAM.** Ba index (`id`, `short_link`, `expire_at`) khoảng 24 GB, lớn hơn hẳn 8 GB RAM của cấu hình ứng viên nhớ. Bảng heap chứa `content` trung bình 1 KB: mỗi dòng khoảng 1,1 KB nên khoảng 7 dòng/trang, 360 triệu dòng ≈ 51 triệu trang ≈ **420 GB**, khớp ước lượng 440 GB ở mục 2.
5. **Kiểm chứng.** Nạp 1 triệu dòng mẫu rồi chạy `SELECT pg_size_pretty(pg_relation_size('paste_pkey'))`, nhân lên theo tỷ lệ. Số thật thường cao hơn lý thuyết 5–15% vì trang không đầy đều.

Ba hệ quả cho thiết kế:

- **Index lớn hơn RAM không phá SLO.** B-tree 360 triệu entry cao 4 tầng. Root và tầng 2 luôn nằm trong RAM; trường hợp lạnh nhất đọc 2–3 trang lá/heap từ EBS, mỗi lần khoảng 1 ms, nên một lượt đọc miss hoàn toàn vẫn dưới 10 ms. Lo ngại "index lớn hơn 8 GB" ghi ở mục 5 là đúng về số nhưng không đáng lo về latency.
- **"Working set nằm gọn trong RAM" không áp dụng cho index `short_link`.** Vì `short_link` là hash ngẫu nhiên, paste mới nằm rải đều trên mọi trang lá, nên tập trang lá "nóng" gần bằng cả index. Muốn đọc nóng không chạm đĩa thì RAM phải chứa cả index này (8 GB) cộng heap của các paste nóng; chọn instance 32 GB là hợp lý ở năm thứ 3.
- **Bớt index.** `id` chỉ là đầu vào HMAC, không ai tra cứu theo `id`, nên làm `short_link` thành khóa chính và bỏ index trên `id`, tiết kiệm 8 GB. Index `expire_at` làm partial (`WHERE expire_at IS NOT NULL`) vì mặc định paste không hết hạn, kích thước tỷ lệ với số paste có hạn. Với MySQL InnoDB, bảng cluster theo khóa chính và mỗi index phụ mang theo khóa chính, nên để `short_link` làm khóa chính càng có lợi.

### 8.2. Giữ hay xóa dòng database của paste hết hạn

Xóa hẳn (hard delete), theo lô. Lý do:

- `id` lấy từ sequence nên không bao giờ cấp lại, và `short_link` = HMAC(`id`), nên **xóa dòng không làm link cũ được cấp lại** (chỉ có va chạm hash như đã tính, cỡ 0,005%). Lo ngại "xóa thì tái sử dụng link" không tồn tại trong thiết kế này.
- Giữ dòng chỉ làm bảng và index phình mà không có lợi ích chức năng (paste không cần lịch sử).
- Job chạy `DELETE ... WHERE short_link IN (SELECT short_link FROM paste WHERE expire_at < now() ORDER BY expire_at LIMIT 1000)` lặp cho đến khi không còn, mỗi lô một transaction ngắn, có nghỉ giữa các lô để autovacuum theo kịp. Với 4 ghi/s trung bình, mỗi lần chạy 5 phút chỉ có vài nghìn dòng tới hạn.
- Thay thế khi tải lớn hơn nhiều: partition theo tháng `expire_at` rồi `DROP PARTITION`. Không hợp ở đây vì mặc định paste không hết hạn, phần lớn dòng sẽ rơi vào partition `NULL`.

### 8.3. Cache và fail-over: chốt bằng số

- **Availability không có cache:** giả định mỗi năm 4 lần fail-over ngoài ý muốn và 4 lần bảo trì có chuyển đổi, mỗi lần 2 phút, tổng 16 phút/năm, tức khoảng **99,997%**. Ngay cả 60 phút/năm vẫn là 99,989%. Yêu cầu "high availability" (thường hiểu là 99,9–99,99%) được đáp ứng mà không cần cache.
- **Kết luận:** cache không phải thành phần HA. Nếu giữ cache thì lý do đúng là (1) giảm p99 đọc cho paste nóng, (2) chống hot key (một paste lan truyền được đọc hàng nghìn lần/giây, DB vẫn chịu được vì trang nằm trong buffer pool, nhưng cache rẻ hơn), (3) bảo vệ DB khi tải vượt dự báo 10 lần. Nếu cache được dùng để đọc trong fail-over thì bản thân cache phải Multi-AZ, không thì chỉ thêm một điểm lỗi.
- **Khuyến nghị cho quy mô này:** bắt đầu không cache; thêm cache khi p99 đọc tiến gần 100 ms hoặc CPU DB duy trì trên 60% ở giờ cao điểm. Thiết kế cuối ở mục 7 giữ cache theo lựa chọn của ứng viên, với TTL và nội dung cache đã chốt.

### 8.4. CDN không phục vụ paste đã hết hạn

- Không gọi invalidation khi hết hạn (chậm, có giới hạn số lần). Đặt `Cache-Control: public, max-age=<giây còn lại>, s-maxage=<giây còn lại>` khi trả nội dung, trần 1 giờ; paste không hạn thì `max-age=3600`. CDN tự hết hạn đúng lúc.
- Với bucket private, CDN không đọc thẳng S3 được. Hai lựa chọn: CloudFront với Origin Access Control tới S3 và signed URL của CloudFront thay cho pre-signed URL S3; hoặc bỏ CDN cho nội dung, vì 400 đọc/s và trung bình 1 KB không cần CDN, chỉ dùng CDN cho tài nguyên tĩnh của giao diện.

### 8.5. Bảo mật và giới hạn ở tầng web

- **Rate limit** theo IP cho `/add` (chống spam, ví dụ 10/phút) và cho lượt 404/410 của `/get` (chống đoán link, ví dụ 60/phút). Đây là lý do giới hạn, không phải băng thông.
- **Nội dung do người dùng tải lên:** trả `Content-Type: text/plain; charset=utf-8`, `X-Content-Type-Options: nosniff`, `Content-Disposition: inline`, và với object S3 đặt cùng metadata khi PUT, để không bao giờ phục vụ HTML/JS của người dùng trên domain của dịch vụ. Dùng domain riêng cho nội dung thô nếu có giao diện web.
- **Nén:** bật gzip/br ở load balancer cho `/get`; text nén 3–5 lần, 205 Mbit/s xấu nhất còn khoảng 50 Mbit/s.
- **Giới hạn body** `/add` 10 MB ở load balancer, từ chối sớm bằng `413`.
- **410 và lộ thông tin:** 410 cho biết link từng tồn tại; chấp nhận được với pastebin công khai, nếu muốn kín hơn thì trả 404 cho cả hai.

### 8.6. Các điểm nhỏ còn lại

- `short_link` phụ thuộc `id`: lấy `id` bằng `nextval` trước, tính hash, rồi INSERT với `id` tường minh. Không INSERT rồi UPDATE (hai lần ghi và có khoảng trống dòng chưa có link).
- Standby đồng bộ chết: cấu hình `synchronous_commit` hạ về bất đồng bộ tự động có cảnh báo (RDS làm sẵn), chấp nhận RPO vài giây trong lúc standby được dựng lại thay vì chặn ghi.
- Negative caching: không cần, vì tỷ lệ 404 thấp và đã có rate limit.

## 9. Chấm điểm

Thang 1–5 cho mỗi tiêu chí.

| Tiêu chí | Điểm | Nhận xét |
|---|---|---|
| Làm rõ yêu cầu | 4 | Chốt được yêu cầu chức năng và phi chức năng rõ, có read-after-write và latency cụ thể. |
| Ước lượng | 3 | Phép tính dung lượng và QPS đúng. Sai đơn vị Mb/MB hai lần, lần hai sau khi đã được sửa. Dựa vào số benchmark nhớ trong đầu mà không nêu cách kiểm chứng. Không tự ước lượng được index, phải hỏi lại. |
| Thiết kế tổng thể | 4 | Kiến trúc gọn, đúng tầm với tải: một primary + standby, không tự scale, không read replica. Tách luồng 64 KB là quyết định tốt. |
| Đi sâu thành phần | 3 | Điểm mạnh: thứ tự ghi S3/DB, thứ tự xóa S3/DB, pre-signed URL với min(`expire_at`, ...), công thức va chạm n²/2N. Điểm yếu: job dọn mồ côi như mô tả ban đầu không chạy được trên S3, bỏ qua race với INSERT, không trả lời giữ/xóa dòng DB. Ý về HMAC bí mật và bắt race bằng UNIQUE do người phỏng vấn cung cấp. |
| Đánh đổi và dứt khoát | 2 | Đổi ý về cache ba lần với lý do đã bị chỉ là mâu thuẫn. Trả lời câu hỏi bằng câu hỏi. "Không cần giới hạn gì" trái với kết luận rate limit của chính mình. Không làm phép tính availability khi được yêu cầu. |
| Giao tiếp | 3 | Ngắn gọn, đúng trọng tâm, nhưng thường trả lời thiếu phần trong câu hỏi nhiều ý. |

**Tổng:** 19/30, tương đương **3,2/5**. Đạt mức kỹ sư mid-level cho bài này; chưa đạt mức senior vì thiếu tính dứt khoát khi đánh đổi và chưa tự chủ với số liệu (đơn vị, index, availability).

**Nên ôn tiếp:**

1. Ước lượng kích thước index và heap (mục 8.1), và cách kiểm chứng bằng `pg_relation_size`.
2. Quy đổi đơn vị byte/bit và tính availability từ RTO × số lần sự cố.
3. Khi đổi ý một quyết định, nêu rõ dữ kiện mới nào khiến đổi ý. Nếu không có dữ kiện mới thì giữ nguyên.
4. S3 API: ListObjects theo prefix, không lọc theo thời gian; thiết kế key phục vụ việc quét.
5. Bảo mật nội dung do người dùng tải lên: `nosniff`, `text/plain`, domain riêng.
