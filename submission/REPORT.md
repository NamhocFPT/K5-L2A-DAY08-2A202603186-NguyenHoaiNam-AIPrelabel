# Báo cáo Day 08 — 2A202603186 - Nguyễn Hoài Nam

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
| --- | --- | --- | --- |
| person / bicycle / motorbike / car / bus / truck | YOLO26n (máy chỉ có CPU nên dùng cỡ n, `--imgsz 640`) | Mặc định 0.3 cho cả 6 lớp COCO: không lớp nào đạt recall 90% ở ngưỡng ≤ 0.5 (person tối đa 69%, motorbike 75%, car 88%, bicycle 0%); bus có 1 vật và truck có 0 vật trong gold nên không chọn riêng (`qc/threshold_sweep.md`) | r01, r02: nhận đủ 6 lớp COCO, 0.5 giây/ảnh trên CPU; nâng ngưỡng 0.25 lên 0.5 giảm 120 xuống 54 box nhưng mất vật bị che, bị cắt mép ảnh và vật nhỏ ở xa (chi tiết trong `model_log.csv`) |
| three_wheeler | YOLOE-26s, `--classes three_wheeler`, prompt mặc định từ `taxonomy.json`: tuk-tuk, cyclo, cargo tricycle | Mặc định 0.3: recall tối đa 83% (5/6 vật), ở 0.3 precision 62% (`qc/threshold_sweep.md`) | r03: tìm đúng tuk-tuk ở img_002 (0.97) và xích lô ở img_032 (0.86, 0.70) mà YOLO26 bỏ sót hoặc gán bicycle; nhưng nhận nhầm xe tải ở img_011 (0.34) và bỏ sót hai xích lô bên phải img_032 |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json` (không dùng `--thr` vì mọi lớp đều ở ngưỡng mặc định 0.3, xem `qc/threshold_sweep.md`)
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: SAM 2.1 base (`sam2.1_b.pt`), đã chạy ở r04 (10 mask trên 2 ảnh, 6.64 giây/ảnh trên CPU). Lý do: mask bám viền mái, thân và bánh xe tốt ở cả 3 vật khó chỉ với 3 đến 6 click; lỗi chủ yếu do người che xe (img_013, img_010), không phải do model; chậm trên CPU (6.64 giây/ảnh ở r04) nhưng chấp nhận được vì chỉ làm 3 vật; không cần quyền truy cập đặc biệt như SAM 3. Chưa so sánh với SAM 1 hoặc SAM 3.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? Tuk-tuk / xích lô: YOLO26 chỉ có 80 lớp COCO, không có lớp xe ba bánh, nên đổi ngưỡng không tạo ra được nhãn `three_wheeler`: img_002 tuk-tuk không có box nào (bỏ sót), img_032 đuôi hai xích lô bị gán `bicycle` (0.38 và 0.27, gán sai lớp). Sửa bằng cách dùng YOLOE gõ chữ cho lớp `three_wheeler` (r03), rồi vẫn phải duyệt tay vì YOLOE còn bỏ sót xích lô ở img_032 và nhận nhầm xe tải ở img_011. Trên gold (không có truck nào), YOLO26 vẫn cho box truck sai ở mọi ngưỡng: 9 / 5 / 3 / 1 box ở 0.1 / 0.3 / 0.5 / 0.7, trong đó có tuk-tuk ở img_012 (0.53, 0.66).
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): Điểm thấp mà đúng: xe tải thật ở img_011 (YOLO26, 0.34) và xe máy cùng người ngồi trên xe ở img_002 (0.42 và 0.5), đều mất khi nâng ngưỡng lên 0.5. Điểm cao mà sai: YOLO26 gọi tuk-tuk là truck với điểm 0.53 và 0.66 ở img_012 (trùng gold three_wheeler, IoU 0.86 và 0.87); YOLOE gán three_wheeler 0.71 cho một box trùng (IoU 0.98) với một người trong gold ở img_020. Điểm số không báo trước được hai loại lỗi này.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan: gold chỉ có 4 ảnh, 48 đối tượng (person 16, car 8, motorbike 8, three_wheeler 6, bicycle 5, ignore_region 4, bus 1, truck 0), nên bus và truck không đủ vật để chọn ngưỡng riêng, three_wheeler và bicycle chỉ ở mức ước lượng thô.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
| --- | --- | --- | --- |
| accepted | 167 | 71.7% | 233 box máy (đã loại 23 đối tượng quanh bẫy) |
| edited | 38 | 16.3% | 233 box máy |
| deleted | 28 | 12.0% | 233 box máy |
| added | 46 | 18.3% | 252 đối tượng sau duyệt (trong đó 5 ca là đổi lớp, máy sót thật khoảng 41) |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)? `three_wheeler`: 12/22 box máy bị sửa hoặc xoá (54.5%, theo `review_stats.json`: giữ 10, sửa 6, xoá 6). Theo ảnh (tính riêng, chưa loại vùng bẫy nên có thể lệch ±1): img_008 và img_010 cả 3 box đều phải sửa (box lỏng); img_013 và img_018 box YOLOE gán `three_wheeler` thực ra là xe buýt (đổi sang `bus`), img_034 là xe tải (đổi sang `truck`), cùng kiểu nhận nhầm xe tải ở img_011 trong r03. Đứng sau là `motorbike`: 21/64 = 32.8% bị sửa hoặc xoá, và vẽ thêm 16 box; nhiều nhất ở img_018 (máy 6 box: giữ 1, sửa 4, xoá 1, thêm 5; ảnh dày xe máy) và img_034 (sửa 4, thêm 3). `bus`, `truck`, `bicycle` chỉ có 2 đến 5 box máy nên không kết luận.
- So với gold: precision / recall của **máy** 76.5% (KTC 95%: 60-88%) / 59.1% (KTC 95%: 44-72%); TP 26, FP 8, FN 18 (mẫu số recall: 44 đối tượng gold; số đo lạc quan vì ngưỡng và chấm cùng 4 ảnh gold)

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 10/12 = 83.3% (KTC 95%: 55-95%); theo loại: extra 3/3, wrong_class 3/3, missing 2/3, loose 2/3. Loại nào lọt: img_024 (missing: box `car` ở khoảng x=703, y=340, cỡ 166x86 px bị xoá khỏi pre-label; đây là một trong 6 ảnh dày xe máy) và img_022 (loose: box `motorbike` bị dịch lên trên, cạnh trên cách box gốc khoảng 120 px và cao 508 thay vì 436 px). Vì sao (bạn đã bỏ bước nào của checklist)? Đọc từ dữ liệu: img_024 là ảnh dày xe máy (một trong 6 ảnh nhiều vật nhất), box `car` bị xoá nằm giữa nhiều box xe máy nên thiếu một box rất khó nhận ra (lỗi bỏ sót, điểm số không báo); ở img_022 box vẫn đúng lớp `motorbike` và vẫn bao vật nên dễ bị nhìn lướt qua ở bước 4 (kiểm lỏng). Tôi đã đối chiếu với lần duyệt của mình và thấy phù hợp.
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? Accepted 71.7% đi cùng bắt bẫy 83.3% (đạt ngưỡng 80%, nhưng KTC 55-95% nên chưa kết luận mạnh). Hai loại bẫy bắt đủ 3/3 là extra (box ma) và wrong_class (sai lớp); hai loại lọt là missing và loose, tức lỗi nằm ở chỗ máy không vẽ hoặc vẽ lệch vài chục pixel. Recall của máy so với gold chỉ 59.1% nên accepted cao không có nghĩa máy bắt đủ vật.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 5 phút/ảnh (20 phút / 4 ảnh) vs duyệt 3.3 phút/ảnh (80 phút / 24 ảnh), nhanh hơn gán tay khoảng 33% (1 - 3.33/5). Thiên lệch (bạn gán gold trước nên đã quen cảnh): các ảnh gold và batch cùng kiểu phố đông xe máy, người, tuk-tuk nên thời gian duyệt có thể thấp hơn so với khi chưa quen cảnh.
- Kết luận: chỉ đáng dùng có điều kiện, dựa trên số liệu: duyệt nhanh hơn gán tay khoảng 33% (3.3 so với 5 phút/ảnh), accepted 71.7% (167/233 box máy), bắt bẫy 10/12 = 83.3%; nhưng máy chỉ đạt recall 59.1% so với gold (44 vật) và `three_wheeler` bị sửa hoặc xoá 12/22 box. Vì vậy dùng pre-label cho person, car, motorbike; riêng `three_wheeler` và ảnh dày xe máy phải quét ảnh trước và kiểm kỹ phần bỏ sót, không tin điểm số.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. Quét ảnh trước và ghi số vật đếm được theo lớp của từng ảnh vào log trước khi hiện nhãn máy, ở 24/24 ảnh; sau khi duyệt xong so số box cuối với số đã đếm (kiểm bằng log: mỗi ảnh có một dòng đếm).
  2. Với 6 ảnh dày xe máy (005, 018, 021, 024, 028, 029): chia ảnh thành 4 góc phần tư, kiểm bỏ sót từng góc, cụm xe cao dưới 20 px thì dùng một `ignore_region`; kiểm bằng số ảnh dày đã làm đủ 4 góc (6/6) và số box `added` ở các ảnh này (img_018: 5 box motorbike thêm), vì bẫy missing lọt ở img_024.
  3. Với mọi box `three_wheeler` luôn kiểm lại lớp (xe buýt hoặc xe tải bị gán nhầm: img_013, 018, 034) và độ ôm sát (img_008, 010 cả 3 box phải sửa); kiểm bằng tỉ lệ box `three_wheeler` bị sửa hoặc xoá ở lần duyệt sau (lần này 12/22 = 54.5%) và nghỉ 3 đến 5 phút sau mỗi 10 ảnh (kiểm bằng dòng `break` trong `time_log.csv`).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
| --- | --- | --- | --- | --- | --- |
| img_013 (tuk-tuk có người lái) | SAM 2.1 base (`sam2.1_b.pt`) | 3 (2 dương, 1 âm) | Có: người lái bị tô gần hết (còn lốm đốm điểm không được chọn), một số hành khách ngồi trong xe cũng bị tô; không có hàng hoá rời | 1 click âm lên người lái không đủ để loại người ra khỏi mask xe; mask 316400 px (45.3% ảnh), 1 polygon | `masks/tuktuk_img013.jpg` |
| img_010 (xích lô nhìn từ sau, có người đạp) | SAM 2.1 base (`sam2.1_b.pt`) | 5 (3 dương, 2 âm) | Không gồm người đạp (click âm lên áo khoác loại được), chỉ còn vài điểm lẻ trên áo | Mask bị chia 3 polygon vì người đạp che giữa xe; khung và cột mái mảnh chỉ được tô lốm đốm; mask 78416 px (11.3% ảnh); không lấn sang xích lô bên cạnh | `masks/xichlo_img010.jpg` |
| img_023 (tuk-tuk giữa, bị người đứng che, chạm hai xe bên cạnh) | SAM 2.1 base (`sam2.1_b.pt`) | 6 (3 dương, 3 âm) | Không gồm người đứng áo vàng; không lấn sang hai tuk-tuk bên cạnh | 3 click âm đủ để tách khỏi hai xe chạm nhau; mask 52592 px (6.7% ảnh), 1 polygon; còn khuyết nhỏ ở bánh trước | `masks/tuktuk_img023.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? Giúp: 4.2.4 (hai vật chạm nhau thì mỗi vật một polygon, vùng chạm thuộc vật phía trước) giải thích vì sao phải click âm lên người lái và người đạp; 4.2.2 (lỗ nhỏ dưới ~5% diện tích thì bỏ qua) phù hợp với mask lốm đốm. Thiếu: chưa nói mask xe ba bánh có gồm người ngồi trong buồng lái không (img_013), chưa có quy tắc cho chi tiết mảnh như khung mái xích lô mà SAM tô lốm đốm (img_010), và câu hỏi mở số 5 về kính chắn gió chưa chốt.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: không làm (dùng script `sam2_points.py` offline với 3 đến 6 click mỗi vật, SAM 2.1 base, không cài interactor trong CVAT)

