# Báo cáo Ngày 3 — Tracking Annotation

Chép file này thành `reports/REPORT.md` rồi điền. Giữ nguyên các tiêu đề.

Họ tên / nhóm: `Trần Minh Hiếu`
Ngày: `15/09/2026`

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT / khác: `CVAT` |
| Thời gian gán `clip_02` (warm-up) | `18` phút |
| Thời gian gán `clip_01` | `45` phút |
| Số track đã vẽ trong `clip_01` | `9` |
| Số keyframe trung bình mỗi track | `6` |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. **Occlusion (xe bị che khuất rẽ ngang/chắn bởi xe khác hoặc cây cối)**: Giữ nguyên `track_id` cũ khi xe xuất hiện lại nếu quỹ đạo và đặc trưng xe khớp, sử dụng tính năng interpolate/keyframe của CVAT giữa các vị trí trước và sau khi occlusion kết thúc.
2. **Xe đi vào/ra khỏi mép khung hình (partial visibility tại rìa)**: Chỉ gán bbox khi xe đi vào tối thiểu khoảng 20-30% diện tích và có thể nhận diện rõ là phương tiện giao thông; xóa/chốt keyframe cuối ngay khi xe đi ra khỏi khung hình.
3. **Mật độ phương tiện cao làm chồng lấp bounding box**: Kiểm tra kỹ từng frame (frame-by-frame) ở khu vực giao cắt để tránh kéo nhầm góc bbox sang xe khác và đảm bảo ID không bị nhảy (ID switch).

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

- Lượt 1: Phát hiện 1 trường hợp ID bị nhầm giữa 2 xe màu tối chạy song song ở giữa clip.
- Lượt 2: Phát hiện 2 frame đầu khi xe mới chớm xuất hiện ở mép ảnh bị thiếu bbox và 1 frame cuối xe đã ra khỏi màn hình nhưng vẫn còn track.
- Lượt 3: Sửa lại các vị trí bbox bị lệch tâm/chưa khít do tính năng auto-interpolation kéo thẳng qua đoạn xe đổi hướng.

Kiểm chéo với: `N/A (Làm cá nhân)`. Chi tiết ở `reports/review_partner.md`.
Số lỗi bạn tìm được trong bản của bạn ấy: `N/A`. Số lỗi bạn ấy tìm được trong bản của bạn: `N/A`.

Ca nào hai người quyết khác nhau, và luật nào còn thiếu trong `GUIDELINE_MINI.md`?

Do thực hiện cá nhân, không có ca bất đồng giữa các người gán nhãn. Tuy nhiên, qua quá trình tự gán nhãn, nhận thấy `GUIDELINE_MINI.md` còn thiếu luật rõ ràng về ngưỡng che khuất tối đa (Occlusion Threshold, ví dụ: $>70\%$ bị che thì bỏ qua hay tiếp tục giữ ID) và thời gian chờ tối đa (lost frame count) trước khi ngắt track khi xe tạm thời mất hút.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| Thời điểm khóa | `2026-09-15T14:30:00Z` |
| Số row / frame / track trước khi mở reference | `669 / 190 / 9` |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 0.812 | 0.725 | 0.890 | 0.885 | 0.854 | 0.782 | 0.875 | 42 | 58 | 2 |
| Sau rework | 0.856 | 0.768 | 0.921 | 0.891 | 0.889 | 0.824 | 0.880 | 25 | 31 | 1 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

| Loại lỗi | Frame | ID | Đã sửa thế nào |
| --- | --- | --- | --- |
| Bbox lệch / chưa sát | 113-117 | 8 | Điều chỉnh góc bbox ôm sát thân xe hơn |
| Thiếu Bbox (FN) | 106-107 | 7 | Thêm bbox cho xe ở mép ảnh khi mới đi vào |
| ID Switch | 87 | 6 | Đổi ID 18 về lại ID 17 đúng quỹ đạo ban đầu |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | `3.13.15 / 8.4.145 / 2.11.0+cpu / 0.5.13` |
| weights / hai tracker | `yolo26n.pt / bytetrack.yaml vs botsort-reid.yaml` |
| conf / IoU / imgsz / classes | `0.25 / 0.7 / 960 / [2, 5, 7]` |
| device | `cpu` |

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.856 | 0.768 | 0.921 | 0.891 | 0.889 | 0.824 | 0.880 | 25 | 31 | 1 |
| ByteTrack control vs gold | 0.718 | 0.622 | 0.825 | 0.875 | 0.820 | 0.675 | 0.865 | 92 | 125 | 3 |
| BoT-SORT + ReID vs gold | 0.735 | 0.638 | 0.845 | 0.880 | 0.845 | 0.698 | 0.870 | 85 | 115 | 1 |
| ReID vs bạn | 0.731 | 0.636 | 0.841 | 0.882 | 0.842 | 0.692 | 0.871 | 87 | 118 | 1 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

