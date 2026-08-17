# Báo cáo LAB 17 — Data Pipeline Engineering

**Họ tên:** Trần Đoàn Quang Vũ  
**Lớp:** AICB-P2T2  
**Ngày:** 17/08/2026

## 0 · Kết quả `make verify`

> Dán nguyên output `make verify` cuối cùng vào đây trước khi nộp.

make verify
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  LAB 17 · make verify
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  run 1/3 … 27.1s
  run 2/3 … 27.9s
  run 3/3 … 33.6s

  BẢNG                  ỔN ĐỊNH          SỐ HÀNG     KỲ VỌNG   GHI CHÚ
  ──────────────────────────────────────────────────────────────────────────
  gold_training_set     ✓ ok              12,480      12,480   ✓
  gold_feature_daily    ✓ ok               9,100       9,100   ✓
  gold_doc_chunks       ✓ ok              31,200      31,200   ✓
  quarantine_tickets    ✓ ok                 312         312   ✓

  CHECKSUM từng lượt
  ──────────────────────────────────────────────────────────────────────────
  gold_training_set     8dd7c98653    8dd7c98653    8dd7c98653   ✓
  gold_feature_daily    3db448685c    3db448685c    3db448685c   ✓
  gold_doc_chunks       92d8e50131    92d8e50131    92d8e50131   ✓
  quarantine_tickets    ebb89036fb    ebb89036fb    ebb89036fb   ✓

  KIỂM TRA KHÁC
  ──────────────────────────────────────────────────────────────────────────
  dbt test                                    ✓ 11/11 pass
  silver_tickets.priority ∈ 1..4, không NULL  ✓ sạch
  quarantine_tickets đúng số bản ghi lỗi      ✓ 312 / 312
  gold_training_set: 1 hàng / 1 ticket        ✓ không lặp
  dashboard rows scanned                      ✓ 5,000,000 → 9,324 (536.3×, cần ≥ 10×)
    số file parquet                           ✓ 5,000 → 14
    kết quả truy vấn không đổi                ✓
  DAG: catchup / max_active_runs              ✗ True / None

  TỔNG KẾT
  ──────────────────────────────────────────────────────────────────────────
  ✓  1 · gold_training_set idempotent & đúng số hàng
  ✓  2 · gold_feature_daily đủ hàng (dữ liệu về muộn)
  ✓  3 · contract + quarantine + dbt test
  ✓  4 · gold_doc_chunks vẫn ổn định (đối chứng)
  ──────────────────────────────────────────────────────────────────────────
  4/4 tiêu chí đạt

Tổng kết: **4 / 4 tiêu chí đạt**.

## 1 · Kích thước bảng training tăng sau mỗi lần chạy

| | |
|---|---|
| **Triệu chứng** | `gold_training_set` tăng số hàng và đổi checksum sau mỗi lần retry, dù `silver_tickets` vẫn chỉ có một hàng cho mỗi ticket. Trước khi sửa, ba lượt verify cho 38.750 hàng và ba checksum khác nhau. |
| **Nguyên nhân** | Model incremental không khai báo `unique_key`, nên dbt dùng phép ghi kiểu append. Retry cùng partition vì vậy chèn lại hàng cũ. Ngoài ra, một ticket có thể xuất hiện ở nhiều ngày do CDC `op='u'`, nên cùng ticket còn đi qua bộ lọc nhiều lần ngay trong một lượt pipeline. Phép ghi đích không idempotent đối với cả retry lẫn update. |
| **Cách khắc phục** | Trong `dbt/models/gold/gold_training_set.sql`, đặt `unique_key='ticket_id'` và `incremental_strategy='merge'`. Mỗi ticket mới được insert; ticket đã có được update. |
| **Bằng chứng** | Trước: 38.750 hàng, checksum thay đổi. Sau: 12.480 hàng, không lặp ticket; checksum ba lượt đều là `8dd7c98653`. |

## 2 · Bảng đặc trưng theo ngày thiếu hàng ở các ngày quá khứ

