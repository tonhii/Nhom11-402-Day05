# SPEC — AI Product Hackathon

**Nhóm:** Phòng 402 - Nhóm 11
   2A202600369-Hồ Thị Tố Nhi
   2A202600107- Lê Thị Phương
   2A202600077-Hoàng Văn Kiên
   2A202600042-Đỗ Văn Quyết
   2A202600368- Hà Hữu An
   2A202600095- Lê Hoàng Long
**Track:** ☑ VinFast
**Problem statement:** Khách hàng cá nhân và nhân viên sales mất 15–20 phút tư vấn xe VinFast phù hợp vì phải xem nhiều trang và so sánh thủ công — AI hỏi 3–4 câu và gợi ý ngay 2–3 dòng xe phù hợp nhất, giảm thời gian xuống ~3 phút.

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | Khách cá nhân & nhân viên sales tại showroom VinFast mất 15–20 phút so sánh xe thủ công — AI hỏi ngân sách, nhu cầu, số người dùng, ưu tiên điện/xăng → gợi ý 2–3 dòng xe kèm lý do cụ thể | AI gợi xe sai phân khúc → user thấy thông số hiển thị ngay bên cạnh → tự lọc lại hoặc nhập lại ngân sách; AI luôn giải thích *tại sao* gợi xe đó | ~$0.005–0.01/lượt, latency <3s; risk chính: catalog giá/ưu đãi bị lỗi thời nếu không sync định kỳ |

**Automation hay augmentation?** ☑ Augmentation

Justify: AI gợi ý 2–3 xe kèm lý do, user (hoặc sales) xem xét và quyết định. Sales vẫn là người chốt deal cuối. Cost of reject = 0.

**Learning signal:**

1. User correction đi vào đâu? → Log preference pair: xe bị bỏ qua vs xe được chọn → dùng để cải thiện ranking model
2. Product thu signal gì để biết tốt lên hay tệ đi? → % user chọn ≥1 xe được gợi ý; tỷ lệ lead chuyển lịch lái thử qua sales
3. Data thuộc loại nào? ☑ User-specific ☑ Domain-specific ☑ Human-judgment
   Có marginal value không? → Có. Model chung không biết preference ranking của từng user VinFast (ai thích không gian, ai thích hiệu năng, ai bị thuyết phục bởi ưu đãi tài chính) — đây là data có giá trị riêng biệt.

---

## 2. User Stories — 4 paths

### Feature: Gợi ý xe theo ngân sách & nhu cầu

**Trigger:** User nhập ngân sách + nhu cầu → AI hỏi thêm 1–2 câu làm rõ → gợi ý 2–3 dòng xe kèm giá, thông số, lý do

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | User nhập "ngân sách 800 triệu, gia đình 4 người" → AI gợi VF 5, VF 6 kèm giá niêm yết + ưu đãi hiện hành → user thấy phù hợp, nhấn "Đặt lịch lái thử" |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | Ngân sách borderline (vd 1 tỷ 050 triệu — nằm giữa 2 phân khúc) → AI hiển thị 2 nhãn gợi ý + hỏi "Bạn muốn ưu tiên không gian hay hiệu năng?" để thu hẹp |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | AI gợi xe vượt ngân sách do hiểu nhầm chưa gồm phí trước bạ → user thấy giá thực cao hơn khi xem chi tiết → nhập lại ngân sách; hệ thống log lỗi để cải thiện prompt |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | User bỏ qua gợi ý, tự chọn xe khác → hệ thống ghi nhận preference pair (rejected / chosen) → cải thiện ranking model theo thời gian |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☑ Precision

Tại sao? Gợi ít nhưng đúng tốt hơn gợi nhiều mà sai — một lần gợi xe sai phân khúc là đủ mất trust của cả khách lẫn sales.

