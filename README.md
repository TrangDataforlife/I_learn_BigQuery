# I_learn_BigQuery
# 🌉 BigQuery External Table — Kết nối Data Lake và Data Warehouse

## 📑 Mục lục

- [1. External Table là gì?](#1-external-table-là-gì)
- [2. Vì sao thường dùng định dạng Parquet?](#2-vì-sao-thường-dùng-định-dạng-parquet)
- [3. Cách tạo External Table trong BigQuery](#3-cách-tạo-external-table-trong-bigquery)
- [4. Kiểm tra nguồn dữ liệu trên giao diện BigQuery](#4-kiểm-tra-nguồn-dữ-liệu-trên-giao-diện-bigquery)
- [5. Ứng dụng thực tế: Kết hợp Data Lake và Data Warehouse](#5-ứng-dụng-thực-tế-kết-hợp-data-lake-và-data-warehouse)
- [6. Explorer & Classic Explorer — Kiểm tra Dataset và thêm dữ liệu](#6-explorer--classic-explorer--kiểm-tra-dataset-và-thêm-dữ-liệu)
  - [6.1. Thêm dữ liệu bằng "+ Add Data" (source từ Google Cloud)](#61-thêm-dữ-liệu-bằng--add-data-source-từ-google-cloud)
  - [6.2. Thêm dữ liệu bằng query (source từ Google Cloud)](#62-thêm-dữ-liệu-bằng-query-source-từ-google-cloud)
  - [6.3. Thêm dữ liệu bằng query CTAS statement (source từ chính kết quả được tạo ra từ query)](#63-thêm-dữ-liệu-bằng-query-ctas-statement-source-từ-chính-kết-quả-được-tạo-ra-từ-query)
- [📌 Tóm tắt nhanh](#-tóm-tắt-nhanh)
- [7. Partitioning tables](#7-partitioning-tables)
- [8. Partition Pruning trong BigQuery](#8-partition-pruning-trong-bigquery)

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
Vị trí tạo query: + query editor tại datawarehouse (nơi cần kết nối với external table)

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

### 6.1. Thêm dữ liệu bằng "+ Add Data" (source từ Google Cloud)

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
### 6.2. Thêm dữ liệu bằng query (source từ Google Cloud)
Thêm state_region table trong fintech schema

```sql
LOAD DATA OVERWRITE fintech.state_region
(
state string,
subregion string,
region string
)
FROM FILES (
format = 'CSV',
uris = ['gs://sureskills-lab-dev/future-workforce/da-capstone/temp_35_us/state_region_mapping/state_region_*.csv']);
```

### 6.3. Thêm dữ liệu bằng query CTAS statement (source từ chính kết quả được tạo ra từ query)
Tạo bảng loan_with_region bên trong fintech schema, 
A CTAS statement, or CREATE TABLE AS SELECT statement, is a SQL statement that creates a new table based on the results of a SELECT statement. It is a powerful tool that can be used to create new tables quickly and easily. Tables made with CTAS statements can also be exported easily in BigQuery so that they can be shared with others.

```sql
CREATE OR REPLACE TABLE fintech.loan_with_region AS
SELECT
lo.loan_id,
lo.loan_amount,
sr.region
FROM fintech.loan lo
INNER JOIN fintech.state_region sr
ON lo.state = sr.state;
```

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

## 7. Partitioning tables
## 📑 Mục lục

- [7.1. Ba cách Partition Table](#1-ba-cách-partition-table)
- [7.2. Clustered Table & Clustered Columns](#2-clustered-table--clustered-columns)
- [7.3. Ví dụ minh họa: Bảng `orders`](#3-ví-dụ-minh-họa-bảng-orders)
  - [7.3.1. Dữ liệu gốc](#31-dữ-liệu-gốc)
  - [7.3.2. Sau khi Partition theo `order_date`](#32-sau-khi-partition-theo-order_date)
  - [7.3.3. Trong mỗi Partition, tiếp tục Cluster](#33-trong-mỗi-partition-tiếp-tục-cluster)
  - [7.3.4. So sánh hiệu quả quét dữ liệu](#34-so-sánh-hiệu-quả-quét-dữ-liệu)
  - [7.3.5. Vì sao chọn 2 cột này để minh họa](#35-vì-sao-chọn-2-cột-này-để-minh-họa)

---

## 7.1. Ba cách Partition Table

- **Integer range partitioning** — phân vùng dựa trên một cột **số nguyên** (VD: `customer_id`), chia theo các khoảng giá trị (start, end, interval) do bạn tự định nghĩa.
- **Time-unit column partitioning** — phân vùng dựa trên một cột **kiểu ngày/giờ** có sẵn trong bảng (VD: cột `order_date`), chia theo ngày/tháng/năm/giờ.
- **Ingestion time partitioning** — phân vùng tự động theo **thời điểm dữ liệu được nạp (load)** vào bảng, dùng cột ẩn `_PARTITIONTIME` do BigQuery tự sinh ra (không cần bảng có sẵn cột ngày/giờ).

---

## 7.2. Clustered Table & Clustered Columns

- **Clustered table** — bảng có dữ liệu được **sắp xếp/gom nhóm vật lý** theo giá trị của 1 hoặc nhiều cột (tối đa 4 cột), gọi là **clustered columns**.
- **Clustered columns** — các cột dùng để sắp xếp dữ liệu bên trong mỗi khối lưu trữ (block), thường chọn cột hay dùng trong `WHERE`, `GROUP BY`, `JOIN` (VD: `customer_id`, `product_category`).

**Liên quan gì đến Partition?**

- **Partition** = chia bảng thành các **khối lớn** (theo ngày/số nguyên); **Cluster** = trong mỗi phân vùng, tiếp tục **sắp xếp nhỏ hơn** theo cột chỉ định.
- Thường **dùng kết hợp**: Partition trước (lọc thô theo ngày) → Cluster sau (lọc tinh trong từng partition) → tăng tốc truy vấn tối đa.
- Khác partition: Cluster **không tạo phân vùng riêng biệt**, chỉ sắp xếp thứ tự lưu trữ dữ liệu.

**Khi nào dùng?**

- Cột có **quá nhiều giá trị khác nhau (high cardinality)** → không phù hợp làm partition, nhưng phù hợp làm **cluster column** (VD: `customer_id`, `order_id`).
- Truy vấn thường xuyên **lọc (`WHERE`) hoặc gộp (`GROUP BY`/`JOIN`)** theo cột đó.
- Bảng đã partition nhưng vẫn quét nhiều dữ liệu trong 1 partition → thêm cluster để thu hẹp thêm.

**Mục tiêu**

- **Giảm dữ liệu quét (bytes scanned)** khi query → **giảm chi phí** và **tăng tốc độ** truy vấn.
- Không cần khai báo range như partition — BigQuery **tự động** quản lý việc sắp xếp/gom nhóm.

---

## 7.3. Ví dụ minh họa: Bảng `orders`

Partition theo ngày + Cluster theo 2 cột.

### 7.3.1. Dữ liệu gốc

| order_id | order_date | customer_id | product_category | amount |
|---|---|---|---|---|
| 101 | 2026-07-01 | C001 | Electronics | 500 |
| 102 | 2026-07-01 | C045 | Clothing | 80 |
| 103 | 2026-07-01 | C001 | Electronics | 300 |
| 104 | 2026-07-02 | C089 | Clothing | 60 |
| 105 | 2026-07-02 | C001 | Books | 25 |
| 106 | 2026-07-02 | C045 | Electronics | 700 |

-------Có 2 cách tạo partition & cluster table:
a/ Explorer tab để create table: di chuyển đến partition & cluster field (đảm bảo có column phù hợp với partition strategy)


b/ classic explorer tab -> query editor (Create table as select)
Đây là cú pháp CTAS — "Create Table As Select": tạo bảng mới và nạp dữ liệu ngay lập tức từ một bảng đã có sẵn, chỉ trong 1 câu lệnh duy nhất.
```sql
CREATE TABLE `shop.orders`
PARTITION BY DATE(order_date)             -- Partition theo ngày
CLUSTER BY customer_id, product_category  -- Cluster theo 2 cột
AS SELECT * FROM `shop.orders_raw`;
```

CREATE TABLE \shop.orders` ...` chỉ định nghĩa cấu trúc bảng mới (kèm partition + cluster) — nếu dừng lại ở đó, bảng sẽ rỗng, không có dữ liệu.
AS SELECT * FROM \shop.orders_raw` mới là phần **lấy dữ liệu** để đổ vào bảng mới đó — lấy toàn bộ (*) dữ liệu từ bảng gốc orders_raw` (bảng chưa partition/cluster).

### 7.3.2. Sau khi Partition theo `order_date`

BigQuery **tách vật lý** dữ liệu thành từng "khối" riêng theo ngày:

```
📁 Partition: 2026-07-01              📁 Partition: 2026-07-02
   101 | C001 | Electronics | 500        104 | C089 | Clothing    | 60
   102 | C045 | Clothing    | 80         105 | C001 | Books       | 25
   103 | C001 | Electronics | 300        106 | C045 | Electronics | 700
```

> 💡 **Vì sao:** Query `WHERE order_date = '2026-07-02'` → BigQuery **chỉ đọc partition ngày 02**, bỏ qua hoàn toàn partition ngày 01 → giảm bytes scanned ngay từ bước lọc thô.

### 7.3.3. Trong mỗi Partition, tiếp tục Cluster

Cluster theo `customer_id`, `product_category`:

```
📁 Partition: 2026-07-01 (đã sắp xếp theo customer_id → product_category)
   101 | C001 | Electronics | 500   ┐
   103 | C001 | Electronics | 300   ┘ gom cạnh nhau vì cùng C001 + Electronics
   102 | C045 | Clothing    | 80
```

> 💡 **Vì sao:** Query thêm `AND customer_id = 'C001'` → trong partition 01, dữ liệu của `C001` đã được **xếp liền kề nhau** → BigQuery đọc đúng vùng đó, **bỏ qua** dòng của `C045` mà không cần quét toàn bộ partition.

### 7.3.4. So sánh hiệu quả quét dữ liệu

| Query | Không partition/cluster | Có partition + cluster |
|---|---|---|
| `WHERE order_date='2026-07-02' AND customer_id='C001'` | Quét cả 6 dòng | Chỉ quét 1 dòng (`105`) — đúng partition 02, đúng cụm `C001` |

### 7.3.5. Vì sao chọn 2 cột này để minh họa

- `order_date` → làm **partition**: hợp lý vì hầu hết query BI đều lọc theo khoảng ngày (`WHERE order_date BETWEEN ...`), và cột date/timestamp là ứng viên chuẩn cho time-unit partitioning.
- `customer_id`, `product_category` → làm **cluster**: `customer_id` có **cardinality cao** (rất nhiều giá trị khác nhau) nên không thể partition (partition giới hạn số lượng), nhưng lại **hay xuất hiện trong `WHERE`/`JOIN`** → phù hợp làm cluster column để BigQuery sắp xếp gọn, đọc nhanh mà không cần tạo hàng nghìn partition nhỏ lẻ.

### 8. Partition Pruning trong BigQuery

## 📑 Mục lục

- [8.1. Partition Pruning là gì?](#1-partition-pruning-là-gì)
- [8.2. Query theo từng loại Partition](#2-query-theo-từng-loại-partition)
  - [8.2.1. Time-unit column partitioning](#21-time-unit-column-partitioning)
  - [8.2.2. Ingestion-time partitioning](#22-ingestion-time-partitioning)
  - [8.2.3. Integer-range partitioning](#23-integer-range-partitioning)
- [8.3. Best Practices để Pruning hoạt động đúng](#3-best-practices-để-pruning-hoạt-động-đúng)
  - [8.3.1. Dùng biểu thức lọc dạng hằng số (constant)](#31-dùng-biểu-thức-lọc-dạng-hằng-số-constant)
  - [8.3.2. Tách riêng cột partition / chỉ dùng hàm được hỗ trợ](#32-tách-riêng-cột-partition--chỉ-dùng-hàm-được-hỗ-trợ)
  - [8.3.3. Lọc thêm nhiều cột khác (kết hợp AND)](#33-lọc-thêm-nhiều-cột-khác-kết-hợp-and)
  - [8.3.4. Bắt buộc filter partition (Require partition filter)](#34-bắt-buộc-filter-partition-require-partition-filter)
- [📌 Tóm tắt nhanh](#-tóm-tắt-nhanh)

---

## 8.1. Partition Pruning là gì?

**Partition pruning** là cơ chế BigQuery dùng để **loại bỏ (skip)** các partition không liên quan khỏi phạm vi quét, khi query có điều kiện lọc (`WHERE`) đúng trên cột partition.

- Các partition bị loại bỏ **không được tính vào bytes scanned** → **giảm chi phí** query.
- Cách pruning hoạt động **khác nhau tùy loại partition** (time-unit, ingestion-time, integer-range) → cùng 1 lượng dữ liệu nhưng chọn loại partition khác nhau có thể tốn bytes xử lý khác nhau.
- Muốn biết chính xác query sẽ quét bao nhiêu bytes trước khi chạy thật → dùng **dry run**.

---

## 8.2. Query theo từng loại Partition

### 8.2.1. Time-unit column partitioning

Lọc trực tiếp trên cột đã dùng làm partition (VD: cột `transaction_date`):

```sql
SELECT * FROM dataset.table
WHERE transaction_date >= '2016-01-01'
```

### 8.2.2. Ingestion-time partitioning

Loại partition này tạo ra 2 **pseudocolumn** (cột ẩn, không phải cột thật trong schema):

| Pseudocolumn | Ý nghĩa |
|---|---|
| `_PARTITIONTIME` | Thời điểm nạp dữ liệu (UTC), làm tròn theo cấp partition (giờ/ngày/tháng/năm), kiểu `TIMESTAMP` |
| `_PARTITIONDATE` | Chỉ có ở bảng partition theo **ngày** — bằng `_PARTITIONTIME` làm tròn về kiểu `DATE` |

```sql
-- Lọc theo khoảng thời gian trên _PARTITIONTIME
SELECT column
FROM dataset.table
WHERE _PARTITIONTIME BETWEEN TIMESTAMP('2016-01-01') AND TIMESTAMP('2016-01-02')
```

> ⚠️ **Lưu ý:**
> - `SELECT *` **không** tự động trả về `_PARTITIONTIME`/`_PARTITIONDATE` — phải `SELECT` tường minh và đặt alias: `SELECT _PARTITIONTIME AS pt, *`.
> - Dữ liệu đang ở **write-optimized storage** (mới stream vào, chưa commit hẳn vào partition) có `_PARTITIONTIME IS NULL` — muốn query riêng phần này thì lọc `WHERE _PARTITIONTIME IS NULL`.
> - Nên đặt `_PARTITIONTIME` **một mình** ở một vế của phép so sánh để tối ưu hiệu năng, thay vì bọc nó trong biểu thức cộng/trừ phức tạp.

### 8.2.3. Integer-range partitioning

Lọc trực tiếp trên cột số nguyên đã dùng làm partition:

```sql
-- Bảng partition theo customer_id:0:100:10 (start:end:interval)
SELECT * FROM dataset.table
WHERE customer_id BETWEEN 30 AND 50
-- Chỉ quét 3 partition bắt đầu từ 30, 40, 50
```

> ⚠️ Pruning **không hoạt động** nếu áp dụng phép tính lên cột partition:
> ```sql
> -- Quét toàn bộ bảng, KHÔNG pruning
> WHERE customer_id + 1 BETWEEN 30 AND 50
> ```

---

## 8.3. Best Practices để Pruning hoạt động đúng

### 8.3.1. Dùng biểu thức lọc dạng hằng số (constant)

| Loại biểu thức | Có pruning không? | Ví dụ |
|---|---|---|
| **Hằng số (constant)** | ✅ Có | `WHERE t1.ts = CURRENT_TIMESTAMP()` |
| **Giá trị động (subquery, cột khác)** | ❌ Không | `WHERE t1.ts = (SELECT timestamp FROM table3 WHERE key=2)` |
| **So sánh với cột khác trong cùng bảng** | ❌ Không | `WHERE ts >= ts2` |

### 8.3.2. Tách riêng cột partition / chỉ dùng hàm được hỗ trợ

Muốn pruning hoạt động, cột partition phải **đứng riêng một mình** ở một vế so sánh, hoặc chỉ được bọc bởi các **hàm được BigQuery hỗ trợ** (tham số khác phải là hằng số):

`DATE_ADD`, `DATE_SUB`, `DATE_DIFF`, `DATE_TRUNC`, `EXTRACT(YEAR/DATE)`, `DATETIME_DIFF`, `TIMESTAMP_ADD`, `TIMESTAMP_SUB`, `TIMESTAMP_DIFF`, `TIMESTAMP_TRUNC`, `FORMAT_TIMESTAMP` (với vài format cụ thể).

```sql
-- ❌ Chậm hơn: hàm bọc quanh cột nhưng nằm sai vế
WHERE TIMESTAMP_ADD(_PARTITIONTIME, INTERVAL 5 DAY) > TIMESTAMP('2016-04-15')

-- ✅ Nhanh hơn: cột partition đứng riêng một vế
WHERE _PARTITIONTIME > TIMESTAMP_SUB(TIMESTAMP('2016-04-15'), INTERVAL 5 DAY)
```

```sql
-- ❌ Không pruning: cộng cột partition với 1 cột khác trong bảng
WHERE _PARTITIONTIME + field1 = TIMESTAMP('2016-03-28')

-- ❌ Không pruning: hàm FORMAT_DATE và phép cộng thời gian không được hỗ trợ
WHERE FORMAT_DATE('%Y-%m-%d %H', ts) = '2025-03-28 20'
WHERE ts + INTERVAL 1 DAY > CURRENT_TIMESTAMP()

-- ✅ Viết lại để pruning hoạt động — tách cột partition ra khỏi phép tính
WHERE ts >= '2025-03-28 20:00:00' AND ts < '2025-03-28 21:00:00'
WHERE ts > CURRENT_TIMESTAMP() - INTERVAL 1 DAY
```

> 💡 Nếu hay query lặp lại 1 khoảng thời gian cố định, có thể tạo sẵn **view** lọc theo `_PARTITIONTIME` để tận dụng pruning mỗi lần dùng:
> ```sql
> CREATE VIEW dataset.past_week AS
>   SELECT * FROM dataset.partitioned_table
>   WHERE _PARTITIONTIME BETWEEN
>     TIMESTAMP_TRUNC(TIMESTAMP_SUB(CURRENT_TIMESTAMP, INTERVAL 7*24 HOUR), DAY)
>     AND TIMESTAMP_TRUNC(CURRENT_TIMESTAMP, DAY);
> ```

### 8.3.3. Lọc thêm nhiều cột khác (kết hợp AND)

Có thể thêm điều kiện lọc trên **các cột khác** cùng lúc với cột partition — pruning vẫn hoạt động, miễn điều kiện trên cột partition đúng chuẩn ở mục 3.2, và các điều kiện được nối bằng **`AND`** (không phải `OR`):

```sql
-- ✅ Pruning vẫn hoạt động (dù thứ tự predicate không quan trọng)
WHERE meter_id = 1234
  AND ts >= '2025-03-28 20:00:00' AND ts < '2025-03-28 21:00:00'
```

> ⚠️ Nếu đổi `AND` thành `OR` → pruning **mất tác dụng**, vì một partition dù không khớp điều kiện cột partition vẫn có thể chứa dòng thỏa `meter_id = 1234` → buộc phải quét hết.

### 8.3.4. Bắt buộc filter partition (Require partition filter)

Khi tạo bảng, có thể bật tùy chọn **"Require partition filter"** để bắt buộc mọi query phải có `WHERE` lọc theo cột partition — nếu thiếu, BigQuery báo lỗi ngay thay vì âm thầm quét toàn bảng (tốn chi phí).

```sql
-- ✅ Hợp lệ — có điều kiện lọc thuần trên cột partition
WHERE partition_id = "20221231"
WHERE partition_id = "20221231" AND f = "20221130"

-- ❌ Không hợp lệ — điều kiện trên cột partition bị nối bằng OR
WHERE partition_id = "20221231" OR f = "20221130"
```

Với bảng ingestion-time partitioning, dùng `_PARTITIONTIME` hoặc `_PARTITIONDATE` để thỏa yêu cầu này.

---

## 📌 Tóm tắt nhanh

| Câu hỏi | Trả lời |
|---|---|
| Partition pruning là gì? | Cơ chế bỏ qua các partition không liên quan, giảm bytes quét → giảm chi phí query |
| Áp dụng cho loại partition nào? | Cả 3 loại: time-unit, ingestion-time, integer-range — nhưng cú pháp lọc khác nhau |
| Điều kiện để pruning hoạt động? | Cột partition đứng riêng 1 vế so sánh, dùng hằng số hoặc hàm được hỗ trợ, nối `AND` (không `OR`) |
| Cách đảm bảo mọi query đều pruning? | Bật **Require partition filter** khi tạo bảng |
| Cách kiểm tra trước khi chạy thật? | Dùng **Dry run** để xem số bytes sẽ quét |

