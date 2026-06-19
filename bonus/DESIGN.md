# Bonus — Brainstorm: Pipeline RAG + Flywheel cho Chatbot CSKH của một Fintech cho vay tiêu dùng (Việt Nam)

## Bài toán & ràng buộc thực

**Ai dùng.** Một công ty fintech cho vay tiêu dùng tại Việt Nam vận hành chatbot
chăm sóc khách hàng (CSKH) trên app + Zalo OA. Khách hỏi về điều kiện vay, lãi
suất, lịch trả nợ, phí phạt trả chậm, và tình trạng khoản vay của *chính họ*.

**Dữ liệu gì.**
- ~3.000 file PDF/DOCX **chính sách sản phẩm + văn bản pháp luật** (thông tư NHNN,
  biểu phí), tiếng Việt có dấu, nhiều bản scan, bảng nhiều cột, **đổi phiên bản
  hằng tháng** khi biểu lãi suất thay đổi.
- **Trace sản xuất** từ chatbot (giống Day 13): mỗi lượt có prompt, câu trả lời,
  outcome (khách bấm "hữu ích" / mở ticket cho người thật / bỏ ngang).
- **Dữ liệu khoản vay** (dư nợ, lịch sử trả) để bot trả lời câu hỏi cá nhân hoá.

**Vì sao khó.** Tài liệu *trôi* (drift) liên tục; trả lời sai biểu phí là rủi ro
pháp lý thật. Dữ liệu khoản vay nhạy cảm theo **PDPL (Luật 91/2025)**. Và bot
phải vừa tra cứu chính sách (RAG) vừa không bao giờ rò rỉ trạng thái *tương lai*
của khoản vay vào dữ liệu train (point-in-time).

---

## 5 câu hỏi then chốt + quyết định

