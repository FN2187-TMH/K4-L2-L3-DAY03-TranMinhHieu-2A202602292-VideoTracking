# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: `Trần Minh Hiếu (Thực hiện cá nhân)`
Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Bổ sung của nhóm (nếu có): `Chỉ gán các phương tiện di chuyển trên đường giao thông công cộng. Xe đỗ cố định trong gara/sân riêng có rào chắn hoàn toàn không gán.`

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật của nhóm | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che **dưới 25 frame** (mặc định của lab: 25 frame = 2 giây @ 12.5 fps) | Giữ tính liên tục của quỹ đạo (trajectory) khi phương tiện tạm thời bị khuất sau cây cối hoặc rào chắn ngắn. |
| Xe bị che lâu hơn ngưỡng trên | Tạo **track ID mới** khi xe xuất hiện lại. | Tránh đoán mò hoặc gán nhầm ID khi khoảng thời gian mất dấu quá dài. |
| Xe rời khung hình rồi quay lại | mặc định: **track mới** | Đảm bảo đúng chuẩn đánh giá tracking quốc tế khi vật thể đã đi ra ngoài biên ảnh quan sát. |
| Hai xe cắt nhau / chồng lên nhau | Giữ nguyên ID gốc của từng xe, không ngắt track hay hoán đổi ID. | Sử dụng tính năng keyframe/interpolation qua khu vực giao cắt để duy trì ID đúng cho từng xe. |

## 3. Luật bbox

| Tình huống | Luật của nhóm |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm phần **nhìn thấy được** (visible area) |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | bắt đầu track từ frame đầu tiên xác định được là xe bốn bánh; ngưỡng nhóm chọn: `Diện tích xe xuất hiện tối thiểu từ 20% - 30% tại mép ảnh.` |
| Xe đang đỗ, không di chuyển | Gán bbox và giữ nguyên track ID xuyên suốt nếu xe nằm trên làn giao thông chính; không gán nếu đỗ cố định ở bãi đỗ riêng lẻ ngoài đường. |
| Keyframe đặt dày ở đâu | Đặt dày tại các đoạn xe rẽ/đổi hướng, bắt đầu tăng/giảm tốc độ, và ngay trước/sau khi bị che khuất (occlusion). |

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1
- Clip / frame / ID: `clip_01 / frame 80-100 / ID 6 & ID 7`
- Tình huống: Hai xe ô tô màu tối di chuyển song song và cắt nhau tại khu vực có rào chắn che khuất một phần.
- Quyết định: Giữ nguyên ID 6 cho xe làn trong và ID 7 cho xe làn ngoài bằng cách tua kỹ từng frame (frame-by-frame) để kiểm tra góc di chuyển.
- Lý do: Tránh lỗi ID Switch (IDSW) khi hai vật thể có diện tích bbox chồng lấp quá nhiều.

### Ca 2
- Clip / frame / ID: `clip_01 / frame 106-107 / ID 7`
- Tình huống: Xe mới chớm xuất hiện ở góc rìa trái khung hình với diện tích rất nhỏ.
- Quyết định: Bắt đầu tạo keyframe gán bbox từ frame 106 khi xe đã lộ rõ khoảng 25% phần đầu xe.
- Lý do: Đúng quy định diện tích lộ diện tối thiểu để detector/annotator nhận diện chính xác là phương tiện bốn bánh.

### Ca 3
- Clip / frame / ID: `clip_01 / frame 113-117 / ID 8`
- Tình huống: Xe đang di chuyển rẽ cua làm góc nhìn bbox bị thay đổi đột ngột làm phần auto-interpolation kéo bbox lệch khỏi thân xe.
- Quyết định: Chèn thêm 2 keyframe ở giữa (frame 114 và 116) để căn chỉnh bbox ôm sát thân xe hơn.
- Lý do: Khắc phục lỗi Bbox lệch / chưa sát (đảm bảo độ chính xác IoU / LocA).

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:

- **Bổ sung ngưỡng che khuất (Occlusion Threshold)**: Nếu vật thể bị che khuất trên **70%** diện tích bởi vật thể cố định (cây cối, biển báo) nhưng thời gian che khuất dưới 25 frame thì vẫn tiếp tục giữ track ID cũ.
- **Chuẩn hóa quy tắc Edge Boundary**: Bbox ở vùng biên phải bắt đầu chính xác ngay khi diện tích xe xuất hiện đạt từ 20% trở lên và phải kết thúc ngay tại frame cuối cùng trước khi xe hoàn toàn ra khỏi mép khung hình.