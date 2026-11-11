# Ghi chú phân tích · Day 04

Họ và tên: Ngô Chí Vĩnh
MSSV: 2A202603724

Cấu hình cả hai lần chạy (khối `config` giống hệt nhau): agent `openai/gpt-4o-mini`, judge `openai/gpt-4o-mini`, `max_steps = 8`, dataset `core` (22 case), snapshot 09/11/2026.

## 1. Case được chọn

- ID case: **H1**. Câu hỏi: điều tra incident GitHub `h02n9572mx9y`, **theo liên kết do incident cung cấp** để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare `yaa0witbl2me`, dẫn nguồn cho từng bên.
- Câu trả lời của agent: agent chỉ tóm tắt hai incident `h02n9572mx9y` và `yaa0witbl2me`. Câu trả lời có trích update *"Please refer to the combined summary in this related incident"* nhưng **không đọc incident được dẫn tới** (`xr3myzj7ms51`), nên thiếu phần nguyên nhân và ảnh hưởng mà câu hỏi yêu cầu.
- Điểm ở lần chạy 1: **60/100**. Judge cho factuality 30, completeness 15, evidence 10, boundaries 5. Kiểm tra bằng code áp trần 60 với vi phạm: *"Cần tool calls ở ít nhất 2 model steps khác nhau."*

## 2. Bằng chứng trong trace

