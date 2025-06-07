
# Tổng quan kiến trúc
Hệ thống Grapfity bao gồm các thành phần chính sau:
- Client: Ứng dụng ReactJS cung cấp giao diện người dùng.
- API Gateway: Express Gateway, đóng vai trò là điểm nhập duy nhất cho các yêu cầu API và định tuyến chúng đến các dịch vụ phù hợp, định tuyến đến Backend Server và Recommender System Server
- Backend Server: Xử lý logic nghiệp vụ chính và tương tác với cơ sở dữ liệu MSSQL.
- Recommender System Server: Cung cấp gợi ý bài hát và theo dõi sự kiện, tương tác với cơ sở dữ liệu PostgreDB.
- Load Balancer: Nginx, phân phối yêu cầu đến hai phiên bản của Recommender System Server theo thuật toán round-robin.
- Cơ sở dữ liệu:
- PostgreDB: Lưu trữ dữ liệu cho Recommender System Server.
- MSSQL Server: Lưu trữ dữ liệu cho Backend Server.
- Containerization: Sử dụng Docker và Docker Compose để đóng gói và triển khai các thành phần.

# Mô tả chi tiết các thành phần
### 1. Client (ReactJS)
Giao diện người dùng được xây dựng bằng ReactJS.
Gửi các yêu cầu API đến địa chỉ http://localhost:8080/api/..., nơi Express Gateway tiếp nhận và xử lý.
### 2. API Gateway (Express Gateway)
Đóng vai trò trung gian, định tuyến các yêu cầu từ client đến các dịch vụ phù hợp dựa trên đường dẫn (subpath).
Áp dụng các chính sách (policies) như proxy, biến đổi yêu cầu/phản hồi, và các chính sách tùy chỉnh.
Cấu hình được định nghĩa trong file gateway.config.yml.
### 3. Backend Server
Xử lý các chức năng cốt lõi như quản lý bài hát (tạo, xóa, v.v.).
Tương tác với cơ sở dữ liệu MSSQL để lưu trữ và truy xuất dữ liệu.
Nhận các yêu cầu từ Gateway qua đường dẫn /api/*.
### 4. Recommender System Server
Cung cấp các gợi ý bài hát và theo dõi sự kiện người dùng.
Tương tác với cơ sở dữ liệu PostgreDB.
Được triển khai với hai phiên bản, đặt sau Nginx để cân bằng tải.

### 5. Load Balancer (Nginx)
Recommender System Server có nhiệm vụ theo dõi hành vi của người dùng (nhận các request gửi thông tin hành vi của người dùng tương tác trên website được gửi về bởi ReactJS) => số lượng request nhiều => hệ thống dễ bị quá tải => áp dụng một load balencer (nginx) để thực hiện điều phối request đến các instance của Recommender System Server => giảm tải và cải thiện hiệu suất của hệ thống.
Phân phối yêu cầu đến hai phiên bản của Recommender System Server theo thuật toán round-robin.
Đảm bảo khả năng chịu tải và tính sẵn sàng cao cho dịch vụ gợi ý.
### 6. Cơ sở dữ liệu
PostgreDB: Lưu trữ thông tin phục vụ việc tạo gợi ý và theo dõi sự kiện.
Gồm bảng events: lưu trữ các sự kiện hành vi người dùng:
Bảng tracks lưu trữ feature data của các file audio được trích xuất bởi mô hình học máy
MSSQL Server: Lưu trữ dữ liệu chung của ứng dụng như thông tin người dùng và metadata bài hát.

# Tương tác và luồng xử lý
Các luồng xử lý chính của hệ thống được mô tả như sau:
### 1. Yêu cầu gợi ý (/api/recommendation*)
Client gửi yêu cầu đến /api/recommendation*.
Gateway sử dụng pipeline recommendation với chính sách recommendation-pipeline-policy.
Chính sách này gọi:
recommenderService tại http://nginx:80/api/recommendation?user_id= để lấy gợi ý.
backendService tại http://backend:8001/api/tracks/getTracksById để lấy thông tin chi tiết bài hát.
Luồng xử lý của recommendation-pipeline-policy
Kiểm tra user_id:
Lấy user_id từ query string.
Nếu thiếu, trả lỗi 400: {"error": "Missing user_id"}.
Gọi Recommender System Server:
Gửi GET đến http://nginx:80/api/recommendation?user_id=<user_id>.
Nhận danh sách track_ids từ phản hồi.
Gọi Backend Server:
Gửi POST đến http://backend:8001/api/tracks/getTracksById.
Payload: {"track_ids": [<danh sách ID>]}.
Header: Content-Type: application/json.
Trả kết quả:
Trả mã 200 và dữ liệu JSON từ Backend Server về client.
Xử lý lỗi:
Bắt lỗi, ghi log: Error in recommendation policy: <message>.
Trả mã 500: {"error": "Internal server error"}.

### 2. Theo dõi sự kiện (/api/event_tracking)
Client gửi yêu cầu đến /api/event_tracking.
Gateway sử dụng pipeline event_tracking, proxy trực tiếp đến dịch vụ Recommender qua http://nginx:80.
Sau đó 
### 3. Xóa bài hát (/api/tracks/:id)
Client gửi yêu cầu đến /api/tracks/:id.
Gateway sử dụng pipeline deleteTrack với chính sách deleteTrack-pipeline-policy.
Chính sách này gọi:
backendService tại http://backend:8001/api/tracks để xóa bài hát.
recommenderService tại http://nginx:80/api/delete_track để cập nhật mô hình gợi ý.
Luồng xử lý của deleteTrack-pipeline-policy
Kiểm tra trackId:
Lấy trackId từ req.params.id.
Nếu thiếu, trả lỗi 400: {"error": "Missing trackId"}.
Gọi Backend Server:
Gửi DELETE đến backendService/<trackId>.
Trả kết quả: mã trạng thái và JSON {message: "Backend delete result", status, data}.
Nếu lỗi hoặc status không phải 200, dừng và trả lỗi: mã (từ response hoặc 500), JSON {error: "Backend delete failed", details}.
Gọi Recommender Service (ngầm):
Nếu Backend thành công, gửi DELETE đến recommenderService/<trackId>.
Thử tối đa 3 lần, delay 500ms giữa các lần.
Ghi log thành công hoặc cảnh báo mỗi lần thử.
Nếu thất bại sau 3 lần, ghi log lỗi, không ảnh hưởng phản hồi client.

### 4. Tạo bài hát (/api/tracks/create-track)
Client gửi yêu cầu đến /api/tracks/create-track.
Gateway sử dụng pipeline createTrack:
Proxy yêu cầu đến Backend qua http://backend:8001.
Áp dụng chính sách createTrack-pipeline-policy, gọi recommenderUrl tại http://nginx:80/api/add_track để thông báo về bài hát mới.
Luồng xử lý của createTrack-pipeline-policy
Lấy dữ liệu từ Backend:
Truy xuất dataFromBackend từ res.locals.proxyResponse.body.
Nếu không có dữ liệu, trả lỗi 500: {"error": "No data from backend response"}.
Trả kết quả về Client:
Gửi mã 200 và dữ liệu JSON từ Backend (dataFromBackend) về client.
Gọi Recommender Service (ngầm):
Lấy track_id và track_file_name từ dữ liệu Backend.
Gửi POST đến recommenderUrl với payload {track_id, track_file_name}, header Content-Type: application/json.
Thử tối đa 3 lần, delay 500ms giữa các lần.
Ghi log thành công hoặc cảnh báo mỗi lần thử.
Nếu thất bại sau 3 lần, ghi log lỗi, không ảnh hưởng phản hồi client.
Xử lý lỗi:
Bắt lỗi bất ngờ, ghi log: [RecommendationPolicy] Unexpected error: <message>.
Trả mã 500: {"error": "Unexpected server error"}.

### 5. Các yêu cầu Backend khác (/api/*)
Các yêu cầu không khớp với các endpoint trên được proxy trực tiếp đến Backend qua http://backend:8001.


# Containerization và triển khai

- Tất cả các thành phần được đóng gói trong các container Docker.  
- Docker Compose được sử dụng để định nghĩa và chạy ứng dụng đa container.  
- Phương pháp này cho phép dễ dàng mở rộng, quản lý và triển khai toàn bộ hệ thống.

| Service         | Address and Port       | Address and Port (external) |
|----------------|------------------------|------------------------------|
| postgres        | postgres:5432          |                              |
| redis           | redis:6379             |                              |
| nginx           | nginx:80               |                              |
| mssql           | mssql:1433             |                              |
| backend         | backend:8001           |                              |
| api_gateway     | api_gateway:8080       | localhost:8080              |
| frontend        | frontend:5173          | localhost:5173              |
| zookeeper       | zookeeper:2181         |                              |
| kafka           | kafka:9092             |                              |
| recommender1    | recommender1:8000      |                              |
| recommender2    | recommender2:8000      |                              |


# Hướng dẫn chạy dự án
- Bổ sung extractor_model.pkl vào recommender/static/matrix/
- Thay đổi file database\mssql\script.sql bằng file sinh dữ liệu (giữ lại tên script.sql) 
- Thay đổi file database\postgre\init-db.sql bằng file sinh dữ liệu (init-db.sql)
 => sinh dữ liệu khi database chưa có dữ liệu ( lần đầu chạy project)
- docker-compose up => chạy project 
