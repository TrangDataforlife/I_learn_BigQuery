# I_learn_BigQuery
# 🌉 BigQuery External Table — Kết nối Data Lake và Data Warehouse

## 📑 Mục lục

- [1. External Table là gì?](#1-external-table-là-gì)
- [2. Vì sao thường dùng định dạng Parquet?](#2-vì-sao-thường-dùng-định-dạng-parquet)
- [3. Cách tạo External Table trong BigQuery](#3-cách-tạo-external-table-trong-bigquery)
- [4. Kiểm tra nguồn dữ liệu trên giao diện BigQuery](#4-kiểm-tra-nguồn-dữ-liệu-trên-giao-diện-bigquery)
- [5. Ứng dụng thực tế: Kết hợp Data Lake và Data Warehouse](#5-ứng-dụng-thực-tế-kết-hợp-data-lake-và-data-warehouse)
- [6. Explorer & Classic Explorer — Kiểm tra Dataset và thêm dữ liệu](#6-explorer--classic-explorer--kiểm-tra-dataset-và-thêm-dữ-liệu)
  - [6.1. Thêm dữ liệu bằng "+ Add Data"](#61-thêm-dữ-liệu-bằng--add-data)
- [📌 Tóm tắt nhanh](#-tóm-tắt-nhanh)

---

## 1. External Table là gì?

**External Table** là một bảng trong BigQuery **không lưu trữ dữ liệu trực tiếp** bên trong BigQuery, mà chỉ **trỏ tới (point to)** dữ liệu đang nằm ở một vị trí lưu trữ bên ngoài — ví dụ **Cloud Storage**.

- Dữ liệu **vẫn được lưu tại nguồn** (Cloud Storage), **không** được sao chép/nạp (load) vào BigQuery.
- Dù vậy, bạn vẫn có thể **truy vấn (query) như một bảng BigQuery bình thường** — dùng `SELECT`, `JOIN`, `WHERE`... như mọi bảng khác.
- Kiểu dữ liệu (data type) của từng cột trong external table sẽ được BigQuery **tự động suy luận (inferred)** dựa trên dữ liệu thực tế tại nguồn.

> 🔑 **Điểm chính:** External Table = "cửa sổ" để BigQuery nhìn vào dữ liệu bên ngoài, không phải bản sao dữ liệu.

---

## 2. Vì sao thường dùng định dạng Parquet?

Khi tạo external table trỏ tới dữ liệu trong Cloud Storage, định dạng file phổ biến được khuyến khích là **Parquet**, vì hai lý do chính:

| Ưu điểm | Giải thích |
|---|---|
| **Nén dữ liệu (compressed)** | File Parquet được nén sẵn → chiếm ít dung lượng lưu trữ hơn so với các định dạng dạng text như CSV, JSON |
| **Tự chứa schema (self-describing)** | Cấu trúc dữ liệu (schema — tên cột, kiểu dữ liệu) được lưu **ngay trong file** → dễ quản lý, không cần khai báo schema thủ công khi tạo external table |

---

## 3. Cách tạo External Table trong BigQuery

Cú pháp dùng `CREATE OR REPLACE EXTERNAL TABLE`, khai báo định dạng file (`format`) và đường dẫn tới dữ liệu nguồn (`uris`):

```sql
CREATE OR REPLACE EXTERNAL TABLE `thelook_gcda.product_returns`
OPTIONS (
  format = "PARQUET",
  uris = ['gs://sureskills-lab-dev/DAC2M2L4/returns/returns_*.parquet']
);
```

**Giải thích các thành phần:**

| Thành phần | Ý nghĩa |
|---|---|
| `thelook_gcda.product_returns` | Tên bảng external table sẽ tạo trong BigQuery (dataset `thelook_gcda`, tên bảng `product_returns`) |
| `format = "PARQUET"` | Khai báo định dạng file nguồn là Parquet |
| `uris = [...]` | Đường dẫn tới file/thư mục chứa dữ liệu trên Cloud Storage. Dấu `*` (wildcard) cho phép trỏ tới **nhiều file cùng lúc** khớp với mẫu tên (VD: `returns_1.parquet`, `returns_2.parquet`...) |
| Tiền tố `gs://` | Ký hiệu chuẩn cho biết dữ liệu đang nằm trên **Google Cloud Storage** |

---

## 4. Kiểm tra nguồn dữ liệu trên giao diện BigQuery

Sau khi tạo, bạn có thể kiểm tra lại external table trên **BigQuery UI**, tại cột **"Source URI(s)"**:

- Cột này hiển thị **vị trí lưu trữ thực tế** của dữ liệu mà external table đang trỏ tới.
- Nếu thấy tiền tố **`gs://`** → xác nhận dữ liệu đang được lưu trên **Cloud Storage**, đúng như đã khai báo ở bước tạo bảng (mục 3).

> 🔑 **Điểm chính:** Cột "Source URI(s)" chính là bằng chứng trực quan cho thấy: dữ liệu của external table **không nằm trong BigQuery**, mà chỉ được BigQuery "đọc nhờ" từ Cloud Storage mỗi khi có truy vấn.

---

## 5. Ứng dụng thực tế: Kết hợp Data Lake và Data Warehouse

**Mục đích chính của External Table:** cho phép **JOIN** dữ liệu đang nằm rải rác ở **data lake** (Cloud Storage — dữ liệu thô, chưa được nạp vào warehouse) với dữ liệu đã có sẵn trong **data warehouse** (BigQuery standard table), mà **không cần** phải nạp (load/ETL) toàn bộ dữ liệu data lake vào BigQuery trước.

**Quy trình:**

```
Cloud Storage (Data Lake)                       BigQuery (Data Warehouse)
   dữ liệu Parquet                                bảng standard table
        │                                                  │
        ▼                                                  ▼
   External Table  ───────────   JOIN   ───────────  Standard Table
                                  │
                                  ▼
   Kết quả: dữ liệu kết hợp từ cả Data Lake + Data Warehouse
```

```sql
-- Ví dụ: JOIN external table (data lake) với standard table (data warehouse)
SELECT o.*, r.return_reason
FROM `thelook_gcda.orders` AS o          -- standard table (data warehouse)
JOIN `thelook_gcda.product_returns` AS r -- external table (data lake)
  ON o.order_id = r.order_id;
```

> 🔑 **Điểm chính:** Đây chính là giá trị cốt lõi của External Table — giúp truy vấn kết hợp dữ liệu giữa 2 nguồn (data lake ngoài Cloud Storage + data warehouse trong BigQuery) trong **cùng một câu lệnh SQL**, tiết kiệm thời gian và chi phí so với việc phải ETL toàn bộ dữ liệu vào BigQuery trước khi phân tích.

---

## 6. Explorer & Classic Explorer — Kiểm tra Dataset và thêm dữ liệu

BigQuery Console cung cấp bảng điều hướng (panel) bên trái để duyệt qua các **project → dataset → table**, có 2 phiên bản giao diện:

| Giao diện | Đặc điểm |
|---|---|
| **Explorer** (giao diện mới) | Mặc định hiện nay, gọn hơn, có thanh tìm kiếm nhanh, nút **`+ Add Data`** để thêm dataset/table/nguồn dữ liệu mới |
| **Classic Explorer** (giao diện cũ) | Cách hiển thị cây thư mục dạng cũ (project → dataset → table), vẫn dùng để **kiểm tra nhanh** danh sách dataset/table đang có mà không cần đổi qua giao diện mới |

> 🔑 **Điểm chính:** Cả 2 giao diện đều dùng để **duyệt và kiểm tra** dataset/table hiện có trong project — Classic Explorer thiên về xem cấu trúc cây, Explorer thiên về thao tác nhanh (tìm kiếm, thêm dữ liệu).

### 6.1. Thêm dữ liệu bằng "+ Add Data"

Nút **`+ Add Data`** ở Explorer cho phép tạo bảng mới và nạp (load) dữ liệu **trực tiếp vào Data Warehouse** hiện có — khác với External Table (chỉ trỏ tới dữ liệu ngoài, không nạp vào).

Khi chọn nguồn (VD: **Upload**, **Google Cloud Storage**, **Drive**...), cần điền các trường chính sau để tạo bảng:

| Trường (Field) | Ý nghĩa |
|---|---|
| **Source** | Nguồn dữ liệu — file tải lên máy, đường dẫn Cloud Storage (`gs://...`), hoặc Google Sheets/Drive |
| **File format** | Định dạng file nguồn: CSV, JSON, Avro, Parquet, ORC... |
| **Project** | Project chứa dataset đích |
| **Dataset** | Dataset (database) sẽ chứa bảng mới — chọn dataset **đã có sẵn** để bảng mới nhập chung vào database hiện tại |
| **Table name** | Tên bảng mới sẽ tạo |
| **Table type** | Loại bảng: **Native table** (tạo bản copy, nạp dữ liệu hẳn vào BigQuery) hoặc **External table** (chỉ trỏ tới nguồn, như đã trình bày ở mục 1–5) |
| **Schema** | Cấu trúc cột — có thể để **Auto detect** (BigQuery tự suy luận data type) hoặc tự khai báo tên cột + kiểu dữ liệu thủ công |
| **Partition and cluster settings** *(tùy chọn)* | Thiết lập phân vùng/gom cụm dữ liệu để tối ưu tốc độ truy vấn cho bảng lớn |

> 🔑 **Điểm chính:** Khác biệt cốt lõi so với mục 1–5 — `+ Add Data` **tạo bảng mới trong dataset hiện có** (nhập/join vào database sẵn có), còn khi chọn **Table type = External table**, kết quả tương đương với việc chạy câu lệnh `CREATE OR REPLACE EXTERNAL TABLE` bằng SQL đã trình bày ở mục 3 — chỉ khác là thao tác qua giao diện thay vì gõ SQL.

---

## 📌 Tóm tắt nhanh

| Câu hỏi | Trả lời |
|---|---|
| External Table lưu dữ liệu ở đâu? | Vẫn ở nguồn (Cloud Storage), **không** sao chép vào BigQuery |
| Có query được như bảng thường không? | Có — dùng `SELECT`, `JOIN` như standard table |
| Định dạng khuyến nghị? | **Parquet** (nén tốt + tự chứa schema) |
| Cách kiểm tra dữ liệu trỏ tới đâu? | Xem cột **Source URI(s)** trên BigQuery UI (tiền tố `gs://`) |
| Lợi ích chính? | Kết hợp (JOIN) dữ liệu **data lake** với dữ liệu **data warehouse** mà không cần ETL trước |
| Xem dataset/table hiện có ở đâu? | Panel **Explorer** hoặc **Classic Explorer** bên trái BigQuery Console |
| Muốn tạo bảng mới nhập vào dataset sẵn có? | Dùng **`+ Add Data`**, chọn `Table type` = Native table (nạp dữ liệu) hoặc External table (chỉ trỏ nguồn) |