Nếu sai ngược lại thì chuyện gì xảy ra? Nếu chọn precision nhưng low recall → user không thấy đủ lựa chọn → hỏi thêm → vẫn ổn, không nghiêm trọng bằng gợi sai.

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| % gợi ý xe đúng phân khúc ngân sách | ≥ 90% | < 75% trong 1 tuần |
| % user chọn ≥ 1 xe được gợi ý | ≥ 60% | < 40% liên tục 2 tuần |
| Latency phản hồi gợi ý | < 3s | > 6s trung bình |
| Tỷ lệ lead chuyển lịch lái thử (qua sales) | ≥ 15% | < 8% sau 1 tháng |

---

## 4. Top 3 failure modes

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | Catalog giá/ưu đãi bị lỗi thời (chưa sync với hệ thống VinFast) | AI gợi giá thấp hơn thực → khách tức khi ra showroom → mất trust nghiêm trọng | Sync catalog tự động mỗi ngày; hiển thị rõ "Giá tham khảo, cập nhật [ngày]" trên UI |
| 2 | User nhập ngân sách chưa rõ — chưa gồm phí trước bạ, không rõ mua thẳng hay trả góp | AI gợi xe không phù hợp, tự tin cao — **user không biết mình bị gợi sai** (failure mode nguy hiểm nhất) | Bắt buộc hỏi rõ "Ngân sách này mua thẳng hay trả góp? Đã gồm phí trước bạ chưa?" trước khi gợi ý |
| 3 | Sales dùng AI gợi ý mà không kiểm tra tồn kho / màu xe thực tế | Hứa xe với khách nhưng thực tế không có → bể deal, khiếu nại | AI luôn thêm disclaimer: "Vui lòng xác nhận tồn kho với showroom trước khi tư vấn khách" |

---

## 5. ROI 3 kịch bản

|   | Conservative | Realistic | Optimistic |
|---|-------------|-----------|------------|
| **Assumption** | 50 lượt/ngày, 60% hài lòng | 200 lượt/ngày, 80% hài lòng | 500+ lượt/ngày, 90% hài lòng, mở rộng toàn chuỗi |
| **Cost** | ~$15/ngày inference | ~$50/ngày | ~$120/ngày |
| **Benefit** | Tiết kiệm ~2h tư vấn/ngày của sales | Tiết kiệm ~8h/ngày, tăng tỷ lệ lead chuyển đổi ~10% | Giảm headcount tư vấn cơ bản, mở rộng toàn quốc |
| **Net** | Dương nếu 1 deal thêm/tuần nhờ AI | Dương rõ ràng nếu conversion tăng 10% | Lợi thế cạnh tranh lớn cho chuỗi showroom |

**Kill criteria:** Dừng khi cost > benefit 2 tháng liên tục, hoặc precision < 75% mà không cải thiện được sau 1 tháng tối ưu.

---

## 6. Mini AI spec

Sản phẩm giải bài toán tư vấn xe VinFast phù hợp ngân sách — hiện mất 15–20 phút/lượt vì phải xem nhiều trang và so sánh thủ công. AI chatbot hỏi 3–4 câu nhanh (ngân sách, mục đích sử dụng, số người dùng, ưu tiên điện hay xăng), sau đó gợi ý 2–3 dòng xe kèm lý do cụ thể và link so sánh chi tiết.

Sản phẩm chạy theo mô hình **augmentation**: AI không chốt thay người, chỉ rút ngắn hành trình khám phá. Cả khách hàng lẫn nhân viên sales đều là user — khách dùng trực tiếp trên web/app, sales dùng như công cụ hỗ trợ tư vấn nhanh tại quầy.

**Precision** được ưu tiên hơn recall: gợi ít nhưng đúng, tránh gợi xe sai phân khúc ngân sách vì điều đó phá vỡ trust ngay lần đầu. Risk chính là catalog giá bị lỗi thời và user không làm rõ ngân sách — cả hai đều có mitigation cụ thể.

**Data flywheel:** mỗi lần user chọn hoặc bỏ qua một gợi ý tạo ra preference signal. Khi đủ volume, dữ liệu này dùng để cải thiện ranking model — sản phẩm tốt hơn theo thời gian mà không cần label thủ công. Model chung không có data này; đây là moat thực sự của sản phẩm.
