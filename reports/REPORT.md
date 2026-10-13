# Lab 25B — Report

**Họ tên / nhóm:** Phạm Quân / 2A202602890  **Commit:** a0887b7  **Điểm tự động (`make grade`):** 78 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | ❌ |
| G1 | yours | 1.0000 | 4.541 | 1.137 | 0.002 | 2.025 | 0.0 | ✅ |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | ❌ |
| G2 | yours | 1.0000 | 11.387 | 1.059 | 0.003 | 2.107 | 0.0 | ✅ |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | ❌ |
| G3 | yours | 1.0000 | 4.651 | 1.078 | 0.044 | 2.150 | 0.0 | ✅ |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | ❌ |
| G4 | yours | 1.0000 | 5.554 | 1.020 | 0.002 | 2.030 | 0.0 | ✅ |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | ❌ |
| G5 | yours | 0.8179 | 5.514 | 1.004 | 0.002 | 2.506 | — | ✅ |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | ❌ |
| G6 | yours | 1.0000 | 6.348 | 1.086 | 0.040 | 2.320 | 0.0 | ✅ |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | G4 naive p95 là 13.764 s; nếu timeout dài hơn 8 s thì request chậm vẫn bị giữ nhiều giây, nhưng không được quá lâu như 30 s. `attempt_timeout_s = 8` đủ để chặn hung request mà vẫn giữ deadline user (30 s cho cả lượt). |
| max_attempts_per_provider | 2 | G1/G2/G6 đều cần failover nhanh, nhưng không lặp lại vô hạn trên cùng provider. Naive client với 3 tầng retry lên tới 36 attempt / logical call khiến G2 amplification lên 9.904. Chỉ cho 1 retry trên mỗi provider giữ `amplification <= 1.6` như yêu cầu. |
| backoff_base_s / cap_s | 0.25 / 4.0 | G1 là spike rate-limit: `retry_after_violations` của naive lên 36027. Backoff full-jitter giữ 429 được xử lý theo retry-after, không coi như "retry ngay bừa". Cap 4 s tránh retry storm khi provider đã yếu. |
| retry_budget_ratio | 0.1 | G2/G3/G6 đều là các băng mà lỗi tạm thời và tài nguyên thật sự bị chặn. `retry_budget_ratio = 0.1` chặn retry swarm khi nơi lỗi không tự hồi phục; G3 resilient chỉ có retry_ratio = 0.044, còn naive 0.164. |
| breaker.window_size / min_calls | 20 / 10 | `min_calls=10` ngăn breaker bật quá sớm trên nền thấp; `window_size=20` đủ phản ứng nhanh với ngưỡng lỗi của G2/G3/G4 mà không quá thô. |
| breaker.failure_rate_threshold | 0.5 | G2 và G3 đều có lỗi theo mô hình burst hoặc brownout. Dựa trên G2 naive 529 + rate-limit rất cao; ngưỡng 50% chặn sự cố trước khi hỏi lại provider. |
| breaker.slow_call_s | 5.0 | G4 slow region không có lỗi nhưng latency tăng thêm 5 s. Vấn đề không phải lỗi, mà là call chậm. `slow_call_s = 5.0` phát hiện “provider làm chậm” trước khi p95 vượt 10 s. |
| breaker.open_duration_s | 10.0 | G2/G6 đều cần dừng gửi request trong thời gian ngắn để đỡ tải. Khi provider hồi phục, 10 s cool-down cho phép drain queue mà không để lại retry loop kéo dài. |
| breaker.half_open_max_probes | 2 | Giữ kênh half-open nhỏ để không cho 2 probe “phá nát” lại hệ thống; G2/G6 đều dễ bị metastable. Hai probe đủ cho xác nhận provider đã hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```text
400 invalid_request_error / malformed
  -> breaker: no
  -> retry: no
  -> failover: no (đừng lặp lại lỗi client)
  -> cooldown: no

429 rate_limit_exceeded
  -> breaker: yes if sustained or queueing pressure
  -> retry: yes, nhưng chỉ sau retry_after + jitter
  -> failover: yes nếu quota/429 kéo dài, không lặp ngay trên provider đang rate-limit
  -> cooldown: yes, tạm thời khóa provider

529 / 503 / timeout / api_error
  -> breaker: yes (bao gồm slow-call / failure-rate)
  -> retry: yes trên cùng provider nếu còn retry budget, nhưng bị giới hạn và full jitter
  -> failover: yes sau retry cuối cùng / khi breaker mở
  -> cooldown: yes; lưu rate-limit / quota / slow-call để tránh thừa tải

insufficient_quota
  -> breaker: no (không phải hỏng infra)
  -> retry: no
  -> failover: yes nếu provider khác có quota, nhưng không lặp lại trên cùng provider
  -> cooldown: maybe short; không retry tiếp tục trên account hết quota
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Trong G2 naive, `amplification = 9.904` và `retry_ratio = 8.904` trong khi resilient chỉ còn `1.059` và `0.003`. G3 naive vẫn có `retry_ratio = 0.164` dù không sập, nhưng resilient giảm xuống `0.044`. Nghĩa là breaker và failover là lớp chính, retry budget là lớp chặn “khi mọi thứ đã lệch” để không cho retry swarm trở lại. Bỏ budget là vô ích nhất khi provider đã hồi phục nhưng queue vẫn đầy: `recovery_s` của naive G2 là 155 s, bằng chứng cho thấy cả hệ thống đã quá tải lâu hơn thời gian phục hồi thực của GPU. 
2. **`attempt_timeout_s: 30`.** Chúng ta không giữ timeout 30 s vì G4 naive p95 13.764 s nhưng availability vẫn `1.0`; tức là phần lớn request đều hoàn thành trước 30 s, và chậm không phải do “request treo vô hạn”. Hệ thống càng lâu timeout thì càng dễ giữ request sống trong queue lâu hơn, khiến cải thiện latency không đáng kể nhưng làm tăng load. Đúng nhất là timeout nên dựa vào p99 quãng bình thường + deadline user + khoảng cần cho failover, không phải “để request nặng nhất kịp chờ 30 s”; với G4, `give_up_reasons = {}` và p95=13.764 s cho thấy 30 s là quá dài và không cần thiết. 
3. **Bỏ lọc tool calling khỏi `ladder()`.** Trong G6 resilient, `incompatible_routes = 0`, còn G6 naive bị dịch vụ nền không có tool-calling model tỏ ra vụng về. Trong thực tế, nếu `gamma-mini` (không hỗ trợ tool) được sử dụng cho tool-turn, product manager thấy đây là “một route sai khiến 1 request nào đó phải thất bại dù hệ thống còn khác route tốt hơn”. Vì vậy, `ladder()` phải bảo toàn khả năng tool-calling: `alpha`, `alpha-eu`, `beta` giữ route tool, `gamma-mini` chỉ dùng cho non-tool hoặc fallback cuối cùng. G6 resilient đạt `availability = 1.0`, `incompatible_routes = 0`, họ chứng minh đường dẫn tool luôn phù hợp với loại call. 

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 cents | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trả lời: chỉ idempotency key là chưa đủ vì nếu worker crash sau khi thanh toán nhưng trước khi ghi checkpoint, hệ thống có thể tạo refund trùng hoặc gửi email trùng khi retry. Ngược lại, chỉ checkpoint cũng chưa đủ vì nếu checkpoint ghi trước khi payment thực sự thành công thì một crash ở bước giữa có thể làm hệ thống nghĩ “đã refund” trong khi thật ra payment chưa xảy ra, gây mất tiền hoặc overpayment. Với `checkpoint` được ghi sau khi step hoàn tất và idempotency key ổn định theo `workflow_id + step_id`, hệ thống có thể resume an toàn: lần retry không tạo side effect lặp lại; worker crash và resume không gây duplicate refund/email. 

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 1 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page | none | — |
| slow_leak | ticket | none | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page | none | — |
| small_bump | none | page | none | — |

Error budget 30 ngày: đã tiêu 42.95%, còn 57.05%. `freeze_releases = false`, vì nên không đóng băng release: budget còn 57% và không có vết nứt lớn theo mô hình 30 ngày. `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa vì mức lỗi này không vượt qua ngưỡng burn-rate 1h/6h/3d; nó chỉ là “sự biến động nhỏ” chứ không phải lỗi của hệ thống. Burn-rate alerter dùng 1h/6h/3d + error budget chuyên sâu; điều này tránh false page khi `deploy_blip`, `night_spike`, và `small_bump` không nguy hiểm. 

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):** Tại t=120, α mất 75% GPU và retry storm bắt đầu lên tới 24–250 attempt/s trong 3 phút tiếp theo. `availability` rơi xuống 45.6%, `amplification` lên 9.904, và `retry_ratio` lên 8.904—một hệ thống vẫn hoạt động nhưng đang tự phá hủy chính nó bằng retry. Khi provider hồi phục, requests vẫn không phục hồi vì hàng đợi đầy và client đã đưa lượng retry quá lớn vào cùng 1 deployment; kết quả là hệ thống bước vào trạng thái metastable failure chưa tự ổn định được. 

