# REPORT — Lab Day 9

| | |
|---|---|
| Họ và tên | Lê Nguyễn Trâm Anh |
| MSSV (đúng bằng `LAB_STUDENT_ID` trong `.env`) | `2A202602760` |
| GitHub | `itskathy05` |
| Trip dùng cho bài tập sập và chạy lại (xem GUIDE, bước 5) | `trip-08448` |

## Kết quả cuối

- `dataset_version` của bạn: `b83bf1f34f4b`
- Số khung hình train / val / test của bạn: `687 / 175 / 157` (tổng 1019; 18 / 4 / 4 trip).
- `report/verify.txt`: verify cuối sau retry đạt `27/27 checks passed`.
- Ảnh bằng chứng đã có: `airflow_graph.jpg`, `airflow_failure_log.jpg`, `airflow_retry_green.png`, `minio_lake.png`, `metabase_dashboard.png`.

## Bước 1 · Bronze

### Câu 1

Nguồn A dùng `ts_utc` dạng ISO UTC, `speed_mps`, `heading_deg`, `lat/lon`; nguồn B dùng `timestamp_ms` epoch millisecond, `spd_kmh`, `hdg`, `latitude/longitude`.
A ghi pedal `gas` và `brake` theo tỷ lệ; B ghi `throttle_pct` và `brake_pct` theo phần trăm. B còn có đồng hồ chạy nhanh hơn đồng hồ video 1,5 giây.
Giữ bảng bronze tách riêng và giữ giá trị thô giúp truy vết nguồn, kiểm tra schema và chạy lại chuẩn hoá khi phát hiện quy tắc sai mà không làm mất dữ liệu ban đầu.

## Bước 2 · Silver telemetry

### Câu 2

Nguồn A đã có tốc độ m/s và timestamp ISO nên được ép kiểu về `timestamptz`; nguồn B đổi km/h sang m/s, epoch ms sang UTC và trừ lệch đồng hồ 1500 ms.
Thời điểm đầu B sau hiệu chỉnh là `2025-10-08 14:30:00.009 UTC`, khớp đầu video trong 0,5 giây; A bắt đầu `2025-10-07 08:00:00.014 UTC`.
Mỗi drive có 20 duplicate, 30 `speed_missing`, 3 `speed_out_of_range`; tổng 53 dòng bị cách ly mỗi drive. Dòng sạch: A 2968/3021, B 3181/3234.
Quarantine giữ dấu vết và lý do loại để kiểm toán hoặc sửa quy tắc, thay vì âm thầm xoá dữ liệu có thể cần điều tra.

### Câu 3

`max_gap_ms` là `warn` vì khoảng trống cảm biến cần được nhìn thấy nhưng không nhất thiết làm dừng pipeline; `in_range` là `error` vì tốc độ ngoài miền hợp lệ làm hỏng dữ liệu downstream.
Trong `break_gate.json`, nới giới hạn lên 1000 m/s khiến test `in_range_silver_telemetry_speed_mps__70__0` bắt được 6 dòng và cổng chuyển `ERROR=1`.
Khôi phục luật gốc cho kết quả `PASS=6 WARN=1 ERROR=0`. Cổng chất lượng chặn dữ liệu sai nghiêm trọng nhưng cho phép pipeline tiếp tục khi chỉ có cảnh báo về độ phủ.

## Bước 3 · Silver frames

### Câu 4

`idempotency.json` ghi 1246 dòng trước và sau hai lần chạy lại, fingerprint vẫn `6c7480dfde7e`; riêng trip thử có 60 frame.
Biến thể append hai lần tạo 120 dòng cho trip đó, tức nhân đôi dữ liệu. Overwrite theo predicate `trip_id` thay thế đúng partition và không cộng thêm bản sao.
Ghi theo partition giúp chạy lại sau lỗi giữa chừng mà không phải xoá toàn bảng; Delta commit nguyên tử giúp trạng thái đọc được nhất quán.

