# Scenario Taxonomy — Sơ đồ quyết định phân loại case

Nguồn: phần phân công nghiên cứu case (Thy — Analyst/Research), đối chiếu với mục 6 của
[CLAUDE.md](../CLAUDE.md). Đây là bản taxonomy chính thức dùng để gắn nhãn **mọi cuộc gọi**
trong dataset (`data/customers/*.json`) và để thiết kế bộ test (`data/scenarios/*.json`,
Phụ lục B). Thay thế cách gắn `case_tags` đơn nhãn ban đầu bằng **multi-label theo mã Q**
ở dưới — một cuộc gọi có thể đồng thời thuộc nhiều nhánh (ví dụ CASE-9, CASE-18).

## 1. Sơ đồ quyết định tổng thể

Xét 1 lượt tương tác của khách hàng (1 cuộc gọi hoặc 1 phiên chat):

```
Q1. Đây có phải LẦN ĐẦU khách liên hệ không?
│
├─ CÓ (khách mới) ──────────────► bỏ qua Q2–Q3, đi thẳng xuống Q4
│
└─ KHÔNG (đã liên hệ trước) ────► đi tiếp Q2

Q2. Giữa lần trước và lần này, ngữ cảnh bị NGẮT bởi yếu tố gì?
    (có thể ngắt bởi nhiều yếu tố cùng lúc)
│
├─ (a) Khoảng cách THỜI GIAN     → Q2a Session Continuity / Multi-session
├─ (b) KÊNH liên hệ khác nhau    → Q2b Multi-channel
└─ (c) NGƯỜI/BOT xử lý khác nhau → Q2c Handoff

Q3. Với mỗi fact mang từ lần trước sang, nó còn ĐÚNG không?
│
├─ (a) Đúng, dùng lại bình thường           → Q3a Memory Carry-over (baseline, KHÔNG phải lỗi)
├─ (b) Khách TỰ đổi ý                       → Q3b Change of Mind
├─ (c) 2 nguồn nói KHÁC nhau, không ai đổi  → Q3c Conflicting Information
└─ (d) Đã quá HẠN hợp lý (TTL)              → Q3d Expiry

Q4. (áp dụng cho MỌI trường hợp — kể cả khách mới ở Q1)
    Mỗi khi agent nói ra 1 thông tin kiểm chứng được hoặc thực hiện 1 hành động,
    có tool call THẬT đứng sau không?
│
├─ Có, đúng dữ liệu tool trả về                    → hợp lệ
└─ Không có / tool lỗi nhưng agent vẫn trả lời      → VI PHẠM (Hallucination)

Q5. (độc lập, KHÔNG nằm trong luồng Q1–Q4 — không liên quan bộ nhớ khách hàng)
    Câu hỏi của khách có nằm trong phạm vi tri thức hệ thống (catalog, chính sách) không?
│
├─ Có → trả lời bình thường (qua Q4 để kiểm chứng)
└─ Không → Q5 Knowledge Gap — agent phải thừa nhận không biết, không bịa
```

**Logic phân tầng:** Q1–Q2 trả lời "ngữ cảnh có đứt gãy không, đứt bởi cái gì" (tiền đề).
Q3 chỉ có ý nghĩa sau khi đã xác định có ngữ cảnh cũ để mang sang (hệ quả của Q2) — vì vậy
Change of Mind / Conflicting Info / Expiry là các khả năng xảy ra **sau** Q2, không ngang
hàng với Session Continuity / Multi-channel. Q4 là bước kiểm chứng, áp dụng bất kể Q2/Q3 trả
lời gì. Q5 tách hẳn khỏi luồng vì hỏi về tri thức hệ thống, khác bản chất với bộ nhớ khách hàng.

## 2. Q2 — Ngữ cảnh bị ngắt bởi yếu tố gì

### Q2a. Khoảng cách thời gian — Session Continuity / Multi-session
Bao gồm cả "khách gọi lại 2 lần" (M1) và "≥3 lần" (M2).