**Ảnh hưởng:** 1,285 lượt user thất bại (`good_turns = 1077/2362`), hệ thống kéo dài 155 s trước khi recovery_threshold đạt 95% thành công, và theo giả định tháng chỉ có sự cố này thì tiêu tốn khoảng 42.95% error budget (dữ liệu SLO 30 ngày). Đây là “sụp trong 3 phút, và không tự hồi phục” chứ không chỉ là một outage ngắn. 

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% năng lực | `calls_ok=7`, `calls_done=7`, `attempts_alpha=5` và sớm bắt đầu tăng mạnh |
| 121 | retry bắt đầu bùng nổ | `attempts_alpha=7`, trong khi `calls_ok` chỉ còn 2 |
| 134 | queue đang tắc và provider bị nén | `attempts_alpha=24`, `calls_ok=2` |
| 137–140 | punc đầu tiên của storm | `attempts_alpha=82` ở t=137, 122 ở t=138, 216 ở t=140; `calls_ok` = 0 |
| 145 | provider quá tải, cực điểm | `attempts_alpha=250`, `calls_ok=0` |
| 300 | provider đã hồi phục nhưng hệ thống vẫn chưa hồi phục | `recovery_s = 155.0`, `availability = 0.456` và `retries_exhausted = 655` cho thấy retry loop vẫn duy trì tải quá sâu |