### Câu 5

Khung đen gần như không có thông tin hình ảnh nên bị loại; khung tối có thể là cảnh đêm, đường hầm hoặc bóng râm nên được gắn cờ và vẫn giữ nếu đạt các tiêu chí khác.
Dữ liệu của tôi có 6 khung đen ở A và 4 ở B; 61 khung tối được giữ ở B. Ngưỡng dHash 0 / 2 / 6 đánh dấu trùng: A `61 / 149 / 363`, B `8 / 53 / 244`.
Tôi dùng ngưỡng 2 như cấu hình lab: ngưỡng cao loại nhiều hơn nhưng có thể nhầm cảnh gần giống; xe chạy chậm hoặc camera đứng hình tạo nhiều frame gần trùng hơn xe chạy nhanh.

| dHash khác tối đa | số khung trùng, drive A | số khung trùng, drive B |
|---|---:|---:|
| 0 | 61 | 8 |
| 2 | 149 | 53 |
| 6 | 363 | 244 |

## Bước 4 · Gold

### Câu 6

`leakage.json` có 1019 frame thuộc 26 trip. Chia ngẫu nhiên theo frame làm cả 26 trip xuất hiện ở nhiều split và cả 26 trip có frame ở cả train lẫn test.
Chia theo trip cho 0 trip bị tách. Các frame liền nhau trong cùng cảnh thường rất giống nhau, nên để một số vào train và số khác vào test làm mô hình gần như đã thấy cảnh kiểm thử.
Vì vậy điểm test theo frame có thể đẹp giả tạo và không phản ánh khả năng tổng quát hoá sang chuyến đi mới.

### Câu 7

`dataset_version` là 12 ký tự đầu của SHA256 tính từ danh sách frame ID và split, nên nhận diện nội dung dataset thay vì thời điểm chạy hay số lần chạy.
Xuất lại cùng nội dung cho cùng version `b83bf1f34f4b`; đổi MSSV làm thay đổi thứ tự xếp trip vào split, do đó hash và version thay đổi.
Manifest hiện tại ghi 1019 frame: train 687, val 175, test 157; 18 / 4 / 4 trip tương ứng.

### Câu 8

Manifest `gold/datasets/b83bf1f34f4b/manifest.json` ghi lineage: `bronze.raw_telemetry_a=0`, `bronze.raw_telemetry_b=0`, `bronze.trips_raw=0`, `bronze.video_files=1`.
Các bảng silver là `silver.frames=25`, `silver.telemetry=0`, `silver.frames_flagged=0`, `silver.frame_telemetry=0`.
Video A có SHA256 `636ba1a98dfc5ba665d1c935e1d2064e69f4df360ff9ffbd15795957d5e7e9aa`; video B có `bfa62703f645ff77a8578ae3fc74a00881140d38a404d43c8e2af265bbee9fdc`.
Các phiên bản Delta, hash video, quy tắc loại frame và quy tắc split trong manifest cho phép đối chiếu nguồn và tái tạo dataset về sau.

## Bước 5 · Điều phối và phục vụ

### Câu 9

Lần chạy `07:55 UTC` có đúng một mapped task `silver_frames` đỏ: map index 22, trip `trip-08448`; bảy task phía sau chuyển `upstream_failed`.
Task lỗi được clear riêng, lần thử thứ hai xanh. Vì các task downstream vẫn cam sau đó, tôi clear `dbt_run_silver` để chạy tiếp chuỗi downstream; cuối cùng toàn bộ DAG xanh.
Manifest sau orchestration vẫn version `b83bf1f34f4b` và 1019 frame, không nhân đôi partition. Verify cuối sau retry đạt `27/27 checks passed` và đã lưu trong `report/verify.txt`.

### Câu 10

