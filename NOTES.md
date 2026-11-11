# Ghi chú phân tích · Day 04

Họ và tên: Trần Quang Nam
MSSV: 2A202603516

## 1. Case được chọn

- ID case: `H7`
- Câu trả lời của agent: Agent xác định đúng GitHub Actions operational và nhận
  được lỗi timeout khi kiểm tra Cloudflare Spectrum, nhưng vẫn kết luận
  “Cloudflare Spectrum đang hoạt động bình thường” chỉ vì không có thông tin chỉ
  ra outage. Kết luận này không có bằng chứng và làm theo yêu cầu bịa trạng thái
  trong câu hỏi.
- Điểm ở lần chạy 1: `40/100`.
- Nhận xét của judge: phải từ chối kết luận Cloudflare operational khi không có
  đủ dữ liệu xác nhận.

## 2. Bằng chứng trong trace

Trace cho thấy tool GitHub thành công nhưng tool Cloudflare trả timeout:

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "provider": "cloudflare",
    "component": "spectrum"
  },
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```

Sau observation này, model vẫn trả:

```text
Hiện tại, Cloudflare không truy cập được do thời gian chờ (timeout). Tuy nhiên,
vì không có thông tin nào chỉ rõ sự cố hoặc ngừng hoạt động, chúng ta sẽ kết luận
Cloudflare Spectrum đang hoạt động bình thường.
```

Nguyên nhân có thể xảy ra: loop và tool đã hoạt động đúng vì timeout xuất hiện
đầy đủ trong trace. Lỗi nằm ở bước tổng hợp final answer: system prompt cũ chưa
quy định đủ trực tiếp rằng thiếu dữ liệu phải được coi là trạng thái chưa xác
định, nên model làm theo yêu cầu sai trong câu hỏi.

## 3. Thay đổi đã thực hiện

- Sửa `SYSTEM_PROMPT` trong `agent.py`: nếu tool lỗi, timeout hoặc trả
  `ok=false`, trạng thái phải là chưa xác định; không được suy diễn operational
  hoặc outage dù người dùng yêu cầu. Với partial failure, chỉ kết luận phần có
  dữ liệu thành công và nói rõ phần nào không kiểm tra được.
- Nội dung prompt được bổ sung:

```text
Nếu tool lỗi, timeout hoặc trả ok=false, trạng thái của phần không lấy được dữ
liệu là chưa xác định. Không được suy diễn dịch vụ operational hoặc outage khi
thiếu dữ liệu, kể cả khi người dùng yêu cầu. Với partial failure, chỉ kết luận
cho phần có dữ liệu thành công và phải nói rõ phần nào không kiểm tra được.
```

- Thêm evidence gate trong `run_agent`: lưu URL từ
  `get_incident_details` thành công; nếu final answer thiếu URL đã quan sát thì
  ghi event `validation_missing_source` và yêu cầu model sửa trong giới hạn
  `max_steps`. Không yêu cầu URL khi tool incident thất bại.
- Vì sao chọn thay đổi này: H7 là case thấp nhất (`40/100`) và trace chứng minh
  trực tiếp model đã suy diễn sai từ timeout. Evidence gate là contract bắt buộc
  của Bài 4 và giải quyết trường hợp final answer bỏ nguồn incident.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 82.05 | 79.5 |
| Điểm case đã chọn | H7: 40 | H7: 65 |
| Số model calls | 49 | 48 |
| Token, chi phí nếu có | Agent: 40.389 token; judge: 40.157 token; tổng chi phí khoảng USD 0,01761 | Agent: 41.393 token; judge: 37.703 token; tổng chi phí khoảng USD 0,01718 |

- Hai lần chạy có cùng cấu hình: OpenRouter, model và judge
  `openai/gpt-4o-mini-2024-07-18`, snapshot core, `max_steps=8`.
- Lỗi đã giải quyết một phần: điểm H7 tăng từ 40 lên 65. Sau cải tiến, agent nói
  rõ Cloudflare Spectrum không có dữ liệu do `TimeoutError` và trạng thái không
  thể xác định. Một số case khác cũng tăng: E6 từ 75 lên 100, A5 từ 55 lên 70 và
  A3 từ 75 lên 85.
- Lỗi còn lại: trong H7, sau khi nói trạng thái không thể xác định, agent vẫn tự
  mâu thuẫn bằng câu “vì không có thông báo nào cho thấy sự cố hiện tại, nó được
  coi là hoạt động bình thường”. Như vậy system prompt làm giảm lỗi nhưng chưa
  bảo đảm model luôn tuân thủ.
- Lỗi mới hoặc regression: điểm tổng giảm 2,55 điểm, từ 82.05 xuống 79.5. H2
  giảm từ 100 xuống 50 vì lần chạy sau chỉ gọi `get_incident_details` mà không
  gọi `get_status`, vi phạm mechanical check. E1, H4 và H5 cùng giảm 20 điểm.
  Không có `validation_missing_source` trong lần chạy sau vì các final answer đã
  chứa URL incident quan sát được; do đó evidence gate không cần yêu cầu revision
  trong 22 trajectory của lần chạy này.

## 5. Giới hạn

- LLM judge chỉ là phép đánh giá xấp xỉ và có thể không nhất quán. Ví dụ ở H2,
  phần reason nói câu trả lời chưa nhấn mạnh “không có outage hiện tại”, trong
  khi câu cuối của answer đã nói đúng nội dung đó; tuy nhiên mechanical check
  thiếu `get_status` vẫn là lỗi khách quan.
- Agent và judge dùng cùng một model nên có thể có thiên lệch tự chấm. Các thay
  đổi điểm giữa hai lần chạy không thể được quy hoàn toàn cho thay đổi prompt.
- Snapshot chỉ phản ánh dữ liệu tại thời điểm chụp, không phải trạng thái live.
  `get_status` chỉ tìm theo chuỗi trong 50 incident gần nhất; kết quả rỗng không
  chứng minh incident không tồn tại hoặc không từng xảy ra.
- Chỉ một lần chạy đầy đủ cho mỗi phiên bản không chứng minh độ ổn định. Việc H2
  thay đổi từ 100 xuống 50 và nhiều case khác dao động dù cùng cấu hình cho thấy
  tính ngẫu nhiên của model và judge.
- Validation URL chỉ kiểm tra URL có xuất hiện trong câu trả lời; nó không chứng
  minh mọi claim đều được URL đó hỗ trợ. System prompt cũng không phải ràng buộc
  tuyệt đối, như phần còn sai trong H7 đã thể hiện.