| Field | Ghi chú |
|---|---|
| `session_count` | số lần liên hệ đã có (≥2) |
| `session_gap_days` | khoảng cách giữa các lần |
| `call_brief` | sinh trước khi bắt máy: khách là ai, đã tư vấn gì, rào cản gì, đề xuất tiếp theo |
| `persona` | do dự / so giá / khiếu nại / hỏi nhiều không mua / hết kiên nhẫn (chỉ ở session_count ≥3) |

Case tiêu biểu: CASE-1 (2 lần, do dự, chốt đúng SKU/giá cũ) · CASE-2/M2 (3 lần, so giá rồi
hết kiên nhẫn — đo Calls-to-Close) · CASE-3 (lần liên hệ không có fact mới, agent không được
tự suy diễn) · CASE-4/khó (gọi từ số lạ, hệ thống không nhận diện được).

### Q2b. Kênh liên hệ khác nhau — Multi-channel
| Field | Ghi chú |
|---|---|
| `identity_keys` | {phone, zalo_id, fb_id} |
| `identity_resolved` | đã gộp về cùng 1 khách chưa |
| `channel_log[]` | {channel, timestamp, fact_or_quote} |

Case tiêu biểu: CASE-5 (Fanpage báo giá, hotline phải nhất quán) · CASE-6/khó (identity không
resolve được — xử lý như khách mới, không bịa rằng đã biết khách).

### Q2c. Người/bot xử lý khác nhau — Handoff
| Field | Ghi chú |
|---|---|
| `handoff_type` | bot_to_human / shift_to_shift / agent_resignation / scheduled_callback |
| `handoff_brief` | intent, facts đã khai thác, câu hỏi treo, hành động đề xuất |
| `receiving_party_ack` | bên nhận có thực sự dùng brief hay bỏ qua |

Case tiêu biểu: CASE-7 (voicebot 4 phút → chuyển máy → nhân viên dùng đúng brief) · CASE-8
(nhân viên nghỉ việc, đơn hàng vẫn truy xuất được).

**Ví dụ ngắt bởi NHIỀU yếu tố cùng lúc:** CASE-9 — ca sáng hẹn "chiều gọi lại", ca chiều là
người khác trực → vừa Q2a (khoảng cách thời gian) vừa Q2c (người xử lý khác nhau). Không cần
chọn 1 nhãn — liệt kê cả 2.

## 3. Q3 — Fact mang sang còn đúng không (chỉ áp dụng sau khi đã qua Q2)

### Q3a. Đúng, dùng lại bình thường — Memory Carry-over (baseline)
Hành vi đúng cần có, không phải lỗi — nhưng vẫn phải test vì dễ sai theo 2 hướng: quên dùng
(như không có bộ nhớ) hoặc dùng sai chỗ (đọc vẹt).

Case tiêu biểu: CASE-10 (fact phải xuất hiện trong tham số tool, không chỉ trong lời nói) ·
CASE-11/khó (agent đọc dồn 5 fact trong câu chào đầu → vẫn bị tính vi phạm dù không "hỏi lại").

### Q3b. Khách tự đổi ý — Change of Mind
| Field | Ghi chú |
|---|---|
| `slot_changed`, `old_value`, `new_value` | |
| `change_scope` | trong cùng lượt / cùng session / khác session |
| `invalidation_required` | giá trị cũ bắt buộc bị vô hiệu ngay, không tồn tại song song |

Case tiêu biểu: CASE-12 ("thôi lấy màu trắng" cùng lượt) · CASE-13/khó (session 1 chốt model
X, session 2 đổi hẳn sang Y — order.create session 2 phải dùng Y) · CASE-14 (sau đổi ý, session
sau vô tình hỏi lại/dùng giá trị cũ → fail test).

### Q3c. 2 nguồn nói khác nhau, không ai chủ động đổi — Conflicting Information
| Field | Ghi chú |
|---|---|
| `conflict_type` | tự mâu thuẫn trong cùng cuộc / giữa 2 kênh (=Q2b) / khách nói khác hệ thống ghi |
| `resolution_rule` | nguồn nào được ưu tiên khi xử lý |
| `disclosed_to_customer` | agent có giải thích mâu thuẫn hay âm thầm chọn 1 giá trị |

