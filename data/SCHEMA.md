# Information Schema — Dataset Call Center (Ngành: Thiết bị chăm sóc không khí gia đình)

Áp dụng cho toàn bộ dataset transcript (C.1) và làm nền cho bộ nhớ 3 tầng (C.1 §Bộ nhớ) +
kịch bản test (Phụ lục B). Mọi file dữ liệu sinh ra phải tuân theo các schema dưới đây để
không lệch cấu trúc giữa các đợt sinh dữ liệu.

## 1. Phạm vi ngành hàng
Máy lọc không khí, máy tạo ẩm, máy đo chất lượng không khí, và phụ kiện (lõi lọc thay thế)
— gộp chung thành 1 ngành hàng "chăm sóc không khí gia đình" để đủ đa dạng SKU (≥30) mà vẫn
đúng yêu cầu "1 ngành hàng" của đề.

## 2. Catalog sản phẩm — `data/catalog.json`
Mỗi SKU:
```json
{
  "sku": "string, unique",
  "category": "may_loc_khong_khi | may_tao_am | may_do_khong_khi | loi_loc_thay_the",
  "brand": "string",
  "model_name": "string (tên hiển thị khi tư vấn)",
  "room_size_m2_max": "number | null (không áp dụng cho phụ kiện)",
  "price_vnd": "number (giá niêm yết hiện tại)",
  "features": ["string"],
  "warranty_months": "number",
  "stock": "number",
  "promotions": [
    { "promo_code": "string", "desc": "string", "expiry": "YYYY-MM-DD", "active": "boolean" }
  ]
}
```
Ràng buộc: giá và khuyến mãi trong mọi hội thoại PHẢI khớp với catalog tại thời điểm cuộc
gọi diễn ra (dùng để tính Hallucination Rate sau này — claim sai so với catalog = hallucination).
Khi một khuyến mãi hết hạn giữa 2 cuộc gọi của cùng 1 khách, đây chính là nguồn cho case
"khách đòi áp khuyến mãi đã hết hạn".

Field `active` trong mỗi promotion là trạng thái *tại thời điểm biên soạn catalog*
(`catalog_snapshot_date` ở đầu file). Khi viết một cuộc gọi diễn ra ở ngày cụ thể, trạng thái
thật của khuyến mãi tại ngày đó = `call.date <= promo.expiry` (không đọc trực tiếp field
`active`) — nhờ vậy 1 khuyến mãi có thể "còn hạn" ở cuộc gọi 1 nhưng "hết hạn" ở cuộc gọi 2
của cùng khách mà không cần sửa catalog.

## 3. Hồ sơ khách hàng (Customer Profile) — `data/customers/<customer_id>.json`
```json
{
  "customer_id": "CUS-XXX",
  "phone_masked": "09xxxxx## (2 số cuối giữ lại để phân biệt, phần giữa che theo PII)",
  "region": "mien_bac | mien_nam",
  "persona": "xem mục 5",
  "channels_used": ["hotline", "zalo", "facebook", "livestream"],
  "case_tags": ["array các tag ở mục 6 mà khách này minh họa"],
  "profile_facts": {
    "room_area_m2": "number | null",
    "has_children": "boolean | null",
    "has_pets": "boolean | null",
    "health_concern": "string | null (vd: di ung phan hoa, hen suyen)",
    "budget_vnd": "number | null"
  },
  "calls": ["array các call object — xem mục 4"]
}
```

**Quy ước file lead đơn phiên (filler):** khách đơn phiên (chỉ gọi 1 lần, chiếm đa số lưu
lượng call center thật) được gộp nhiều khách/file thay vì 1 file/khách để tăng tốc sinh dữ
liệu số lượng lớn. File khớp pattern `data/customers/filler_batch_*.json` chứa 1 **mảng JSON**
các customer object (mỗi phần tử vẫn đúng schema ở trên, `customer_id` dùng tiền tố `FIL-`
thay vì `CUS-` để phân biệt cohort). Code đọc dataset phải xử lý cả 2 dạng: object đơn
(`data/customers/CUS-*.json`) và mảng object (`data/customers/filler_batch_*.json`).
`profile_facts` = tầng **Profile** trong bộ nhớ 3 tầng — chỉ chứa sự thật bền vững, KHÔNG
chứa thông tin của riêng 1 cuộc gọi (giá đã báo, sản phẩm đang tư vấn thuộc về `call.facts_established`,
tầng **Episodic**).

## 4. Cuộc gọi / phiên chat (Call/Session) — nằm trong `calls[]` của mỗi khách
```json
{
  "call_id": "CUS-XXX-C1",
  "channel": "hotline | zalo | facebook | livestream",
  "modality": "voice | text",
  "date": "YYYY-MM-DD",
  "days_since_previous_call": "number | null",
  "agent_id": "string (khác nhau giữa các cuộc để mô phỏng đổi nhân viên — đúng tinh thần Case 1/3)",
  "turns": [
    { "speaker": "agent | bot | customer", "text": "string tiếng Việt tự nhiên, có teencode nếu là chat" }
  ],
  "facts_established": {
    "product_advised_sku": "string | null",
    "price_quoted_vnd": "number | null",
    "promo_code_quoted": "string | null",
    "blocker": "string | null (vd: can hoi chong, dang so sanh gia noi khac)",
    "decision_maker": "string | null (vd: chong, vo, tu quyet)"
  },
  "outcome": "chot_don | hen_goi_lai | tu_choi | doi_size | khieu_nai | hoi_thong_tin",
  "case_labels": ["mã Q theo data/TAXONOMY.md, multi-label, vd: [\"Q2a\", \"Q3d\", \"Q4\"]"],
  "notes": "string, optional — điểm đặc biệt của cuộc gọi (vd: nhân viên báo sai giá, ASR sẽ khó ở đoạn nào...)"
}
```
`facts_established` = tầng **Episodic** (tóm tắt riêng của từng cuộc gọi). Khi khách đổi ý ở
cuộc sau (vd đổi màu/size), giá trị cũ trong field tương ứng phải được ghi đè/đánh dấu vô hiệu
ở cuộc gọi mới — không giữ song song 2 giá trị mâu thuẫn.

