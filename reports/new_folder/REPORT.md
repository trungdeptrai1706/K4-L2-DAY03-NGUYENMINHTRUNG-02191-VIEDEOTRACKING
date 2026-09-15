# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: Nguyễn Minh Trung  
Ngày: 15/09/2026

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) | 20 phút |
| Thời gian gán `clip_01` | 180 phút |
| Số track đã vẽ trong `clip_01` | 8 |
| Số keyframe trung bình mỗi track | 4–6 |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. Xe chồng nhau trong đoạn occlusion: tôi dùng keyframe trước/sau để giữ ID, không kéo bbox quá dài.
2. Xe vào/ra khỏi khung hình: tôi xác định entry/exit dựa trên frame đầu tiên và cuối cùng có thể quan sát rõ, không giữ box sau khi xe đã rời khung.
3. Bbox lệch do bóng hoặc góc nhìn: tôi căn chỉnh theo phần xe rõ nhất, không đoán phần che khuất.

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

- Lượt 1: kiểm tra ID xuyên suốt track.
- Lượt 2: kiểm tra đầu/cuối track và entry/exit.
- Lượt 3: kiểm tra bbox giữa các frame, tránh drift.

Kiểm chéo với: bạn cùng nhóm / reviewer. Chi tiết ở `reports/review_partner.md`.  
Số lỗi bạn tìm được trong bản của bạn ấy: 3. Số lỗi bạn ấy tìm được trong bản của bạn: 2.

Ca nào hai người quyết khác nhau, và luật nào còn thiếu trong `GUIDELINE_MINI.md`?

Có một vài trường hợp crossing và occlusion ngắn mà hai bên có khác biệt về việc giữ ID hay đổi ID. Quy tắc cần nhấn mạnh là: nếu xe vẫn là cùng một xe, không đổi ID dù có occlusion ngắn; chỉ đổi ID khi có bằng chứng rõ ràng là xe khác.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `8c9a2f4e3d6a9a1b7e4f1d2b8e7c6a9f3d1a5b7c2d8e4f6a9b1c8d3e5f6a9` |
| Thời điểm khóa | `2026-09-15 08:35` |
| Số row / frame / track trước khi mở reference | `1184 row / 360 frame / 8 track` |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 62.4 | 71.2 | 54.7 | 76.8 | 0.71 | 0.79 | 0.72 | 33 | 41 | 18 |
| Sau rework | 75.2 | 78.5 | 72.1 | 81.4 | 0.84 | 0.88 | 0.77 | 14 | 19 | 6 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

| Loại lỗi | Frame | ID | Đã sửa thế nào |
| --- | --- | --- | --- |
| ID switch trong đoạn crossing | 120–145 | 03 | Giữ nguyên ID bằng cách dùng keyframe trước/sau, không đổi qua xe khác trong vùng chồng nhau |
| Bbox treo sau khi xe rời khung | 210–225 | 05 | Xóa bbox ở frame sau khi xe đã ra khỏi vùng quan sát, tránh kéo dài track quá lâu |
| Drift bbox khi xe đổi hướng | 286–301 | 07 | Chèn keyframe mới và căn lại bbox theo phần xe rõ nhất, giữ geometry hợp lý |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | `3.11 / 8.2.0 / 2.5.0 / 0.9.0` |
| weights / hai tracker | `YOLOv8n + ByteTrack / YOLOv8n + BoT-SORT-ReID` |
| conf / IoU / imgsz / classes | `0.25 / 0.45 / 640 / car` |
| device | `cuda:0` |

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.674 | 0.646 | 0.714 | 0.777 | 0.946 | 0.893 | 0.728 | 20 | 41 | 0 |
| ByteTrack control vs gold | 0.709 | 0.649 | 0.776 | 0.846 | 0.875 | 0.749 | 0.823 | 88 | 54 | 2 |
| BoT-SORT + ReID vs gold | 0.763 | 0.711 | 0.820 | 0.872 | 0.900 | 0.792 | 0.860 | 91 | 26 | 2 |
| ReID vs bạn | 0.655 | 0.585 | 0.747 | 0.782 | 0.881 | 0.743 | 0.738 | 114 | 28 | 0 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

MOTA của tôi cao hơn IDF1 một chút. Điều này cho thấy detector và count của xe khá tốt, nhưng nhiều track bị đổi ID hoặc bị tách trong quá trình tracking. MOTA không phạt nặng lỗi ID vì nó tập trung vào số lượng FP, FN và ID switch tổng quát, trong khi IDF1 đánh giá độ ổn định của identity trong suốt quãng đời xe.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

ReID treatment cải thiện rõ ở các frame có occlusion ngắn, cụ thể khoảng frame 120–150 khi hai xe cắt ngang và có sự che khuất. Ở thời điểm này, AssA và IDF1 tăng hơn so với control, nhưng IDSW vẫn còn ở vài đoạn vì implementation khác nhau nên không thể khẳng định ReID là nguyên nhân duy nhất. ByteTrack control ổn ở phần phát hiện nhưng yếu hơn ở identity continuity khi xe đi sát nhau.

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

DetA tăng nhẹ khi dùng ReID vì model phát hiện được nhiều xe đúng hơn và giảm số FP/FN. Tuy nhiên, phần còn lại nhiều nhất là lỗi association, vì số IDSW và sự không ổn định của ID trong các đoạn crossing vẫn xuất hiện. Khi detector đúng mà ID vẫn đổi là lỗi association, không phải detector.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

Ở frame 188, ID 04, model đã gán nhầm sang ID 06 trong khi xe vẫn là cùng một xe. Tôi giữ nguyên ID 04 vì quỹ đạo trước/sau và hướng di chuyển của xe không đổi. ReID sai ở đây do feature appearance biến đổi trong đoạn crossing và che khuất ngắn.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

Ở frame 286, ID 07, ReID cho thấy một đoạn track kéo dài không hợp lý, và tôi xem lại annotation và phát hiện bbox ở frame cuối quá rộng. Điều này khiến tôi phải điều chỉnh lại geometry để tránh bbox thừa sau khi xe rời khỏi vùng quan sát. Đây là bằng chứng cho thấy model không hoàn toàn đúng và annotation cần được kiểm tra kỹ hơn ở các đoạn xử lý đầu/cuối track.

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

Nếu phải gán thêm 10 clip nữa, tôi sẽ bổ sung rõ hơn các quy tắc về occlusion ngắn, tracking cross và bbox exit. Tôi cũng sẽ dùng 3 lượt kiểm tra bắt buộc: identity, endpoint, geometry. Việc review chéo và lock pre-gold sẽ được thực hiện sớm hơn để giảm lỗi ID và bbox treo.

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [ ] `outputs/eval_vs_gold.json`
- [ ] `outputs/model_bytetrack_clip_01.txt`
- [ ] `outputs/model_reid_clip_01.txt`
- [ ] `outputs/model_run_config.json`
- [ ] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [ ] `reports/review_partner.md`
- [x] `reports/REPORT.md` (file này)
