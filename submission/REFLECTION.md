# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

---

**1. The flywheel.**
Bước dễ sai nhất là `decontaminate()` trong `dataset.py`. Nếu không loại các prompt
trùng với eval set ra khỏi preference pairs, model sẽ train trên chính những câu
nó dùng để tự chấm điểm — eval score tăng ảo nhưng production không cải thiện.
Cách phát hiện: sau mỗi pipeline run, log tỉ lệ `len(clean) / len(pairs)`. Nếu tỉ
lệ này đột ngột tăng về 1.0 (không có gì bị loại), đó là dấu hiệu decontamination
đã bị bỏ qua hoặc không match được gì — cần alert.

**2. Decontamination.**
Nếu bỏ bước này, model học thuộc đáp án của chính bài kiểm tra. Eval loss giảm
nhanh, accuracy trông rất cao — nhưng đó là memorization, không phải generalization.
Cái "nói dối" sẽ lộ ra khi triển khai thực tế: người dùng hỏi câu tương tự nhưng
diễn đạt khác một chút, model mất phương hướng hoàn toàn. Dấu hiệu sớm nhất: gap
lớn giữa eval score nội bộ và human evaluation bên ngoài.

**3. Point-in-time.**
Trong hệ thống chấm điểm tín dụng: feature `total_outstanding_debt` của khách hàng
thay đổi liên tục. Nếu join theo giá trị hiện tại thay vì giá trị tại thời điểm
khoản vay được phê duyệt, model thấy số nợ đã trả hết (tương lai), nghĩ rủi ro
thấp, dự đoán sai hoàn toàn. Hậu quả thực tế: mô hình chấp thuận những khoản vay
lẽ ra phải từ chối.

**4. Graph vs vector.**
Graph trả lời tốt câu hỏi multi-hop: *"Widget giao hàng từ kho nào?"* — câu trả
lời cần đi qua 2 bước (widget → IS_A → accessory → SHIPS_FROM → Hanoi), không có
chunk đơn lẻ nào chứa đủ cả hai. Vector retrieval không thể bridge được khoảng
cách này vì nó chỉ trả về chunk gần nhất chứ không nối fact.
Graph là overkill cho câu hỏi lookup đơn giản: *"Chính sách đổi trả widget là
bao nhiêu ngày?"* — một chunk duy nhất đã có đủ câu trả lời, vector search là đủ
và rẻ hơn nhiều.