**Trigger vs root cause:** trigger là mất 75% GPU trên `alpha` (các 529/overloaded error tức thời). Root cause là retry 3 tầng không có jitter, không có retry-after, không có budget, không có breaker; lượng retry được nhân lên theo từng tầng thành một vòng lặp “tự làm tổn hại chính mình”. Vì vậy provider đã hồi phục nhưng hệ thống vẫn sập — đó là metastable failure: hàng đợi đã đầy, client tiếp tục retry vào cùng provider, khiến load luôn cao hơn khả năng phục hồi thực của cluster. 

**Điều gì giúp / điều gì làm tệ hơn:** Điều giúp là `alpha` thực sự có thể phục hồi nếu không có retry storm; việc có 18 server + bộ giữ hàng đợi không làm gì khi client tăng tải lên 250 attempt/s. Điều làm tệ hơn là cả request bị timeout và rate limit đều nằm trong cùng đường retry, làm cho queue đầy rồi kéo dài thêm, không phải do 1 provider chết mãi. Hệ thống không có circuit breaker, nên mỗi lần provider ngã xuống, client lại “đẩy mạnh hơn” để lấy lại thành công. 

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thêm breaker sliding-window với `min_calls = 10`, `failure_rate_threshold = 0.5`, `slow_call_s = 5.0` và `open_duration_s = 10` vào mọi client | prevent | Client infra | CI đánh G2/G6: `amplification <= 1.6`, `availability >= 95%`, `recovery_s <= 30` |
| 2 | Bắt buộc `retry_after` + full jitter + retry budget; không retry 429 ngay ngay trên cùng provider | prevent | Reliability/SRE | Trong G1: `retry_after_violations = 0`, `amplification <= 1.2` |
| 3 | Dẫn cảnh báo và “auto-shed” khi `provider_error_rate >= 10%` trong 5 phút và `slow_call_rate >= 60%` | detect | SRE / on-call | Alerting tests công khai: `false_pages = 0`, `slow_leak` phải phát hiện ticket đúng lúc |
| 4 | Thêm queue-filter để tắt retry khi hàng đợi đầy và provider đang phục hồi | mitigate | Platform | So sánh `recovery_s` trước/sau, đo `in_queue / retry_load` giảm và không còn metastable loop |

## 7. Một điểm yếu còn lại