## 6. Calibration (Part 6)

- Số quyết định: 6 (`calibration_decisions.csv`: CAL-01 đến CAL-06); quan trọng nhất: CAL-03 (YOLOE gán `three_wheeler` cho xe buýt ở img_013, img_018 và xe tải ở img_034: đổi sang `bus` / `truck`) và CAL-01 (xe máy đỗ dày gần camera: gán từng xe, cụm quá nhỏ hoặc bị che thì `ignore_region`, img_018, img_024)
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: chốt câu 2, 3, 7 của mục 6 (xe máy đỗ dày; hàng nhô ra trên 20% chiều dài xe không tính vào box; không đặt ngưỡng `conf` tối thiểu); `three_wheeler` không gồm xe từ 4 bánh trở lên và đuôi xích lô không phải `bicycle` (CAL-03, CAL-04); mask xe ba bánh loại người (CAL-05). Chưa chốt câu 1, 4, 6, và câu 5 (kính chắn gió) chỉ chốt tạm: 4 ca guideline còn khoảng trống
- Nguyên nhân gốc: máy cho CAL-03 và CAL-04 (YOLOE nhận nhầm hình dạng, YOLO26 không có lớp xe ba bánh); guideline mơ hồ cho CAL-01, CAL-02, CAL-05 (v1.0 chưa nói ngưỡng xe đỗ dày, hàng nhô và người trong mask)

