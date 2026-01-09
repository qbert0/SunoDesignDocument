# AI Music Generation Platform – System Design

## Step 1: Outline Use Cases & Constraints

### Use cases chính

#### 1. Upload & Quản lý bản nhạc

- User upload bản nhạc gốc (audio input)
- Hệ thống lưu:
  - Track gốc
  - Các phiên bản (version)
  - Metadata liên quan
- Bản nhạc có thể được dùng làm reference cho các lần generate sau

#### 2. Generate Music (Trình tạo nhạc)

- User nhập prompt mô tả bài hát
- Có thể kèm:
  - Lyrics
  - Chọn instrumental hoặc có vocal
- Hệ thống AI generate audio mới
- Kết quả được lưu thành một track/version trong Library

#### 3. Library

- Lưu toàn bộ:
  - Track đã upload
  - Track được generate
  - Các phiên bản của mỗi track
  - Lịch sử generation
- User có thể:
  - Nghe lại
  - Tải về

#### 4. Listen / Discover (Nghe nhạc)

- User nghe:
  - Nhạc của chính mình
  - Nhạc public từ cộng đồng
- Stream audio với độ trễ thấp
- Hỗ trợ feed khám phá và tìm kiếm

### 5. Credits / Pricing (Thanh toán)

- Hệ thống có 2 loại plan:
  - Free plan (credits giới hạn)
  - Paid plan (mua thêm credits)
- Credits được dùng cho:
  - Generate nhạc
  - Tạo nhiều bài song song
- Paid user được ưu tiên hàng đợi (priority queue)

---

### Out of scope

- Chỉnh sửa âm thanh nâng cao:
  Chỉnh sửa âm thanh nâng cao: Các chỉnh sửa âm thanh thủ công như lọc tạp âm, tách nhạc, tune, v.v. không được hỗ trợ trong dự án này

---

### Constraints & Requirements

#### 1. Trải nghiệm người dùng

- Âm thanh khi nghe phải:
  - Phát liên tục
  - Không bị ngắt đoạn
- Cho phép bắt đầu nghe trước khi tải toàn bộ bài (streaming)

#### 2. Tính sẵn sàng (Availability)

- Web/App + Playback:
  - Luôn available
- Generate:
  - Có thể chậm
  - Có queue và priority
  - Không yêu cầu realtime

#### 3. Lưu lượng / Traffic

- Traffic không đồng đều:
  - Một số bài / prompt hot gây spike
- Tỷ lệ:
  - 1 người tạo → 100–1000 người nghe
- Quy mô:
  - ~100k người tạo / ngày
  - ~10–100 triệu lượt nghe / ngày

#### 4. Hiệu suất (Performance)

- Generate nhạc:
  - GPU-bound
  - Inference latency biến thiên theo prompt
  - Chấp nhận thời gian xếp hàng để tránh quá tải
- Nghe / tải nhạc:
  - Nhẹ hơn nhiều
  - Read-heavy

#### 5. Tài nguyên (Resources)

- Storage lớn bao gồm:
  - Audio files
  - Waveform
  - Metadata
  - Versioning

#### 6. Tuân thủ & An toàn

- Moderation nội dung:
  - Lyrics
  - Voice-like content
- Abuse detection
- Rate limiting

### Assumptions

- Lyric được viết dưới dạng ngôn ngữ tự nhiên
- Prompt cho style âm nhạc được viết dưới dạng ngôn ngữ tự nhiên lai với ngôn ngữ hướng miền của âm nhạc

## Step 2: Create a high level design

## Step 3 : Design core components

### Use case 1: User upload bản nhạc gốc (Audio Input):

_Mục đích_: Nhận file âm thanh từ user, chuẩn hóa audio để lưu trữ lâu dài và làm input cho các pipeline AI phía sau. Tách upload và xử lý nặng để tránh block request.

- Client upload file qua Web Server, đồng thời làm nhẹ file hoặc áp dụng CDN (Reverse Proxy). _File đầu vào có thể là:.mp3,.wav,.flac_
- Web server gửi lại cho Write API thực hiện các kiểm tra nhẹ, đồng bộ:
  - Kiểm tra kỹ thuật: < 10 phút, < 50MB
  - Kiểm tra an toàn
  - Kiểm tra nghiệp vụ/phiên làm việc: Số lượng request
  - Phản hồi nếu thấy lỗi:
- Lưu file âm thanh: Đẩy file chưa xử lý vào Object Store (như S3).
- Lưu vào Metadata: Lưu thông tin file vào SQL Database
- Write API đẩy event vào Message Queue chờ được thực thi chuẩn hóa. Để làm nhẹ request
- Worker xử lý chuẩn hóa.
- Lưu audio đã chuẩn hóa và cập nhật metadata

### Use case 2: User tạo/biến đổi nhạc (Generate/Transform):

_Mục đích_: Biến input của user (prompt / lyrics / audio tham chiếu) thành một bản nhạc mới. Đồng thời quản lý tài nguyên cho GPD và đáp ứng được khi có lượng truy cập lớn.

Use case này là một **Asynchronous AI Inference Pipeline**, có các đặc điểm:

- Không realtime
- GPU-bound
- Có thể retry
- Có versioning
- Gắn chặt với billing

Các bước bao gồm

- Client gửi yêu cầu Generate: prompt, lyrics, dòng nhạc, kế thừa âm thanh, chọn model AI
- Web server gửi yêu cầu vào hàng đợi(Queue) của Music service
- Music service thực hiện xác minh
- Mucsic service thực hiện gen nhạc
- Lưu vào Storage S3MoniO.
- Lưu lại thông tin vào Metadata

### Use case 3: Phát trực tuyến nhạc (Streaming/Playback):

_Mục đích_: CHo phép nghe nhạc liên tục, không gián đoạn, Khả năng chịu tải lớn. Các bản nhạc của bản thân hoặc cộng đồng.

- Client request phát nhạc
- Web server gửi reqquest đến Read API
- Read API – Xác thực & kiểm tra quyền
- Sinh Streaming URL an toàn, lưu file nhạc vào CDN.
- Client bắt đầu stream qua CDN. Với nhạc public được nghe lại nhiều, lưu sẵn vào CDN, thời gian tồn tại lâu hơn

## Step 4: Scale the design

- Sử dụng các load balancer để cân bằng tải không bị quá tải khi có lượng lớn người dùng
- Chia ra thành các service để sử lý tập trung nhiệm vụ không dồn vào 1 service gây quá tải, khó bảo trì
