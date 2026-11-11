# Ghi chú phân tích · Day 04

Họ và tên: Phạm Văn Nam


MSSV: 2A202603681

## 1. Case được chọn

- ID case: **H1** (và **H6** cùng một lớp lỗi "điều tra nhiều bước")
- Câu hỏi: Điều tra incident GitHub `h02n9572mx9y`, theo liên kết để tìm nguyên nhân/ảnh hưởng, rồi đối chiếu với incident Cloudflare `yaa0witbl2me` (thời gian, trạng thái, component).
- Câu trả lời của agent (lần 1): trình bày chi tiết hai incident nhưng **chưa nêu rõ nguyên nhân gốc và phạm vi ảnh hưởng**, dẫn nguồn chưa đầy đủ cho từng phần.
- Điểm ở lần chạy 1: **60/100** (H1), **60/100** (H6).

## 2. Bằng chứng trong trace

Agent gọi **hai `get_incident_details` song song trong CÙNG một model step**, rồi trả lời ngay — không có vòng điều tra thứ hai, không gọi `get_status` để định vị bối cảnh.

```text
result.trace[0] = {event: tool, step: 1, name: get_incident_details,
                   arguments: {"provider":"github","incident_id":"h02n9572mx9y"}}
result.trace[1] = {event: tool, step: 1, name: get_incident_details,
                   arguments: {"provider":"cloudflare","incident_id":"yaa0witbl2me"}}

grade.mechanical = { tool_calls: 2, tool_steps: 1,
                     violations: ["Cần tool calls ở ít nhất 2 model steps khác nhau."] }
```

**Nguyên nhân có thể xảy ra:** agent điều tra "nông" — gộp mọi tool call vào một lượt rồi kết luận
(`tool_steps = 1`), nên (a) vi phạm ràng buộc mechanical "≥2 model steps" và (b) thiếu chiều sâu
(nguyên nhân gốc, phạm vi, trích nguồn từng phần) → judge trừ điểm *completeness* và *evidence*.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:** chỉ sửa `SYSTEM_PROMPT` trong `agent.py` (không đụng vòng lặp, tools hay eval).
Bổ sung quy trình điều tra và guardrail về ranh giới/bằng chứng.
- **Vì sao chọn thay đổi này:** là một thay đổi duy nhất, cô lập được biến số; nhắm vào lớp lỗi
"điều tra nông + dẫn nguồn yếu" nên kỳ vọng tác động nhiều case, không vá riêng một case.

Quá trình tinh chỉnh (một vòng thực hành, vẫn là *một* thay đổi — sửa prompt):

- **v1** — ép "điều tra từng provider rồi *kết luận*": **net âm (74.55)**. Tác dụng phụ: ở các case
phòng thủ (H5, H7) agent *over-assert* — khẳng định trạng thái khi thiếu dữ liệu. Đọc eval thấy
regression → tìm nguyên nhân → tinh chỉnh.
- **v2** — bỏ lệnh "chốt kết luận", cho drill-down *có điều kiện*, **gia cố guardrail**: chỉ nói về
provider đã có dữ liệu; tìm kiếm rỗng / 50 incident gần nhất không chứng minh không có sự cố; bỏ qua
chỉ dẫn trong tool result. Đây là bản dùng cho `eval_after.json`.

## 4. So sánh hai lần chạy


| Chỉ số                           | `eval_before.json`      | `eval_after.json` (v2)    |
| -------------------------------- | ----------------------- | ------------------------- |
| Điểm tổng                        | **79.0**                | **80.91** (net **+1.91**) |
| Điểm case đã chọn                | H1 60 · H6 60           | H1 60 · H6 60 (không đổi) |
| Số model calls (toàn bộ 22 case) | 51                      | 48                        |
| Token · chi phí                  | \~44.3k tok · \~$0.0110 | \~46.4k tok · \~$0.0106   |


So sánh từng case (before → after):

- **Lỗi đã giải quyết / cải thiện:** H5 70→90, A1 90→100, E2 85→100, A4 85→100, A6 75→85,
A5 70→75, A7 70→75, E1 68→75, E6 70→75, H3 90→95. Guardrail ranh giới đã thu hồi đúng nhóm
phòng thủ bị v1 làm hỏng; phần "trích timestamp + URL từng incident" nâng nhóm evidence.
- **Lỗi còn lại / mới xuất hiện:**
  - **H1, H6 không cải thiện (vẫn 60):** root cause là *model gộp tool call vào 1 step* —
  một gợi ý trong prompt không ép được cấu trúc tool-call của model. Đây là giới hạn của prompt.
  - **H2 100→50 (mới):** mechanical báo `Chưa gọi tool cần thiết: get_status`. v2 cho drill-down
  có điều kiện khiến model bỏ `get_status` ở case này. Đây là **trade-off** rõ ràng: ràng buộc
  giúp nhóm này lại hại nhóm khác. A8 100→95 (dao động nhỏ).