Ở quy mô 500 xe/ngày, bước trích frame sẽ nghẽn trước vì mỗi trip chạy tuần tự (`max_active_tis_per_dagrun=1`); tôi sẽ dùng worker pool phân tán, giới hạn song song và hàng đợi có backpressure.
Tiếp theo, nạp file và MinIO đơn máy sẽ thành điểm nghẽn; thay bằng object storage phân tán, ingestion theo sự kiện và partition theo ngày/xe.
DuckDB và Postgres đơn node cũng không phù hợp cho nhiều consumer/khối lượng lớn; chuyển xử lý sang Spark hoặc engine lakehouse phân tán và phục vụ analytics bằng warehouse mở rộng ngang, như nội dung slide 12, 22, 23.

## Câu hỏi riêng của bạn trên Metabase

```sql
select drive_id, trip_id, frames, frames_kept,
       round(100.0 * (frames - frames_kept) / nullif(frames, 0), 1) as drop_pct,
       avg_speed_kmh, lighting, conditions
from marts.mart_trip_summary
order by drop_pct desc, drive_id, trip_id
limit 10;
```

Kết quả truy vấn warehouse xếp `drive_a / trip-04622` cao nhất: 26/60 frame được giữ, tỷ lệ loại 56,7%. Tôi đã lưu câu hỏi `Top trips by frame drop rate` vào dashboard Metabase để nhóm vận hành ưu tiên rà soát các trip có tỷ lệ loại frame cao.

## Câu hỏi Metabase bonus thứ hai

Câu hỏi `Frame retention by drive and conditions` nhóm số trip, tổng frame, frame được giữ và tỷ lệ loại theo drive/điều kiện để nhận ra nguồn cần rà soát. Tôi đã lưu câu hỏi này trên dashboard. Kết quả: drive_a có 602 frame, giữ 436 (loại 27,6%), tốc độ trung bình 35,7 km/h; drive_b có 644 frame, giữ 583 (loại 9,5%), tốc độ trung bình 67,6 km/h. Tỷ lệ loại của A cao hơn rõ rệt, nên cần kiểm tra các quy tắc chất lượng/video của A trước khi gộp dữ liệu.

SQL:

```sql
select drive_id, lighting, conditions, count(*) as trips,
       sum(frames) as frames, sum(frames_kept) as frames_kept,
       round(100.0 * sum(frames - frames_kept) / nullif(sum(frames), 0), 1) as drop_pct,
       round(avg(avg_speed_kmh)::numeric, 1) as avg_speed_kmh
from marts.mart_trip_summary
group by drive_id, lighting, conditions
order by drop_pct desc, trips desc
limit 20;
```

## Thí nghiệm bonus · Sync tolerance

Dùng ngưỡng mặc định 150 ms làm baseline: 7 frame mất đồng bộ mỗi drive và 1019 Gold frame. Tôi áp dụng đúng predicate `sync_gap_ms <= 50` của model lên snapshot Delta `silver.frame_telemetry` đã lưu (1246 frame), cùng các bộ lọc Gold.
Ở 50 ms, frame mất đồng bộ tăng thành A 272, B 328; Gold còn A 242, B 296, tổng 538. Như vậy giảm 481 frame, tức 47,2% so với baseline.
Ngưỡng thấp đòi hỏi ghép thời gian chặt hơn nhưng làm giảm mạnh lượng dữ liệu huấn luyện. Phép tính dùng snapshot và predicate thực tế; model cuối vẫn ở cấu hình 150 ms.

## Nhìn lại

Tôi bất ngờ khi chia theo frame làm cả 26 trip cùng rò rỉ train-test, trong khi chia theo trip loại bỏ hoàn toàn hiện tượng đó. Ngưỡng đồng bộ cũng ảnh hưởng mạnh đến số frame Gold: giảm từ 150 xuống 50 ms làm mất gần một nửa dữ liệu.
Một lỗi pipeline có thể chưa phát hiện là tọa độ GPS hợp lệ về kiểu và miền nhưng nhảy vị trí bất khả thi giữa hai mẫu; cần thêm kiểm tra vận tốc suy ra từ khoảng cách GPS và thời gian.