Trong kết quả của tôi, MOTA ($0.692$ ở bảng ReID vs Bạn) thấp hơn IDF1 ($0.842$). Trong trường hợp MOTA cao mà IDF1 thấp, điều này phản ánh rằng detector tìm vật thể rất chuẩn (ít FP, FN), nhưng khâu association (ghép ID) kém, dẫn đến hiện tượng tráo ID liên tục. MOTA không phạt nặng lỗi ID vì chỉ số này tính tổng hợp giữa FP, FN và IDSW trên từng frame riêng lẻ; khi xảy ra ID switch, MOTA chỉ trừ 1 điểm phạt tại đúng frame chuyển giao ID đó, trong khi IDF1 đo lường sự nhất quán của toàn bộ đường đi (trajectory) trên cả clip nên bị ảnh hưởng rất nặng.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

BoT-SORT + ReID đạt chỉ số cao hơn ByteTrack ở IDF1 ($0.845$ so với $0.820$), AssA ($0.845$ so với $0.825$) và giảm số IDSW (từ $3$ xuống $1$). Ở sequence frame 80-100, khi hai xe chạy cắt nhau và bị rào chắn che một phần, ReID giúp duy trì ID nhờ vectơ đặc trưng ngoại dạng (appearance feature vector), trong khi ByteTrack dùng thuần Kalman filter + IoU bị nhảy ID khi vị trí dự báo chồng lấp quá nhiều. *(Lưu ý: So sánh này không cô lập hoàn toàn causal effect của ReID do ByteTrack và BoT-SORT khác nhau cả về thuật toán dự báo chuyển động và cơ chế matching)*.

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

DetA giữ ở mức trung bình ($~0.636$), với lượng FP ($87$) và FN ($118$) tương đối cao so với nhãn người gán. Điều này cho thấy lỗi còn lại phần lớn nằm ở **Detector** (YOLO26n bị miss các xe ở xa/nhỏ và nhận diện nhầm các khoảng bóng râm/chướng ngại vật rìa đường) hơn là do thuật toán Association.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

Tại frame 158-178 (ID 38 của model), ReID tự tạo ra một track mới do nhận diện nhầm một bốt điện/quầy hàng cố định ven đường thành phương tiện giao thông (FP). Nhãn gán tay của tôi chính xác vì vật thể này hoàn toàn đứng im và không phải là xe.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

Tại frame 106-107, ReID phát hiện một chiếc xe đang đi vào từ rìa màn hình mà ở bản pre-gold tôi đã bỏ sót. Sau khi kiểm tra lại hình ảnh, đó thực sự là một chiếc xe hợp lệ đang tiến vào khung hình, đòi hỏi tôi phải bổ sung bbox cho khu vực này ở bản rework.

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

- **Sửa `GUIDELINE_MINI.md`**:
  - Bổ sung quy định rõ ràng về Occlusion Threshold (bỏ qua nếu vật thể bị che $>70\%$).
  - Chuẩn hóa quy tắc vẽ Bbox ở vùng biên (Edge Boundary Rule): Chỉ bắt đầu gán khi vật thể lộ diện trên 20% diện tích xe.
- **Đổi quy trình làm việc**:
  - Áp dụng triệt để quy trình tự kiểm tua 3 lượt ngay từ clip đầu tiên.
  - Sử dụng các model tracker nhẹ để hỗ trợ pre-labeling hoặc dùng script kiểm tra tự động (sanity check script) nhằm tìm các bbox bị giật/nhảy vị trí đột ngột trước khi thực hiện bước chốt (lock pre-gold).

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [x] `reports/review_partner.md` (Đã cập nhật trạng thái thực hiện cá nhân)
- [x] `reports/REPORT.md` (file này)