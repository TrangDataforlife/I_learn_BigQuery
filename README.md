# I_learn_BigQuery

🌉 BigQuery External Table — Kết nối Data Lake và Data Warehouse
1. External Table là gì?

External Table là một bảng trong BigQuery không lưu trữ dữ liệu trực tiếp bên trong BigQuery, mà chỉ trỏ tới (point to) dữ liệu đang nằm ở một vị trí lưu trữ bên ngoài — ví dụ Cloud Storage.

Dữ liệu vẫn được lưu tại nguồn (Cloud Storage), không được sao chép/nạp (load) vào BigQuery.
Dù vậy, bạn vẫn có thể truy vấn (query) như một bảng BigQuery bình thường — dùng SELECT, JOIN, WHERE... như mọi bảng khác.
Kiểu dữ liệu (data type) của từng cột trong external table sẽ được BigQuery tự động suy luận (inferred) dựa trên dữ liệu thực tế tại nguồn.

🔑 Điểm chính: External Table = "cửa sổ" để BigQuery nhìn vào dữ liệu bên ngoài, không phải bản sao dữ liệu.

2. Vì sao thường dùng định dạng Parquet?

Khi tạo external table trỏ tới dữ liệu trong Cloud Storage, định dạng file phổ biến được khuyến khích là Parquet, vì hai lý do chính:

Ưu điểm	Giải thích
Nén dữ liệu (compressed)	File Parquet được nén sẵn → chiếm ít dung lượng lưu trữ hơn so với các định dạng dạng text như CSV, JSON
Tự chứa schema (self-describing)	Cấu trúc dữ liệu (schema — tên cột, kiểu dữ liệu) được lưu ngay trong file → dễ quản lý, không cần khai báo schema thủ công khi tạo external table
3. Cách tạo External Table trong BigQuery

Cú pháp dùng CREATE OR REPLACE EXTERNAL TABLE, khai báo định dạng file (format) và đường dẫn tới dữ liệu nguồn (uris):

sql
CREATE OR REPLACE EXTERNAL TABLE `thelook_gcda.product_returns`
OPTIONS (
  format = "PARQUET",
  uris = ['gs://sureskills-lab-dev/DAC2M2L4/returns/returns_*.parquet']
);

Giải thích các thành phần:

Thành phần	Ý nghĩa
thelook_gcda.product_returns	Tên bảng external table sẽ tạo trong BigQuery (dataset thelook_gcda, tên bảng product_returns)
format = "PARQUET"	Khai báo định dạng file nguồn là Parquet
uris = [...]	Đường dẫn tới file/thư mục chứa dữ liệu trên Cloud Storage. Dấu * (wildcard) cho phép trỏ tới nhiều file cùng lúc khớp với mẫu tên (VD: returns_1.parquet, returns_2.parquet...)
Tiền tố gs://	Ký hiệu chuẩn cho biết dữ liệu đang nằm trên Google Cloud Storage
4. Kiểm tra nguồn dữ liệu trên giao diện BigQuery

Sau khi tạo, bạn có thể kiểm tra lại external table trên BigQuery UI, tại cột "Source URI(s)":

Cột này hiển thị vị trí lưu trữ thực tế của dữ liệu mà external table đang trỏ tới.
Nếu thấy tiền tố gs:// → xác nhận dữ liệu đang được lưu trên Cloud Storage, đúng như đã khai báo ở bước tạo bảng (mục 3).

🔑 Điểm chính: Cột "Source URI(s)" chính là bằng chứng trực quan cho thấy: dữ liệu của external table không nằm trong BigQuery, mà chỉ được BigQuery "đọc nhờ" từ Cloud Storage mỗi khi có truy vấn.

5. Ứng dụng thực tế: Kết hợp Data Lake và Data Warehouse

Mục đích chính của External Table: cho phép JOIN dữ liệu đang nằm rải rác ở data lake (Cloud Storage — dữ liệu thô, chưa được nạp vào warehouse) với dữ liệu đã có sẵn trong data warehouse (BigQuery standard table), mà không cần phải nạp (load/ETL) toàn bộ dữ liệu data lake vào BigQuery trước.

Quy trình:

Cloud Storage (Data Lake)          BigQuery (Data Warehouse)
   dữ liệu Parquet                     bảng standard table
        │                                      │
        ▼                                      ▼
   External Table  ───────────  JOIN  ───────────  Standard Table
        │
        ▼
   Kết quả: dữ liệu kết hợp từ cả Data Lake + Data Warehouse
sql
-- Ví dụ: JOIN external table (data lake) với standard table (data warehouse)
SELECT o.*, r.return_reason
FROM `thelook_gcda.orders` AS o          -- standard table (data warehouse)
JOIN `thelook_gcda.product_returns` AS r -- external table (data lake)
  ON o.order_id = r.order_id;

🔑 Điểm chính: Đây chính là giá trị cốt lõi của External Table — giúp truy vấn kết hợp dữ liệu giữa 2 nguồn (data lake ngoài Cloud Storage + data warehouse trong BigQuery) trong cùng một câu lệnh SQL, tiết kiệm thời gian và chi phí so với việc phải ETL toàn bộ dữ liệu vào BigQuery trước khi phân tích.

📌 Tóm tắt nhanh
Câu hỏi	Trả lời
External Table lưu dữ liệu ở đâu?	Vẫn ở nguồn (Cloud Storage), không sao chép vào BigQuery
Có query được như bảng thường không?	Có — dùng SELECT, JOIN như standard table
Định dạng khuyến nghị?	Parquet (nén tốt + tự chứa schema)
Cách kiểm tra dữ liệu trỏ tới đâu?	Xem cột Source URI(s) trên BigQuery UI (tiền tố gs://)
Lợi ích chính?	Kết hợp (JOIN) dữ liệu data lake với dữ liệu data warehouse mà không cần ETL trước