Một điểm yếu vẫn còn: trong G2 p95, resilient client vẫn ở 11.387 s — khá gần với ngưỡng 10 s, và khi cả 4 provider cùng chậm và 1–2 provider đang gánh tải lớn, giảm hiệu quả failover. Cụ thể, G4 / G6 cho thấy nếu tất cả provider cùng chậm hoặc bị rate-limit, p95 có thể vượt mục tiêu sau khi hệ thống chính đã ổn định. Cách sửa: thêm “provider score” theo độ sẵn sàng và latency 95th, không chỉ `open/closed` breaker, đồng thời ưu tiên route sang provider có fallback an toàn cho tool-calling, và limit concurrency trên mỗi provider khi `queue_delay > 2s` hoặc `slow_call_rate > 0.6`. Việc này giảm độ lệch giữa “đã bỏ retry storm” và “vẫn đang chậm do tải tập trung” khi 4 provider cùng chậm. 

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | ❌ |
| cached (của bạn) | 0.8469 | 0.8587 | 0.742 | 0 | 181 | 80 | 0.921 | ❌ |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.80 | Đặt 0.80 là ngưỡng cân bằng giữa hit ratio với độ chính xác; với 0.80, `wrong_answers = 181`, phần lớn là `other_question` trong `wrong_answers_by_reason`. Đây là bằng chứng cho thấy ngưỡng quá lỏng; nếu quá ngắn thì cache trả câu gần đúng nhưng không đúng. |
| ttl_s | 600 | TTL=600 s giúp cache còn fresh đủ cho phần lớn truy vấn FAQ mà không lưu stale quá lâu. Tuy không hoàn hảo, `faq_availability = 0.8587` cao hơn không cache (0.8424), nhưng nếu tăng TTL hơn 600 s sẽ gia tăng `outdated_answers` vì không có webhook nào dọn cache nhanh hơn TTL. |
| max_stale_s | 0 | Chatbot chỉ dùng stale khi hệ thống đang mất provider; với `max_stale_s = 0`, cache không phép phục vụ stale như một hit bình thường, nên tránh câu trả lời lỗi lâu. Mức này làm `stale_served = 0` nhưng ảnh hưởng trực tiếp tới `faq_availability` khi provider toàn sập: nếu `max_stale_s` > 0 thì có thể tăng availability nhưng lại tăng nguy cơ trả nhầm. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Với `similarity_threshold = 0.80`, cache đạt `cache_hit_ratio = 0.742` nhưng `wrong_answers = 181` và `wrong_answers_by_reason = { "other_question": 181 }`; đây là “gần nhưng khác” chứ không phải câu trả lời sai do không đúng mục đích. Nếu ngưỡng chặt hơn, `wrong_answers` sẽ giảm nhưng hit ratio giảm; 0.80 là điểm chốt vì `faq_availability` tăng từ 0.8424 (không cache) lên 0.8587, nhưng vẫn mất khoảng 15% vì câu hỏi cá nhân / non-FAQ / sập provider. 
2. **TTL.** TTL 600 s là mức cân bằng cho C1: `faq_availability = 0.8587` > 0.8424, nhưng `outdated_answers = 80` cho thấy cache vẫn có stale answer sau khi nội dung đã thay đổi. Webhook `on_content_update` giúp invalidate, nhưng webhook **không** giải quyết `outdated_answers` nếu cache không được invalidate đúng lúc; TTL là giới hạn cuối cùng để cấm stale sống quá lâu trong cache. 
3. **Stale-if-error.** Với `max_stale_s: 0`, `stale_served = 0`. Điều này làm `faq_availability` giảm xuống 0.8587, thấp hơn nếu cho stale để phục vụ khi mọi provider hết hàng. Tức là cache không thể giữ dashboard xanh khi provider chết, nhưng đồng thời không bớt nguy cơ trả về answer đã cũ. Dịch nghĩa là: stale-if-error giúp survivability, nhưng chỉ với `max_stale_s` hợp lý; `max_stale_s = 0` là an toàn nhưng thu hẹp khả năng phục vụ. 

Trả lời:

- Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") **không** được cache, kể cả khi cache theo từng user? Bởi vì nó là `other_question` và có dữ liệu riêng tư; nếu cache theo user hoặc theo ticket và không scope thích hợp, cache sẽ lộ dữ liệu giữa các user. Tại C1, wrong_answers_by_reason cho thấy chất lượng cache không chính xác khi lưu câu hỏi gần tương tự nhưng không cùng ngữ cảnh; do đó câu hỏi cá nhân phải bypass cache. 
- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố? Bạn alert dựa trên `provider_error_rate` hoặc `cache_staleness` / `outdated_answers`, không dựa vào dashboard xanh. Đối với cache, metric quan trọng là `provider_unavailable`, `cache_hit_ratio`, `stale_served`, `outdated_answers` và `cache_age`. Khi `provider` liên tục `service_unavailable`, dù dashboard vẫn xanh vì cache, on-call vẫn phải nhận alert dựa trên `provider outage` hoặc tăng `outdated_answers` / `stale_served` để biết hệ thống đang chạy trên dữ liệu cũ. 