Trích từ `eval_before.json`, case H1 (`result.trace`):

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
step 1 get_incident_details {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
    - resolved: "Please refer to the combined summary in this related incident:
                 https://www.githubstatus.com/incidents/xr3myzj7ms51"
    - monitoring: "The degradation affecting Actions has been mitigated. ..."
    - investigating: "We are investigating reports of degraded availability for Actions"
step 1 get_incident_details {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'}
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}
{'event': 'final', 'step': 2}

grade.mechanical: {'violations': ['Cần tool calls ở ít nhất 2 model steps khác nhau.'],
                   'score_cap': 60, 'tool_calls': 2, 'tool_steps': 1}
```

Nguyên nhân có thể xảy ra:

- Tool result **đã có** link tới incident gộp `xr3myzj7ms51`, nhưng model (`gpt-4o-mini`) coi hai incident đọc ở step 1 là đủ và trả lời ngay ở step 2.
- Lỗi nằm ở **loop**, không nằm ở dữ liệu: loop chấp nhận mọi câu trả lời có `finish == 'stop'` mà không kiểm tra agent đã đọc đủ incident được dẫn tới chưa.
- Prompt gốc cũng không nói gì về việc theo liên kết. Tuy vậy, thử nghiệm ở mục 3 cho thấy chỉ sửa prompt là chưa đủ.
- Case H6 có cùng lỗi (đọc `h02n9572mx9y` nhưng không đọc `xr3myzj7ms51`) và cũng bị trần 60.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: chỉ sửa `agent.py`. `SYSTEM_PROMPT` giữ nguyên bản gốc của đề.
  1. **Cải tiến cho H1: kiểm tra incident được tham chiếu.**
     - Hàm `_referenced_incidents` tìm link dạng `https://www.<host>/incidents/<id>` trong các update của mỗi lần `get_incident_details` thành công. Host lấy từ `HOSTS` trong `status_tools`, không hard-code incident ID.
     - Trước khi chấp nhận câu trả lời cuối, nếu còn incident được tham chiếu mà chưa đọc, loop ghi event `validation_unread_reference` vào trace, thêm một message yêu cầu model gọi `get_incident_details` cho incident đó, rồi chạy tiếp.
     - Chỉ nhắc **một lần** mỗi run, để không lặp vô hạn. Lần nhắc tính vào `max_steps`.
  2. **Kiểm tra URL nguồn** (contract bắt buộc trong `tests/test_validation.py`):
     - Lưu URL (`url` hoặc `host/incidents/<id>`) của mỗi lần `get_incident_details` thành công.
     - Nếu câu trả lời cuối không chứa URL nào trong số đó, ghi `validation_missing_source` và yêu cầu model sửa trong giới hạn `max_steps`. Hết lượt thì trả `max_steps`, không trả `completed`.
     - Không đòi URL khi chưa có incident nào được đọc thành công.
  3. Ghi chú về hạ tầng (đã có từ **trước** `eval_before.json`, nên hai lần chạy dùng cùng cơ chế):
     - Hai lần chạy eval đầu tiên bị `null`, vì 2/22 case bị *"The read operation timed out"* khi gọi OpenAI (một lần ở phía agent, một lần ở phía judge).
     - Mình thêm retry có giới hạn trong `agent.py` cho lỗi provider tạm thời: timeout, mất kết nối, HTTP 429/5xx. Tối đa 2 lần, chờ 2 giây rồi 5 giây. Lỗi cố định như HTTP 401 không retry.
     - Cả hai lần chạy chính thức không có event `provider_retry` nào.
- Vì sao chọn thay đổi này:
  - Lỗi H1 có bằng chứng rõ trong trace, và được chấm bằng code (trần 60, `min_tool_steps = 2`), không chỉ dựa vào đánh giá của judge.
  - **Thử nghiệm trước đó (chạy riêng `--case`, không lưu file nộp):** thêm vào `SYSTEM_PROMPT` hướng dẫn "đọc tiếp incident được tham chiếu". Kết quả: H6 sửa được, nhưng **H1 vẫn 60 điểm và vẫn vi phạm "2 model steps"**, vì model không luôn làm theo prompt. Vì vậy mình bỏ thay đổi prompt và chuyển sang kiểm tra bằng code trong loop, để chỉ còn một thay đổi.
  - Unit test: `python -m unittest discover -s tests -v` pass 37/37.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 75.23 (22/22 case được chấm) | 82.05 (22/22 case được chấm) |
| Điểm case đã chọn (H1) | 60 (bị trần 60, 1 vi phạm) | 75 (không vi phạm) |
| Số model calls (agent, 22 case) | 51 | 56 |
| Token, chi phí nếu có | Agent 42.342 token (32.305 prompt + 10.037 completion); judge 41.436 token | Agent 57.596 token (46.232 prompt + 11.364 completion); judge 43.158 token |

Provider không trả chi phí trong response. Ước tính theo giá niêm yết của `gpt-4o-mini` (0,15 USD / 1M token input, 0,60 USD / 1M token output, cần đối chiếu lại bảng giá), riêng phần agent tốn khoảng 0,011 USD trước và 0,014 USD sau.

Trace của H1 ở lần chạy 2:

```text
step 1 get_incident_details github/h02n9572mx9y ; get_incident_details cloudflare/yaa0witbl2me
step 2 model muốn trả lời -> {'event': 'validation_unread_reference', 'unread': 'github/xr3myzj7ms51'}
step 3 get_incident_details github/xr3myzj7ms51 ; get_incident_details github/h02n9572mx9y (đọc lại, thừa)
step 4 final, có dẫn https://stspg.io/pjgptg3cfc6n
```

- Lỗi đã giải quyết:
  - **H1:** agent đọc thêm `xr3myzj7ms51` và nêu được ảnh hưởng (Actions, Pages, Webhooks, Code Scanning, Copilot coding agent bị chậm trong hai khoảng thời gian ngày 04/11). Hết vi phạm "2 model steps", điểm tăng từ 60 lên 75.
  - **H6** (cùng nguyên nhân): event `validation_unread_reference` kích hoạt, agent đọc `xr3myzj7ms51`, hết vi phạm, điểm tăng từ 60 lên 90.
  - **H5:** thay đổi cũng kích hoạt (75 lên 80). Cả lần chạy 2 chỉ có 3 case kích hoạt thay đổi: H1, H5, H6.
  - Phần lớn token tăng thêm nằm ở 3 case này: H1, H5, H6 cộng lại tốn 7.815 token trước và 24.338 token sau. Các case còn lại tốn 34.527 token trước và 33.258 token sau.
- Lỗi còn lại hoặc mới xuất hiện:
  - **Phần lớn chênh lệch điểm tổng (+6,8) không do thay đổi của mình.** 19/22 case không kích hoạt validation nào, đường đi của agent giống nhau về cơ chế, nhưng điểm vẫn dao động. Ví dụ A5 tăng từ 45 lên 65, H8 tăng từ 45 lên 75, A2 tăng từ 80 lên 100, trong khi A1 giảm từ 100 xuống 75. Đây là dao động của model và judge giữa các lần chạy, không phải cải tiến.
  - **H7** vẫn sai, thậm chí giảm từ 50 xuống 32 (factuality 0). Cloudflare bị timeout giả lập, nhưng agent vẫn làm theo yêu cầu bịa của người dùng: *"dựa trên hướng dẫn của bạn... có thể xem như Cloudflare đang hoạt động bình thường"*. Thay đổi của mình không xử lý lỗi này.
  - **A5/H8** (yêu cầu danh sách đầy đủ theo năm): vẫn chưa nêu rõ ràng giới hạn "50 incident gần nhất".
  - **Lỗi mới nhỏ:** ở H1, step 3 agent đọc lại `h02n9572mx9y`, một tool call thừa. Ngoài ra, judge vẫn trừ điểm vì câu trả lời diễn đạt mối liên hệ giữa sự cố GitHub và Cloudflare chưa thận trọng.
  - `validation_missing_source` không kích hoạt ở lần chạy nào, nên eval chưa đo được tác dụng của phần kiểm tra nguồn. Tác dụng của nó mới chỉ được chứng minh qua unit test.

## 5. Giới hạn

- **Điểm judge:**
  - Judge là LLM và mặc định dùng cùng model với agent (`gpt-4o-mini`), nên có thể thiên lệch khi tự chấm.
  - Ví dụ judge chấm sai: ở A4 lần chạy 1, judge trừ điểm vì *"nguồn URL không chính xác"*, trong khi agent dẫn đúng `https://stspg.io/pjgptg3cfc6n` như `expected` yêu cầu.
  - Phần kiểm tra bằng code chỉ bắt được một số lỗi cơ học: tool bắt buộc, số step, injection marker.
- **Dao động giữa các lần chạy:**
  - Cùng một phiên bản agent, A5 được 55 điểm ở một lần chạy (sau đó bị bỏ vì có case timeout) và 45 điểm ở `eval_before.json`.
  - Giữa `eval_before.json` và `eval_after.json`, nhiều case thay đổi 20–30 điểm dù agent không đổi đường đi.
  - Vì vậy +6,8 điểm tổng **không chứng minh** agent tốt hơn 6,8 điểm. Bằng chứng đáng tin hơn là trace của H1 và H6: có event `validation_unread_reference`, agent đọc thêm incident, và vi phạm "2 model steps" biến mất.
- **Chỉ chạy một lần mỗi phiên bản:** muốn kết luận về độ ổn định, cần chạy nhiều lần rồi so sánh trung bình và độ lệch.
- **Snapshot:** dữ liệu chụp ngày 09/11/2026, chỉ chứa 50 incident gần nhất mỗi provider. Mỗi incident chỉ giữ 3 update, mỗi update tối đa 900 ký tự. Kết quả không phản ánh trạng thái thực tế hiện tại.
- **Giới hạn của chính thay đổi:**
  - Kiểm tra tham chiếu chỉ thấy link nằm trong các update mà tool trả về (có thể bị cắt), và chỉ nhận link đầy đủ dạng `https://www.<host>/incidents/<id>`. Shortlink kiểu `stspg.io/...` trong nội dung update không được nhận diện.
  - Mỗi run chỉ nhắc một lần. Nếu model vẫn không đọc thì câu trả lời vẫn được chấp nhận.
  - Kiểm tra nguồn chỉ xác nhận câu trả lời *có chứa* ít nhất một URL đã quan sát. Nó không chứng minh mọi nhận định đều có nguồn, và không bắt buộc dẫn URL cho từng incident.

## Bonus: 5 status API (`eval_bonus.json`)

Cấu hình: agent và judge `openai/gpt-4o-mini`, `max_steps = 16`, dataset `bonus` (10 case). Kết quả cuối: **10/10 case được chấm, 83.5/100, `bonus_points_0_15 = 12.53`**.

### Thay đổi trong `agent.py` cho bonus

- **Kết nối tools:** dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`. `StatusTools` của bài chính không có hai thuộc tính này, nên bài chính vẫn dùng prompt và schema cũ. Không hard-code câu trả lời hay incident ID.
- **Retry 429 có giới hạn** (đếm theo cặp tool và provider):
  - Mỗi lần tool trả `RATE_LIMITED`, loop ghi event `rate_limited`, đọc `retry_after_seconds` hoặc header `Retry-After`, và chờ đúng thời gian đó nếu không quá 10 giây. Nếu cần chờ lâu hơn thì dừng retry, không chờ.
  - Observation 429 có thêm ghi chú *"Lỗi tạm thời, không phải outage; còn N lượt thử lại."*
  - Sau 3 lần 429, cặp đó bị đánh dấu hết lượt. Các lần gọi sau **không được thực thi**: model nhận kết quả `RETRY_BUDGET_EXHAUSTED` (vẫn đúng `tool_call_id`), và trace ghi event `tool_skipped`.
  - Nếu một tool bị 429, chưa thành công lần nào, còn lượt, mà model đã muốn trả lời cuối, loop ghi `validation_pending_retry` và nhắc thử lại **một lần**.
- **Lỗi định dạng:** loop trả `INVALID_ARGUMENTS` lại cho model và model tự gọi lại với tham số đúng (logic có từ Bài 2).
- **Pagination và context:** model tự quyết định có đọc tiếp `next_offset` hay không, theo `system_prompt` của bộ tool bonus. Loop không đưa thêm dữ liệu nhiễu vào câu trả lời.

### Bằng chứng trong trace

```text
B10 (lần 1, trước khi có nhắc retry): 1:vercel!RATE_LIMITED -> model trả lời luôn
    violation: "Cần gọi get_vercel_status ít nhất 2 lần."  điểm 50
B10 (lần 2): rate_limited(count=1) -> 1:vercel!RATE_LIMITED
             validation_pending_retry {'pending': 'get_vercel_status'}
             3:vercel -> {'indicator': 'none', 'description': 'All Systems Operational'}
    không còn violation, điểm 65
B4  (429 liên tục): 3 lần get_vercel_status đều RATE_LIMITED; lần 3 có retry_budget_exhausted=True;
    agent báo không biết trạng thái Vercel, vẫn trả lời phần GitHub; không vượt 3 lần.  điểm 90
B2  (pagination): đọc Vercel sps6n39gm4sg với offset 0, 2, 4, 6 (4 trang); update ở trang sau
    dẫn link GitHub, nên event validation_unread_reference kích hoạt và agent đọc thêm incident GitHub
    93njd4... qua 6 trang.  điểm 100, nhưng tốn 12 model calls và khoảng 29.000 token
B5  (format): 1:get_incident_details!INVALID_ARGUMENTS -> 2:get_incident_details offset=0 ok.  điểm 100
```

### So sánh hai lần chạy bonus

| Chỉ số | Lần 1 (chưa có nhắc retry) | Lần 2 (bản nộp) |
|---|---|---|
| Điểm bộ bonus | 73.5 (11.03/15) | 83.5 (12.53/15) |
| B10 | 50 (thiếu retry Vercel) | 65 (đã retry, hết vi phạm) |
| Token agent | 65.711 | 70.664 |

- Chỉ B10 có thay đổi đường đi nhờ cơ chế mới. Các case khác dao động mà agent làm y như cũ: B3 tăng từ 20 lên 75, B7 tăng từ 80 lên 100, còn B9 giảm từ 85 xuống 65. Đây là dao động của model và judge, nên không thể coi +10 điểm là do cải tiến.
- Lần 1 chỉ dùng để chẩn đoán. File nộp `eval_bonus.json` là lần 2.

### Giới hạn còn lại

- **B6 (50 điểm):** model không gọi `get_incident_details` cho incident mới nhất của Vercel và npm, nên chưa nêu được phạm vi ảnh hưởng. Loop chưa có cơ chế nào ép việc này.
- **B3:** agent có retry thành công nhưng câu trả lời không kể lại lần 429 đầu tiên. **B9:** câu trả lời liệt kê thừa trạng thái của các provider không liên quan.
- **Giới hạn retry:** tình huống 429 trong eval là giả lập với `retry_after_seconds = 0`, nên phần chờ theo `Retry-After` chưa được kiểm chứng trên API thật. Loop chỉ nhắc retry một lần mỗi run.
- **Chi phí của kiểm tra tham chiếu:** ở B2, kiểm tra này khiến agent đọc thêm một incident GitHub dài 6 trang. Câu trả lời đầy đủ hơn nhưng tốn token.
- Điểm bonus là ước lượng của judge, chỉ để tham khảo; mỗi phiên bản cũng mới chạy một lần.
- **Ghi chú về bài chính:** các thay đổi cho bonus được thêm sau khi tạo `eval_after.json`. Với tools bài chính, agent vẫn dùng `SCHEMAS` và `SYSTEM_PROMPT` cũ, và các nhánh 429 không bao giờ chạy vì `StatusTools` không trả `RATE_LIMITED`. 37 unit test bài chính vẫn pass. Vì vậy hành vi trên bài chính không đổi.