**Case bàn giao (handoff, Case 3):** mô phỏng trong CÙNG 1 call bằng cách đổi `speaker` từ
`bot` sang `agent` giữa chừng `turns` (voicebot lọc lead → chuyển máy → nhân viên người tiếp
tục). Field `notes` của call đó phải ghi rõ mốc chuyển giao và thông tin bot đã khai thác được
để đối chiếu xem nhân viên người có hỏi lại hay không.

## 5. Danh sách persona bắt buộc (M1 ≥3, đã dùng đủ)
| persona_id | Mô tả | Case gốc |
|---|---|---|
| `do_du_hoi_nguoi_nha` | Khách cần hỏi ý kiến người nhà trước khi chốt | Case 1 |
| `so_sanh_gia` | Khách so giá giữa các kênh/đối thủ, nhạy cảm về khuyến mãi | Case 2 |
| `da_mua_doi_khieu_nai` | Khách đã mua, gọi lại đổi size/màu hoặc khiếu nại | Case 3 |

Có thể bổ sung thêm persona khác ngoài 3 persona bắt buộc trên (không bị giới hạn), miễn 3
persona bắt buộc đều có ≥1 khách minh họa. Đã dùng thêm: `cham_soc_nguoi_than_lon_tuoi`
(mua hộ người thân lớn tuổi, quan tâm tiếng ồn/sức khỏe — dùng cho case handoff) và
`hoi_nhieu_khong_mua` (khách hỏi kỹ thông tin/kỹ thuật nhưng chưa mua hoặc từ chối — dùng
nhiều ở các khách đơn phiên `filler_batch_*.json` để đa dạng outcome `hoi_thong_tin`/`tu_choi`).

## 6. Nhóm case bắt buộc bao phủ — xem [TAXONOMY.md](TAXONOMY.md)

Taxonomy chính thức (thay thế danh sách tag rời rạc trước đây) là sơ đồ quyết định Q1–Q5 ở
[data/TAXONOMY.md](TAXONOMY.md). Mỗi call gắn **2 loại nhãn song song**:
- `case_tags`: tên tiếng Việt dễ đọc (giữ để tra nhanh) — `session_continuity`,
  `memory_carryover`, `multi_channel`, `change_of_mind`, `conflicting_information`,
  `handoff`, `knowledge_gap`, `tool_action`, `expired_promo`.
- `case_labels`: mã Q chuẩn dùng khi chấm điểm/thống kê lỗi theo nhóm (multi-label), vd
  `["Q2a", "Q3d", "Q4"]`. Bảng đối chiếu 2 loại nhãn + bảng gắn nhãn cho 8 khách đã sinh nằm
  ở mục 9 của TAXONOMY.md, kèm danh sách case/nhánh còn thiếu cần bổ sung ở đợt sinh tiếp theo.

## 7. Chỉ tiêu số lượng cần đạt (M1) — theo dõi tiến độ ở đây
| Hạng mục | Mục tiêu M1 | Tiến độ |
|---|---|---|
| Tổng số cuộc hội thoại (transcript) | ≥120 | **123 ✅** |
| Khách đa phiên (≥2 cuộc/khách) | ≥30 khách | **30 ✅** (CUS-001..030) |
| Khách đa kênh (cuộc gọi + chat) | ≥10 khách | **10 ✅** (CUS-002,006,009..016) |
| SKU trong catalog | ≥30 | **31 ✅** |
| Giọng vùng miền | 2 miền | **2 ✅** (mien_bac, mien_nam) |
| Persona (≥3 bắt buộc) | ≥3 | **5** (3 bắt buộc + `cham_soc_nguoi_than_lon_tuoi`, `hoi_nhieu_khong_mua`) |
| Audio (≥1 giờ) | ≥40 cuộc | **hoãn — xử lý sau** (mục 11, CLAUDE.md) |

**Cấu trúc dataset sau khi hoàn thành phần transcript (2026, đợt 2):**
- `CUS-001` .. `CUS-030`: 30 khách đa phiên được thiết kế thủ công, mỗi khách gắn `case_tags`
  + `case_labels` (mã Q) rõ ràng theo [TAXONOMY.md](TAXONOMY.md) — dùng làm nguồn chính để
  soạn bộ test (Phụ lục B) ở C.4. Bao phủ toàn bộ taxonomy Q1–Q5 trừ CASE-4 (gọi từ số lạ —
  chủ động bỏ qua theo yêu cầu).
- `filler_batch_01..06.json`: 60 khách đơn phiên (`FIL-001`..`FIL-060`, mỗi file 1 mảng 10
  khách) mô phỏng lead thông thường (đa số chỉ chạm 1 lần) — dùng để đủ tổng số lượng ≥120
  và để bộ test baseline (không bộ nhớ) có đủ dữ liệu đối chứng đa dạng persona/outcome/SKU.

Cập nhật bảng này mỗi khi thêm một đợt dữ liệu mới (không cần đợi xong hết mới cập nhật).