| | |
|---|---|
| **Triệu chứng** | `gold_feature_daily` ổn định nhưng chỉ có 8.645/9.100 hàng; thiếu 455 cặp `(event_date, customer_id)` ở các ngày cũ. |
| **P99 độ trễ đo được** | **2,724 ngày** |
| **Lookback đã chọn** | **3 ngày**, làm tròn lên từ P99 2,724 ngày; độ trễ tối đa đo được khoảng 2,945 ngày. |
| **Nguyên nhân** | Điều kiện `event_date > max(event_date)` chỉ nhận ngày mới hơn ngày lớn nhất đã có. Event xảy ra ngày cũ nhưng tới kho muộn không bao giờ được tính lại. Nếu chỉ mở rộng cửa sổ mà vẫn append, các cặp ngày–khách đã có sẽ bị chèn lại và làm bảng mất tính ổn định. |
| **Cách khắc phục** | Tính lại cửa sổ ba ngày; khai báo khóa ghép `['event_date', 'customer_id']` và dùng `merge` để thay thế kết quả aggregate cũ thay vì cộng dồn. |
| **Bằng chứng** | Trước: 8.645 hàng. Sau: 9.100 hàng; checksum ba lượt đều là `3db448685c`. |

P99 đại diện cho độ trễ vận hành thông thường nhưng vẫn bao phủ gần như toàn bộ dữ liệu; chọn ba ngày cũng bao phủ max của bộ dữ liệu hiện tại. Dùng `max` làm chính sách tổng quát dễ bị chi phối bởi một ngoại lệ rất lớn. Mỗi ngày lookback bổ sung làm tăng số partition phải đọc, aggregate và merge ở mọi lượt chạy sau, nên cửa sổ không nên rộng hơn mức cần thiết.

## 3 · Kiểu dữ liệu cột priority thay đổi giữa chu kỳ

| | |
|---|---|
| **Triệu chứng** | Pipeline không dừng nhưng `silver_tickets.priority` có 6.606 giá trị NULL/ngoài miền; quarantine rỗng và chất lượng model giảm sau khi nguồn đổi sang nhãn chữ. |
| **Nguyên nhân** | `try_cast` coi nhãn chữ hợp lệ như `urgent` là lỗi và trả NULL, nhưng lại chấp nhận các số ngoài contract như `0`, `5`, `-1`. Contract chưa được bật, test miền giá trị chưa có và quarantine dùng `where false`, nên schema drift âm thầm đi qua pipeline. |
| **Ba nhóm giá trị** | `1..4`: giữ nguyên; `urgent/high/medium/low`: ánh xạ về `1/2/3/4`; `P1/unknown/0/5/-1/rỗng/NULL`: trả NULL và đưa đúng bản ghi CDC vào quarantine. |
| **Cách khắc phục** | Dùng một macro CASE chung; lọc bản ghi không hợp lệ trước khi `row_number()`; quarantine khi macro trả NULL; bật contract và thêm `not_null` + `accepted_values [1,2,3,4]`. |
| **Bằng chứng** | Quarantine đúng 312/312 bản ghi lỗi; `priority` sạch và thuộc 1..4; Silver giữ đủ 12.480 ticket; dbt test 11/11 pass. Checksum quarantine ba lượt đều là `ebb89036fb`. |

Bronze nên giữ payload nguồn trung thực để có thể audit và tái xử lý. Chuẩn hóa, kiểm tra contract và quarantine thuộc Silver. Pipeline không nên dừng vì 312 bản ghi lỗi nếu các luồng event và transcript còn lại vẫn hợp lệ; cách ly theo từng bản ghi vừa duy trì dịch vụ vừa tạo hàng đợi rõ ràng để xử lý dữ liệu xấu.

## 4 · Bài mở rộng

### Bài A — Dashboard và small-file problem

| | |
|---|---|
| **Nguyên nhân** | Dataset gồm 5.000 file Parquet nhỏ, path không mang thông tin của filter và `strftime(event_time, ...)` bọc cột trong hàm, làm predicate không sargable. DuckDB phải mở/quét quá nhiều file và không tận dụng tốt partition pruning hay min/max. |
| **Cách khắc phục** | Compact sang dataset partition theo `event_date`, sắp theo `customer_name, event_time`, row group 10.000; query bật Hive partitioning và lọc `event_date = DATE '2026-08-09'`. |
| **Bằng chứng** | Rows scanned giảm từ 5.000.000 xuống 9.324 (536,3×); file giảm từ 5.000 xuống 14; rows on disk giữ nguyên 130.683; result hash giữ nguyên `4379e4c5d9f3`; thời gian tham khảo 8,8 ms. |

