# Mô phỏng phỏng vấn: Thiết kế Twitter timeline và search

**Trạng thái:** đã xong yêu cầu và ước lượng, đã xong thiết kế chức năng (API, service, dữ liệu, luồng, luồng riêng cho influencer), chưa giải quyết phi chức năng. Chưa xem lời giải trong `solutions/system_design/twitter/README.md` (xem sau khi tự thiết kế xong).

## 1. Yêu cầu

Đề bài: thiết kế timeline và tìm kiếm của Twitter (tương tự Facebook feed / Facebook search).

### Yêu cầu chức năng

- User đăng tweet. Hệ thống đẩy tweet tới followers, gửi push notification và email.
- User xem user timeline (hoạt động của chính user đó).
- User xem home timeline (hoạt động của những người user đang follow).
- User tìm kiếm theo từ khóa.
- Tính sẵn sàng cao.

### Ngoài phạm vi

- Đẩy tweet sang Firehose và các stream khác.
- Lọc tweet theo cài đặt hiển thị (ẩn @reply, ẩn retweet).
- Analytics.

### Yêu cầu phi chức năng

| Tiêu chí | Yêu cầu |
|---|---|
| Người dùng hoạt động | 100 triệu |
| Tweet mới | 500 triệu/ngày |
| Fanout trung bình | 10 lượt giao/tweet |
| Lượt đọc | 250 tỷ/tháng |
| Lượt search | 10 tỷ/tháng |
| Phân bố lưu lượng | Không đều |
| Đăng tweet | Phải nhanh, kể cả fanout (trừ user có hàng triệu follower) |
| Xem timeline, search | Phải nhanh |
| Hệ thống | Đọc nhiều hơn ghi nhiều |

### Phạm vi và yêu cầu chốt thêm với interviewer

- Search chỉ tìm tweet (không tìm user).
- Hai loại timeline: user timeline (hoạt động của chính user đó) và home timeline (hoạt động của người mình follow).
- Ứng viên tự mở rộng: đăng ký (`/register`) và xóa tweet nằm ngoài danh sách yêu cầu gốc.

### Yêu cầu phi chức năng bổ sung (interviewer chốt)

| Tiêu chí | Yêu cầu |
|---|---|
| Latency timeline (home và user) | p99 dưới 200 ms, tính phía server |
| Latency search | p99 dưới 500 ms |
| Latency đăng tweet | p99 dưới 500 ms, chỉ tính đến lúc lưu tweet và nhận ack. Fanout, notification, email chạy bất đồng bộ nên không tính |
| Tính nhất quán | Eventual consistency. Tweet hiện trên home timeline của follower trễ vài giây là được. Search index trễ tối đa khoảng 1 phút là được. Tác giả phải thấy tweet của mình ngay (read-your-writes) |
| Tính sẵn sàng | Cao. Khi lỗi, thà trả timeline cũ còn hơn không trả gì |
| Hệ số cao điểm | Gấp 5 lần trung bình |

## 2. Ước lượng

Giả định của ứng viên: 1 năm = 360 ngày, lưu 5 năm, mỗi tweet 1 KB (id, author_id, created_at, text, image_url), 10% tweet có 1 ảnh trung bình 500 KB.

| Đại lượng | Cách tính | Kết quả |
|---|---|---|
| Ghi trung bình (tweet/s) | 500 triệu / 86.400 | Khoảng 6.000/s |
| Fanout trung bình (lượt giao/s) | 6.000 × 10 | Khoảng 60.000/s (interviewer tính, ứng viên chưa tự làm) |
| Đọc trung bình (lượt/s) | 250 tỷ / 2,6 triệu giây | Khoảng 100.000/s |
| Search trung bình (lượt/s) | 10 tỷ / 2,6 triệu giây | Khoảng 4.000/s |
| Cao điểm (hệ số 5) | | Khoảng 30.000 tweet/s, 500.000 đọc/s, 20.000 search/s |
| Tweet/năm | 500 triệu × 360 | 180 tỷ |
| Dung lượng text/năm | 180 tỷ × 1 KB | 180 TB (5 năm: 900 TB) |
| Dung lượng ảnh/năm | 18 tỷ ảnh × 500 KB | 9 PB (5 năm: 45 PB) |
| Dung lượng home timeline | 100 triệu user × 600 mục × 1 KB | 60 TB |

