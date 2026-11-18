# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603186 - Nguyễn Hoài Nam

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày gần camera: từng xe hay `ignore_region`? | Gán từng xe nhìn thấy ≥ 20% và cao ≥ 20 px. Cụm còn lại bị che quá nửa hoặc cao < 20 px: một `ignore_region`. Với ảnh dày xe, kiểm bỏ sót theo 4 góc phần tư | img_018, img_024 | CAL-01 |
| 3 | Hàng nhô ra: box xe có gồm hàng không? Mask thì sao? | Hàng trong thùng hoặc nhô ra ≤ ~20% chiều dài xe: nằm trong box xe. Nhô hơn ~20%: box chỉ ôm thân xe. Mask xe không gồm hàng nhô ra. Ví dụ trong lô chỉ có hàng đặt trong xe, chưa có hàng nhô xa | img_023 | CAL-02 |
| 5 | Kính chắn gió thuộc mask xe không? | Chưa chốt hẳn: ở img_013 và img_023 kính trong suốt nhìn thấy người và nội thất, SAM tô lẫn với người; tạm theo 4.2.3 (kính thuộc xe) nhưng phần người nhìn thấy qua kính vẫn loại ra theo CAL-05 | img_013, img_023 | CAL-05 |
| 7 | Có đặt mức `conf` tối thiểu để nhận box không cần sửa? | Không. Máy có box điểm cao mà sai và box điểm thấp mà đúng, nên mọi box phải nhìn ảnh; `conf` chỉ để xếp thứ tự duyệt | toàn lô (Part 3) | CAL-06 |

**Chưa chốt (còn là khoảng trống của guideline):** câu 1 (xe đạp điện: lô này không có ảnh ví dụ rõ), câu 4 (ngưỡng 20% cho người trong xe có mui: chưa có phép đo lệch giữa hai người đánh giá), câu 6 (box xe máy chồng nhau trong ảnh đêm: chưa kiểm riêng ảnh đêm). Ba câu này tính vào `--guideline-gap` khi chạy `qc_report.py`; thêm câu 5 chưa chốt hẳn thì là 4.

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_013, img_018 | Xe buýt gần tuk-tuk / xe máy | YOLOE gán `three_wheeler` cho xe buýt | Đổi sang `bus` | Mục 2, cột Không gồm của `three_wheeler`: xe buýt và xe tải (≥ 4 bánh) không bao giờ là `three_wheeler`; box `three_wheeler` của máy luôn kiểm lại lớp (CAL-03) |
| img_034 | Xe tải ở xa | YOLOE gán `three_wheeler` cho xe tải | Đổi sang `truck` | Như trên (CAL-03) |
| img_032 | Xích lô, đuôi xe thấy bánh sau | YOLO26 gán `bicycle` cho đuôi hai xích lô; YOLOE sót hai xích lô bên phải | Xoá box `bicycle` ở đuôi; một `three_wheeler` ôm cả xe và mui, người là `person` riêng | Mục 2 `three_wheeler`: phần đuôi / bánh sau không phải xe đạp (CAL-04); ảnh thuộc lô pilot |
| img_013, img_010, img_023 | Mask xe ba bánh có người lái, người đạp, người đứng | SAM 2.1 tô cả người khi chỉ 1 click âm (img_013) | Mask loại người; cần ≥ 1 click âm cho mỗi người | Mục 4.2.4: thêm ví dụ xe ba bánh có người; xe ba bánh luôn cần 3 đến 6 click (CAL-05) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu 2, 3, 7; làm rõ `three_wheeler` không gồm xe ≥ 4 bánh và đuôi xích lô không phải `bicycle`; mask xe ba bánh loại người; câu 1, 4, 5, 6 ghi rõ chưa chốt | CAL-01 đến CAL-06 |