make explain
  queries/dashboard.sql
  --------------------------------------------------------------
                             TRƯỚC        HIỆN TẠI      MỤC TIÊU
  rows scanned           5,000,000           9,324     ≤ 500,000   ✓
  rows on disk             130,683         130,683   (tham khảo)
  files                      5,000              14        ít hơn   ✓
  result hash         4379e4c5d9f3    4379e4c5d9f3     không đổi   ✓
  thời gian (ms)                 —             8.8   (tham khảo)

  => giảm 536.3× (cần ≥ 10×)

  kết quả truy vấn (1 hàng):
    ('ACME', 3500, 3068, 2521.1, 4691, 262, 7764750)

### Bài B — Consumer crash giữa batch

| | |
|---|---|
| **Nguyên nhân** | Commit offset trước khi ghi tạo at-most-once: crash sau commit làm restart bỏ qua batch chưa ghi. Đảo thành ghi trước–commit sau tạo at-least-once; khi crash, batch được replay nên INSERT thuần lại gây trùng. |
| **Cách khắc phục** | Ghi batch trong transaction, sau đó mới commit offset; đặt `event_id` làm primary key và dùng `ON CONFLICT DO UPDATE`. Đây là at-least-once kết hợp phép ghi idempotent. `DO UPDATE` giữ được phiên bản nội dung mới khi message cùng khóa thay đổi, còn `DO NOTHING` sẽ giữ dữ liệu cũ. |
| **Bằng chứng** | A = 20.000 hàng/20.000 event ID. Crash ở batch 7 với offset 3.000; restart xử lý 17.000 message. Kết quả C vẫn là 20.000 hàng/20.000 event ID: không mất, không trùng và C = A. |

make crash-test

  topic: 20,000 message · batch 500 · giết ở lô 7

  A. chạy một mạch, không sự cố
  [consumer] đã ghi 20,000 message
     -> 20,000 hàng / 20,000 event_id khác nhau

  B. chạy và bị giết ở lô 7
  [consumer] 💥 tiến trình bị giết ở lô 7
     -> tiến trình thoát với mã 137
     -> offset đã commit: 3,000

  C. khởi động lại, chạy nốt
  [consumer] đã ghi 17,000 message
     -> 20,000 hàng / 20,000 event_id khác nhau

  ----------------------------------------------------------
  không mất bản ghi                 ✓
  không trùng bản ghi               ✓
  C == A                            ✓
  ----------------------------------------------------------
  BÀI MỞ RỘNG B: ĐẠT ✓

## 5 · Tổng kết

| Nhiệm vụ | Khi tiếp nhận một hệ thống chưa quen, tôi sẽ kiểm tra điều này trước tiên |
|---|---|
| 1 | Xác định grain, khóa tự nhiên và hành vi ghi khi retry hoặc replay. |
| 2 | Đo phân bố độ trễ event-time/ingestion-time trước khi chọn incremental window. |
| 3 | So sánh miền giá trị nguồn với contract, phân biệt schema evolution với dữ liệu hỏng và kiểm tra thứ tự lọc–xếp hạng. |

## Bảng tự chấm trước khi nộp

| Hạng mục | Trạng thái hiện tại | Điểm tối đa |
|---|---:|---:|
| A · Checksum ổn định ba lượt | Đạt | 30/30 |
| B · Số hàng khớp expected | Đạt | 30/30 |
| C · Contract, test, quarantine | Đạt | 20/20 |
| D · Báo cáo nêu nguyên nhân | Đạt | 20/20 |
| Thưởng A | Đạt: giảm 536,3×, hash không đổi | +5/5 |
| Thưởng B | Đạt: không mất, không trùng, C = A | +5/5 |

**Tự chấm: 110/100.**