Nhận xét: ảnh gấp 50 lần text nên nằm ngoài database (object store, CDN).

## 3. Thiết kế

Đây là thiết kế của ứng viên ở vòng 1 (chỉ thỏa yêu cầu chức năng, phần phi chức năng bàn sau).

### API

- `/register`: tạo user, lưu `(id, user_name)`.
- `/follow`: `follow(follower_id, followee_id)`.
- `/post`: `create(text, images)`, trả về `tweet_id`.
- `/get/timeline/{user_id}` (user timeline): trả tweet của user đó theo thứ tự thời gian giảm dần, phân trang bằng cursor (không dùng offset), tối đa 600 kết quả (30 page).
- `/get/timeline` (home timeline): trả tweet của những người mình follow, mỗi page 20 tweet, giới hạn 600 kết quả. Cursor là `created_at` của tweet cuối page trước. Mỗi lần gọi lấy tối đa 20 phần tử (có `created_at < cursor`) từ `home_timeline_tweet` của user và từ `influencer_tweets` của từng influencer user đang follow, rồi k-way merge, dừng khi đủ 20 tweet. Kết quả merge chỉ dùng để trả về, không ghi ngược vào NoSQL (sửa so với bản đầu).
- `/delete`: ngoài phạm vi, bỏ qua.

### Kiến trúc

- **user-service** kèm một relational database: bảng user và bảng follow.
- **post-service** kèm một relational database cho tweet. Ảnh lưu trên S3, database chỉ giữ URL.
- **Debezium** đọc log của database tweet và publish tweet mới lên message queue.
- **timeline-service** nhận tweet từ queue, tra bảng follow để tìm follower của tác giả, thêm tweet vào home timeline của từng follower. Home timeline lưu trong NoSQL dạng key-value (ví dụ Redis): `user_id` → danh sách tweet của những người user đó follow.
- **search-service** nhận tweet từ queue, xử lý rồi lưu vào Elasticsearch.
- User thường: timeline-service, sau khi lưu xong home timeline, tự publish một message lên một queue dạng topic. **notification-service** và **mail-service** subscribe topic đó để gửi push notification và email.
- Influencer (sửa so với bản đầu): mỗi influencer có một topic riêng, follower subscribe vào topic đó. Khi influencer đăng tweet, publish một notification lên topic của họ, notification-service và mail-service gửi tới các subscriber.
- **influencer_tweets** (key-value): danh sách tweet của influencer, sắp xếp theo thời gian giảm dần. Xem luồng influencer bên dưới.

### Dữ liệu

- `user(id, user_name, is_influencer)`. `is_influencer` là boolean. Tiêu chí chưa cố định (ví dụ từ 1 triệu hoặc 10 triệu follower), ứng viên cho là không quan trọng. Index `(id, is_influencer)`.
- `follow(follower_id, followee_id)`: mỗi dòng là một quan hệ "follower_id follow followee_id". Hai index riêng:
  - `(followee_id)`: trong `/post`, timeline-service (luồng user thường) tìm follower của tác giả bằng điều kiện `followee_id = author_id`.
  - `(follower_id)`: trong `/get/timeline`, tìm các followee của user bằng điều kiện `follower_id = user_id`.
- `influencer_tweets` (key-value): `influencer_id` → danh sách tweet của influencer đó, sắp xếp theo thời gian giảm dần. Giới hạn số phần tử chưa chốt, ứng viên định giới hạn theo thời gian (giữ 5 hoặc 10 ngày).
- Số liệu interviewer cung cấp: influencer đăng trung bình 10 tweet/ngày, ngày cao điểm tới 50 tweet/ngày. Khoảng 0,1% user là influencer (khoảng 100.000 người).
- `tweet(id, user_id, text, image_url, created_at)`, có index `(user_id, created_at)` (sửa so với bản đầu chỉ có `user_id`): user timeline đọc thẳng theo index, không phải sort.
- Home timeline (NoSQL key-value, ví dụ Redis): `user_id` → danh sách tweet theo thời gian giảm dần, tối đa 600 mục. Ứng viên tính 100 triệu user × 600 × 1 KB = 60 TB nếu mỗi mục là một tweet đầy đủ 1 KB.