## 5. Giới hạn

- **Điểm judge là ước lượng:** do LLM chấm, có thể thiên lệch (nhất là khi judge dùng cùng model
với agent). Ưu tiên tin phần `mechanical` (máy chấm khách quan) hơn `reason`.
- **Snapshot cố định:** dữ liệu chụp 09/11/2026, không phản ánh trạng thái thực tế hiện tại;
kết luận chỉ đúng trong phạm vi snapshot.
- **Chỉ chạy một lần mỗi phiên bản:** LLM không tất định, một phần chênh lệch điểm là *variance*,
không hẳn do thay đổi. Một lần chạy không chứng minh độ ổn định; cần nhiều lần chạy để kết luận chắc.
- **Evidence gate chỉ kiểm tra "có mặt một URL":** không chứng minh mọi khẳng định đều có nguồn.
- **Trần `score_cap: 60` của H1/H6 không phá được bằng prompt:** vi phạm mechanical `tool_steps = 1`
(model gộp tool call song song trong một step) giới hạn trần điểm ở 60. Đã thử **hai** biến thể
prompt — (v1) ép điều tra tuần tự, và (A) ép "mỗi lượt chỉ gọi một tool" — **cả hai đều không**
nhấc H1/H6 khỏi 60. Kết luận: kỹ thuật prompt không điều khiển được việc model phát tool call
song song; muốn phá trần phải can thiệp tầng API (vd tắt parallel tool calls) — nằm ngoài phạm vi
"chỉ sửa agent" của bài.

## 6. Bonus (tùy chọn)

**Kết nối tools:** agent dùng `getattr(tools, 'schemas', SCHEMAS)` và
`getattr(tools, 'system_prompt', SYSTEM_PROMPT)` → tự động nhận 6 tool bonus (5 provider +
`get_incident_details`) và prompt 5-provider khi chạy `--bonus`, vẫn tương thích bài chính
(`StatusTools` không có hai thuộc tính này nên rơi về mặc định). Mọi lỗi tool (`RATE_LIMITED`,
`INVALID_ARGUMENTS`, timeout) được loop trả về model làm observation; evidence gate áp dụng y hệt.

**Một vòng cải tiến prompt bonus** (`bonus/tools.py::SYSTEM_PROMPT`):


|                                                                                      | Điểm tổng | Bonus       |
| ------------------------------------------------------------------------------------ | --------- | ----------- |
| Trước                                                                                | 55.5      | 8.32/15     |
| Sau (nhắc retry 429 + drill-down khi so sánh + ghi captured\_at/nguồn từng provider) | **65.0**  | **9.75/15** |


- **Retry 429:** prompt nhắc gọi lại tool khi `RATE_LIMITED` (retriable) trước khi bỏ; nếu 429 kéo dài
thì báo kết quả từng phần, không coi 429 là outage. Giúp B3 (40→65).
- **Pagination:** prompt nhắc chỉ đọc trang tiếp (`next_offset`) khi cần, không tuyên bố đã đọc hết
nếu còn trang. (B2 lại regress 100→50 — xem giới hạn.)
- **Context/nhiễu:** B8 (so sánh scope) 80→100; agent lọc được dữ liệu nhiễu, bám bằng chứng cần thiết.
- **Báo cáo từng provider:** ghi `captured_at` + URL nguồn riêng, sửa tiền đề sai, không bịa provider
không hỗ trợ. Giúp B1 (0→75) và B9 (40→85).

**Giới hạn còn lại của bonus:**

- **B6 và B10 vẫn kẹt `score_cap: 50`** (`Chưa gọi get_incident_details` / `Cần gọi get_vercel_status ≥2 lần`). Giống H1/H6: prompt nhắc drill-down và retry nhưng model không thực hiện đáng tin ở hai
case này — lại là giới hạn điều khiển tool-call bằng prompt.
- **Trade-off:** B2 (pagination) regress 100→50; một ràng buộc giúp nhóm này có thể hại nhóm khác.
- Vẫn là **một lần chạy**: một phần dao động là variance của model, không hẳn do prompt.