## 7. QC và quyết định (Part 6)

- Mẫu: 8 ảnh (3 ảnh `can_review` + 5 ngẫu nhiên, seed 42), 85 đối tượng; người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt nên dễ cùng bỏ sót với chính mình)
- Lỗi theo **đối tượng**: 6/85 = 7.1% · theo **ảnh**: 5/8 = 62.5% (ngưỡng Release ≤ 2%, không đạt)
- Top lỗi: bỏ sót 5 lỗi (ví dụ img_024: nhiều ô tô nhìn rõ chưa có box; img_004, img_022, img_035 mỗi ảnh có vật thiếu) và thừa 1 lỗi (img_003: motorbike #2 ở mép trái không phải xe máy). Không thấy lỗi sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask trong mẫu. Lỗi `bo_sot` lặp lại nên là lỗi hệ thống của cách duyệt, không phải sai từng ảnh
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): 5 ảnh lỗi trên 8 ảnh, cận trên Wilson 95% của tỉ lệ ảnh lỗi thật là 86%. Với 8 ảnh, ngưỡng 2% không thể được chứng minh kể cả khi mẫu sạch hoàn toàn; chỉ chứng minh được điều ngược lại (tỉ lệ lỗi rõ ràng vượt 2%). Ngoài ra người kiểm cũng là người duyệt, và img_003 truck #7 ở xa không kiểm chứng được nên không tính
- Script kiểm được / không kiểm được: script đếm lỗi theo phiếu mắt người và tính Wilson, chạy `validate_labels.py` kiểm định dạng; script không kiểm được box ôm sát, bỏ sót cùng với máy (người và máy cùng không thấy), hay một vật có nên gán theo guideline