### Luồng xử lý

- **Đăng tweet:** `/post` → post-service lưu ảnh lên S3 → lưu tweet và URL ảnh vào database → trả `tweet_id`. Sau khi transaction commit, Debezium đọc log và publish lên queue → timeline-service và search-service xử lý bất đồng bộ. Sau khi lưu tweet vào home timeline của các follower, timeline-service tự publish message lên topic → notification-service gửi push notification, mail-service gửi email.
- **Fanout trong timeline-service:** nhận tweet mới → query xem tác giả có phải influencer không, kèm danh sách follower.
  - User thường: trả về danh sách follower, thêm tweet vào home timeline của từng follower.
  - Influencer: trả về danh sách follower rỗng (không fanout), thêm tweet vào `influencer_tweets`.
- **User timeline:** truy vấn bảng tweet theo `user_id` bằng index, sắp xếp theo thời gian giảm dần, phân trang bằng cursor.
- **Home timeline:** query các followee của user, lấy ra những followee là influencer. Với `home_timeline_tweet` của user và `influencer_tweets` của từng influencer đó, lấy 20 phần tử có `created_at < cursor`, k-way merge, dừng khi đủ 20. Trả về, không ghi lại. Giới hạn 600 mục. Ảnh hưởng tới latency p99 < 200 ms: ứng viên sẽ xử lý ở phần phi chức năng.

### Vấn đề ứng viên đã nêu, chưa giải quyết

- (a) Sinh `tweet_id` không trùng khi có nhiều instance (6.000 tweet/s).
- (b) Một database không chứa hết tweet, cần partition/sharding. (a) và (b) bàn ở phần phi chức năng.
- (c) Xử lý ở timeline-service cho user có nhiều follower: đã có thiết kế (cờ `is_influencer`, `influencer_tweets`, merge khi đọc). Còn mở: giới hạn của `influencer_tweets`.

### Nhận xét của interviewer sau khi ứng viên sửa thiết kế

Bản đầu có ba lỗi: đường đọc có ghi (read-modify-write gây race với fanout), merge lặp lại gây trùng tweet, notification của influencer rỗng. Bản sửa:

- **Đã giải quyết:** k-way merge không ghi ngược, hết race và hết trùng. Index `(user_id, created_at)` đúng hướng.
- **Còn hở ở home timeline:**
  1. Cursor chỉ có `created_at`: hai tweet cùng timestamp (nhất là từ nhiều user khác nhau) có thể bị bỏ sót hoặc lặp ở ranh giới page. Cần cursor dạng `(created_at, tweet_id)` hoặc ID sắp xếp được theo thời gian.
  2. "Lấy 20 phần tử có `created_at < cursor`" cần cấu trúc có truy vấn theo khoảng. Redis list không làm được điều này, sorted set (score = thời gian) thì được. Cần chốt cấu trúc dữ liệu.
  3. Đường đọc vẫn cần danh sách influencer mà user follow (hiện là query SQL theo `follower_id`), và số list cần đọc là k. Chưa ước lượng k, chưa tính số lệnh Redis/s ở 500.000 đọc/s, và `influencer_tweets` của influencer nổi tiếng là hot key.
  4. Chưa nói mục home timeline là tweet đầy đủ hay chỉ `tweet_id` (cần thêm bước lấy nội dung).
- **Notification cho influencer:** "subscribe topic" chuyển bài toán fanout sang notification-service chứ không làm nó biến mất. Chưa rõ subscribe có khác follow không, tức là có opt-in không (xem câu hỏi vòng 2). Luồng user thường vẫn phụ thuộc timeline-service xong mới notify.

## 4. Câu hỏi cần tự trả lời

- Search index xây thế nào (inverted index), cập nhật gần real-time ra sao?
- Tính nhất quán: chấp nhận eventual consistency ở đâu?
- Nếu một node cache/DB chết thì chuyện gì xảy ra?

## 5. Chấm điểm

(Điền sau khi xong, so sánh với lời giải.)