Case tiêu biểu: CASE-15/khó (2 kênh báo giá khác nhau — agent phải dùng `pricing.get_quote`
làm nguồn sự thật và giải thích cho khách) · CASE-16 (khách khẳng định điều không có trong hệ
thống — nguy cơ memory poisoning) · CASE-17 (khách tự mâu thuẫn trong cùng cuộc — agent phải
hỏi lại xác nhận, không tự chọn đại).

### Q3d. Đã quá hạn hợp lý — Expiry (TTL)
| Field | Ghi chú |
|---|---|
| `ttl_rule` | vd: địa chỉ >6 tháng, `promo_expiry` |
| `expected_behavior` | hỏi lại xác nhận, không phải hỏi mở |

Case tiêu biểu: CASE-18 **(bắt buộc theo đề)** — khuyến mãi hết hạn khi khách gọi lại, không
báo lại giá cũ, giải thích khéo, không bịa · CASE-19 (địa chỉ lưu quá 6 tháng, hỏi lại xác nhận).

*Lưu ý: CASE-18 vừa đi qua Q2a vừa rơi vào Q3d — đây là case "khó nhất" trong nhóm Session
Continuity mà đề bài nhắc ở M2. **`CUS-001-C2` trong dataset hiện tại chính là case này.***

## 4. Q4 — Kiểm chứng bằng tool (áp dụng cho MỌI case, kể cả khách mới ở Q1)

Không phải 1 nhóm case riêng — là bước kiểm tra cộng thêm vào bất kỳ case nào ở mục 2–3.

| Field | Ghi chú |
|---|---|
| `tool_name` | crm.get_customer / catalog.search / inventory.check / order.create / pricing.get_quote / schedule.callback / order.status |
| `claim_grounded` | claim của agent có tool call tương ứng đứng sau không |
| `tool_error_type` | timeout / malformed_schema / stale_price / out_of_stock |

Case tiêu biểu: CASE-20 (chuỗi 3 tool đúng thứ tự, giá khớp catalog.search) · CASE-21/khó
(inventory hết hàng ngay lúc chốt — đề xuất thay thế, không tạo đơn cho SKU hết hàng) ·
CASE-22/khó (agent báo giá không có tool call đứng sau → vi phạm guardrail dù giá tình cờ đúng).

## 5. Q5 — Knowledge Gap (tách riêng khỏi luồng Q1–Q4)

Không liên quan bộ nhớ khách hàng — là giới hạn tri thức chung của hệ thống.

| Field | Ghi chú |
|---|---|
| `gap_reason` | ngoài catalog / ngoài chính sách / câu hỏi nhạy cảm (y tế, kỹ thuật) |
| `logged_as_gap`, `added_to_regression_test` | liên kết cơ chế cải tiến C.5 (Knowledge Gap Loop) |

Case tiêu biểu: CASE-23 **(bắt buộc theo đề)** — câu hỏi ngoài tài liệu, agent phải nói không
biết, không bịa · CASE-24 (hỏi SKU không tồn tại, catalog.search trả rỗng, trả lời trung thực)
· CASE-25/M2 (cùng 1 gap hỏi lại sau khi đã bổ sung tài liệu — kỳ vọng trả lời đúng).

*`CUS-005` trong dataset hiện tại chính là cặp CASE-23 → CASE-25 (vòng 0 → vòng 1 của Knowledge
Gap Loop, dùng để chứng minh C.5).*

## 6. Ví dụ áp dụng đầy đủ (case chị Hoa trong đề bài = `CUS-001`)

- Q1: Không phải lần đầu → đi tiếp Q2.
- Q2: Ngắt bởi khoảng cách thời gian (15 ngày) → **Q2a**.
- Q3: Khuyến mãi đã hết hạn khi gọi lại → **Q3d** (biến thể khó, không phải Q3a đơn giản).
- Q4: Giá xác nhận lại phải đến từ `pricing.get_quote`, không phải nhớ lại từ lượt trước.
- Q5: Không áp dụng.
- → Nhãn đầy đủ của `CUS-001-C2`: **Q2a + Q3d + Q4**.