### 1. Phi cấu trúc → RAG thuần hay Hybrid RAG+KG? (câu 6)
**Quyết định: Hybrid — vector cho lookup, KG nhỏ cho quan hệ phiên bản/multi-hop.**
- *Vector-only*: rẻ, đủ cho "Phí phạt trả chậm là bao nhiêu?" (một chunk trả lời).
- *KG*: cần cho "Sản phẩm vay A đang áp biểu phí *phiên bản nào*, và phiên bản đó
  thay thế cái nào?" — multi-hop qua `sản phẩm → áp_dụng → biểu_phí_v3 → thay_thế
  → v2`. Không chunk đơn lẻ nào chứa đủ chuỗi này.
- **Đánh đổi X vs Y:** KG đầy đủ (trích xuất mọi thực thể) **vs** KG mỏng chỉ mô
  hình hoá *phiên bản & hiệu lực thời gian*. Chọn KG mỏng vì 80% giá trị multi-hop
  nằm ở quan hệ "phiên bản nào còn hiệu lực", còn lại để vector lo — rẻ hơn nhiều
  về token và công bảo trì.

### 2. Batch hay streaming cho ingest tài liệu? (câu 2)
**Quyết định: Batch theo sự kiện (event-triggered), không streaming.**
- Tài liệu đổi *hằng tháng*, không phải mỗi giây. Độ tươi "đủ" = vài phút sau khi
  pháp chế upload bản mới.
- **X vs Y:** cron quét hằng đêm **vs** trigger khi có file mới. Chọn trigger vì
  cron để lọt một biểu phí sai suốt 24h là rủi ro pháp lý; trigger đẩy độ trễ
  xuống phút mà không cần hạ tầng streaming thật (Kafka) tốn kém.

### 3. Train/serve parity — rò rỉ point-in-time ở đâu? (câu 5)
**Quyết định: `ASOF JOIN` mọi feature khoản vay theo `as_of = thời điểm lượt chat`.**
- Câu hỏi "Tôi còn nợ bao nhiêu?" sinh ra feature `du_no`. Nếu khi build dataset
  train ta join `du_no` *hiện tại* thay vì giá trị *tại thời điểm lượt chat*, model
  học từ một con số tương lai (khách đã trả bớt) → trả lời sai cho người đang nợ.
- **X vs Y:** snapshot bảng khoản vay mỗi đêm (đơn giản) **vs** lưu lịch sử đầy đủ
  + ASOF (đúng nhưng tốn hơn). Chọn ASOF — đây chính là lỗi lab số 12 minh hoạ,
  và trong tín dụng nó là thảm hoạ, không phải sai số nhỏ.

### 4. Flywheel — biến trace thành dữ liệu train mà không tự đầu độc? (câu 7)
**Quyết định: eval set từ trace *held-out*, DPO pairs từ ok-vs-escalation, +
decontaminate bắt buộc.**
- `chosen` = lượt được đánh "hữu ích"; `rejected` = lượt phải chuyển người thật.
- **X vs Y:** tin label tự động của bot **vs** chỉ lấy pair có **tín hiệu người
  thật** (bấm hữu ích / mở ticket). Chọn tín hiệu người thật — feedback bot tự
  chấm dễ thiên lệch, làm flywheel khuếch đại lỗi của chính nó.
- Decontamination (lab #11) **không tuỳ chọn**: prompt trùng eval bị loại, nếu
  không eval score tăng ảo do memorization.

### 5. Bối cảnh Việt Nam — PDPL & tiếng Việt đổi gì? (câu 10)
**Quyết định: tách dữ liệu cá nhân khỏi corpus train + chuẩn hoá tiếng Việt sớm.**
- **PDPL:** dư nợ, CCCD là dữ liệu cá nhân. → **redact/pseudonymize ở Bronze**,
  trước khi dữ liệu chạm bất kỳ dataset train nào; chỉ feature đã ẩn danh mới được
  ra Gold. Train trên PII thô là vi phạm pháp lý, không chỉ là "kỹ thuật xấu".
- **Tiếng Việt:** OCR bản scan hay nuốt dấu ("phi" vs "phí"). → bước chuẩn hoá
  Unicode NFC + từ điển sửa lỗi dấu chạy ngay sau OCR, vì lỗi dấu trôi xuống cuối
  pipeline sẽ làm hỏng cả embedding lẫn KG entity-matching.

---

## Phương án bị loại

**Loại: fine-tune model trên toàn bộ corpus chính sách thay vì RAG.** Lý do: tài
liệu đổi *hằng tháng* — fine-tune lại mỗi lần đổi biểu phí là vừa chậm vừa đắt,
và model vẫn có thể "bịa" số liệu cũ (hallucinate). RAG cho phép cập nhật chính
sách bằng cách thay index trong vài phút, có nguồn trích dẫn để kiểm chứng — bắt
buộc với bối cảnh pháp lý. Fine-tune chỉ dùng cho *giọng văn/định dạng trả lời*,
không dùng làm nguồn sự thật về biểu phí.

**Loại phụ: streaming thật (Kafka) cho ingest tài liệu** — over-engineering cho
dữ liệu đổi hằng tháng (xem câu 2).

---

## Sơ đồ kiến trúc

```
            ┌──────────────── NGUỒN ────────────────┐
            │ PDF/DOCX chính sách   Trace chatbot    │  Bảng khoản vay
            │ (scan, VN, drift)     (prompt/outcome) │  (PII, đổi liên tục)
            └───┬───────────────────────┬────────────┴──────┬──────────┘
                │ trigger khi có file    │ batch đêm          │ CDC + lịch sử
                ▼                        ▼                    ▼
        ┌───────────────┐      ┌──────────────────┐  ┌────────────────┐
 BRONZE │ OCR + NFC +   │      │ flatten span tree│  │ redact PII     │
        │ sửa dấu VN    │      │ 1 row/span       │  │ (PDPL)         │
        └──────┬────────┘      └────────┬─────────┘  └──────┬─────────┘
               ▼ chunk + embed          ▼                   ▼
        ┌───────────────┐      ┌──────────────────┐  ┌────────────────┐
 SILVER │ Vector index  │      │ eval (held-out)  │  │ lịch sử feature│
        │ + KG mỏng     │      │ DPO (human sig.) │  │ (as_of stamp)  │
        │ (phiên bản)   │      │ → DECONTAMINATE  │  └──────┬─────────┘
        └──────┬────────┘      └────────┬─────────┘         │ ASOF JOIN
               │                        │                    ▼
               ▼                        ▼            ┌────────────────┐
        ┌──────────────────────────────────────┐    │ feature đúng   │
  GOLD  │ RAG/KG phục vụ chatbot  ◄── eval gate │    │ point-in-time  │
        │           ▲                           │    └───────┬────────┘
        └───────────┼───────────────────────────┘           │
                    └──────── FLYWHEEL: trace mới ───────────┘
                              (idempotent replay, backfill an toàn)
```

**Failure semantics:** ingest idempotent theo `doc_id + content_hash` (replay file
trùng bị bỏ qua, như consumer ở lab #5); backfill chạy lại an toàn vì mọi stage
ghi theo hash, không phụ thuộc thứ tự. Khi `quarantine` (OCR fail / thiếu hiệu
lực) vượt ngưỡng → alert cho pháp chế, một file hỏng không chặn cả batch.