**Quyết định: Hold**, ngày 2026-11-18. Lý do (đi theo Hold → Rework → Release): guideline còn 4 ca chưa rõ (câu 1, 4, 6 chưa chốt; câu 5 kính chắn gió chốt tạm) nên theo cây quyết định phải Hold trước; ngay cả khi chốt hết, ảnh lỗi 5/8 = 62.5% vượt xa ngưỡng 2% và lỗi bỏ sót lặp lại nên sau đó cũng là Rework
Nếu Rework / Hold, việc cần làm tiếp: (1) chốt câu 1, 4, 5, 6 bằng ảnh ví dụ rồi nâng guideline; (2) sửa các box bỏ sót ở img_004, 022, 024, 035 và xoá box thừa ở img_003 trong CVAT, export `qc_fixed.json`; (3) chạy lại `build_release.py` và `qc_sample.py --force` rồi kiểm mẫu mới, ghi vòng 2 vào báo cáo; (4) với ảnh dày xe, kiểm 4 góc phần tư và thêm `ignore_region` cho cụm mờ

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào): Claude (Anthropic), dùng xuyên suốt bài. Việc AI làm: (1) giải thích cấu trúc repo, quy trình và hướng dẫn cài đặt, lệnh chạy từng Part; (2) gợi ý cách hiểu guideline v1.0 trước khi gán gold;
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows, chỉ CPU (torch 2.15.0+cpu, không GPU), Python 3.12.10 với `.venv` đặt ngoài repo, ultralytics 8.4.215, CVAT 2.81.0 chạy bằng Docker. Vướng mắc: lần đầu chạy `check_env.py` bằng Python 3.13 hệ thống vì chưa kích hoạt venv nên báo thiếu yaml và transformers; model tải vào `day08/models` ngoài repo nên phải copy vào `models/` của repo.
- Lỗi / bug script đã gặp (nếu có): chưa gặp lỗi script.
- Một điều bạn làm khác lần sau: sau khi duyệt xong mỗi ảnh dày xe hoặc ảnh đêm, quét lại từng góc ảnh một lần nữa để tìm vật chưa có box (5/6 lỗi QC là bỏ sót), và chốt các câu hỏi mở của guideline bằng ảnh ví dụ ngay từ lúc gán gold thay vì để đến Part 6