## 7. Phân bổ bộ test M1 (≥20 kịch bản đa phiên + ≥5 kịch bản khó) — dùng làm blueprint cho `data/scenarios/`

| Nhánh | Case cơ bản | Case khó | Case đại diện |
|---|---|---|---|
| Q2a — Session/Multi-session | 4 | 1 | CASE-1,2,3,4 / CASE-18 (giao Q3d) |
| Q2b — Multi-channel | 3 | 1 | CASE-5,6 / CASE-15 (giao Q3c) |
| Q2c — Handoff | 2 | 1 | CASE-7,8 / CASE-9 (giao Q2a) |
| Q3a — Carry-over | 3 | 1 | CASE-10 / CASE-11 |
| Q3b — Change of Mind | 2 | 1 | CASE-12 / CASE-13 (bắt buộc theo đề) |
| Q3c — Conflicting Info | 1 | 1 | CASE-16 / CASE-17 |
| Q3d — Expiry | — | 1 | CASE-18 (bắt buộc, đã tính ở Q2a) |
| Q4 — Tool/Action | 2 | 1 | CASE-20 / CASE-21,22 |
| Q5 — Knowledge Gap | 1 | 1 | CASE-24 / CASE-23 (bắt buộc theo đề) |
| **Tổng** | **20** | **8** | vượt ngưỡng ≥20 + ≥5 |

## 8. Quyết định cho các câu hỏi mở (đề xuất — có thể chỉnh lại sau khi team debate)

Đây là câu trả lời tạm dùng để không chặn tiến độ sinh dữ liệu; đổi lại bất cứ lúc nào nếu
team quyết khác, chỉ cần sửa ở đúng 1 chỗ này.

1. **TTL từng slot (Q3d):** `promo_expiry` = đúng ngày hết hạn ghi trong catalog (không có
   độ trễ) · `address` = 180 ngày (6 tháng) · `purchase_intent`/`blocker` = 30 ngày (sau đó
   hỏi lại xem còn nhu cầu không, không tính là hỏi thừa) · `budget_vnd`, `room_area_m2`,
   `has_children`, `has_pets`, `health_concern` = không hết hạn (thuộc tầng Profile, coi là
   bền vững trong phạm vi dự án).
2. **`resolution_rule` khi 2 kênh lệch nhau (Q3c):** nguồn thắng luôn là **tool trả về tại
   thời điểm hỏi** (`pricing.get_quote`/`catalog.search`), không bao giờ là lời agent kênh
   khác đã nói trước đó. Nếu tool và lời khách nói lệch nhau → agent phải giải thích công
   khai cho khách (`disclosed_to_customer = true`), không âm thầm chọn.
3. **Case bắt buộc có audio thật:** hoãn quyết định — thuộc phần audio đã hoãn xử lý (mục 11
   [CLAUDE.md](../CLAUDE.md)). Khi tới lúc làm audio, ưu tiên các case đại diện mỗi nhánh ở
   mục 7 trước (đủ đa dạng để báo cáo WER/CER có ý nghĩa), không cần toàn bộ case.
4. **Case đi qua nhiều nhánh (CASE-9, CASE-18), tránh đếm trùng:** script đánh giá tính
   `RQR`/`CCR`/`TSR`/`HR` ở cấp **câu hỏi/claim/kịch bản**, không ở cấp "nhãn case" — một
   case gắn 2 nhãn (vd Q2a+Q3d) vẫn chỉ là 1 kịch bản, đóng góp 1 lần vào mẫu số của mỗi chỉ
   số liên quan. Nhãn Q-code chỉ dùng để phân tích lỗi theo nhóm nguyên nhân (mục C.6 báo cáo
   "phân tích ≥10 case sai"), không nhân đôi điểm số.
5. **Q4 (Tool/Action) có tách chỉ số riêng khỏi Hallucination Rate không:** không tách — giữ
   đúng theo đề bài (chỉ 5 chỉ số bắt buộc M1, không tự thêm). `claim_grounded=false` được
   tính thẳng vào tử số của Hallucination Rate (claim không có tool đứng sau = claim sai).
   `tool_name`/`tool_error_type` chỉ dùng để debug/phân tích lỗi trong báo cáo, không phải KPI.
6. **`case_labels` cho phép multi-label từ đầu:** **có** — mỗi call mang 1 mảng `case_labels`
   (mã Q, vd `["Q2a", "Q3d", "Q4"]`) thay vì 1 nhãn chính. Lý do: chính tài liệu này chỉ ra
   CASE-9/CASE-18 vốn dĩ đa nhãn; ép về 1 nhãn sẽ làm mất thông tin quan trọng cho phần phân
   tích lỗi ở C.6. `case_tags` cũ (tên tiếng Việt dễ đọc) vẫn giữ song song để dễ hiểu khi đọc
   nhanh, `case_labels` là mã chuẩn dùng khi chấm điểm/thống kê.

## 9. Đối chiếu `case_tags` cũ ↔ `case_labels` mới (áp dụng cho dataset hiện có)

| `case_tags` cũ | `case_labels` (Q-code) tương ứng |
|---|---|
| `session_continuity` | Q2a |
| `memory_carryover` | Q3a |
| `multi_channel` | Q2b |
| `conflicting_information` | Q3c |
| `change_of_mind` | Q3b |
| `handoff` | Q2c |
| `knowledge_gap` | Q5 |
| `tool_action` | Q4 |
| `expired_promo` (tự đặt riêng, không có trong taxonomy gốc) | Q3d |

Bảng gắn nhãn lại cho 8 khách đã sinh (mức **call**, không phải mức khách — vì Q-code áp
dụng cho từng lượt liên hệ):

| Call | case_labels |
|---|---|
| CUS-001-C1 | Q1 (khách mới) + Q4 |
| CUS-001-C2 | Q2a + Q3d + Q4 |
| CUS-002-C1 | Q1 + Q4 |
| CUS-002-C2 | Q2a + Q2b + Q3a + Q4 |
| CUS-002-C3 | Q2a + Q2b + Q3c + Q4 |
| CUS-003-C1 | Q1 + Q4 |
| CUS-003-C2 | Q2a + Q3a + Q4 |
| CUS-004-C1 | Q1 + Q2c + Q4 (bot→người trong cùng call) |
| CUS-004-C2 | Q2a + Q2c + Q3a + Q4 |
| CUS-005-C1 | Q1 + Q5 |
| CUS-005-C2 | Q2a + Q3a + Q4 + Q5 (Q5 vì trả lời gap từ C1) |
| CUS-006-C1 | Q1 + Q4 |
| CUS-006-C2 | Q2a + Q2b + Q3a + Q4 |
| CUS-007-C1 | Q1 + Q4 |
| CUS-007-C2 | Q2a + Q3b + Q4 |
| CUS-008-C1 | Q1 + Q4 |
| CUS-008-C2 | Q2a + Q3a + Q4 |
| CUS-008-C3 | Q2a + Q3a + Q4 |

**Khoảng trống nhánh cần bổ sung ở các đợt sinh tiếp theo** (chưa có case nào minh họa):
CASE-3 (liên hệ không có fact mới), CASE-4 (gọi từ số lạ), CASE-6 (identity không resolve
được), CASE-9 (ngắt kép thời gian+người trực), CASE-11 (agent đọc vẹt — hành vi SAI cần bắt),
CASE-13 (đổi hẳn model khác session — *đã có ở CUS-007 nhưng khác session ngay từ đầu, có thể
cần thêm 1 case đổi ý xảy ra ở session ≥3*), CASE-16 (khách khẳng định điều không có thật),
CASE-17 (khách tự mâu thuẫn cùng cuộc), CASE-19 (địa chỉ hết hạn TTL), CASE-21/22 (tool lỗi/hết
hàng), CASE-2 M2 (3 phiên, đo Calls-to-Close, không bắt buộc M1 nhưng dễ làm luôn nếu sinh
customer ≥3 phiên).
