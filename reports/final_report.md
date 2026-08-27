# Day 25 — Reliability Engineering for Production Agents

**Sinh viên:** Nguyễn Trọng Đức  
**MSSV:** 2A202601636  
**Ngày đo:** 27/08/2026

## 1. Architecture summary

```text
User request
     |
     v
[ReliabilityGateway]
     |
     +--> [Semantic cache: memory/Redis]
     |         | HIT: return route=cache_hit, cost=0
     |         v MISS
     +--> [Circuit breaker: primary] --> FakeLLM primary
     |         | OPEN/error: fail fast
     +--> [Circuit breaker: backup]  --> FakeLLM backup
     |         | OPEN/error
     v
[Static fallback: degraded response]

Every response records route, provider, latency, estimated cost and error.
Every breaker records timestamped CLOSED/OPEN/HALF_OPEN transitions.
```

Pipeline ưu tiên cache để tránh chi phí, sau đó lần lượt gọi primary và backup qua circuit breaker riêng. Khi tất cả provider lỗi hoặc circuit đang OPEN, gateway trả static fallback thay vì làm sập request.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure threshold | 3 | Ba lỗi liên tiếp đủ phát hiện provider bất ổn nhưng tránh mở circuit chỉ vì một lỗi ngẫu nhiên. |
| reset timeout | 2 s | Ngắn để lab đo được recovery, nhưng vẫn tạo khoảng fail-fast chống retry storm. Recovery thực đo 2.22–2.53 s xác nhận cấu hình. |
| success threshold | 1 | Một probe thành công đóng circuit; phù hợp provider giả lập và giảm thời gian degraded. Production nên cân nhắc 2–3 probe. |
| cache TTL | 300 s | Năm phút cân bằng tái sử dụng và độ cũ; câu hỏi dated còn được guardrail kiểm tra số/năm. |
| similarity threshold | 0.92 | Ngưỡng chặt vì false hit nguy hiểm hơn cache miss. Khác năm hoặc ID bốn chữ số luôn bị từ chối dù similarity cao. |
| load requests | 100/scenario | Ba scenario tạo đúng 300 request, đủ quan sát circuit, fallback và cache mà vẫn tái chạy nhanh. |
| providers | primary 25%, backup 5% default fail rate | Backup ít lỗi hơn để giữ availability khi primary suy giảm. |

## 3. SLO definitions

| SLI | SLO target | Actual (main run) | Met? |
|---|---:|---:|---|
| Availability | >= 99% | 99.00% | Yes |
| Latency P95 | < 2500 ms | 313.48 ms | Yes |
| Fallback success rate | >= 95% | 96.15% | Yes |
| Cache hit rate | >= 10% | 65.00% | Yes |
| Recovery time | < 5000 ms | 2277.30 ms | Yes |

Run cuối đạt cả năm SLO. Static fallback vẫn được tính là failed request, vì vậy availability không bị làm đẹp bởi degraded response.

## 4. Metrics

Số dưới đây lấy trực tiếp từ `reports/metrics.json`, sinh bởi 3 scenario × 100 request.

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 0.9900 |
| error_rate | 0.0100 |
| latency_p50_ms | 276.63 |
| latency_p95_ms | 313.48 |
| latency_p99_ms | 319.58 |
| fallback_success_rate | 0.9615 |
| cache_hit_rate | 0.6500 |
| estimated_cost | 0.041988 |
| estimated_cost_saved | 0.195000 |
| circuit_open_count | 9 |
| recovery_time_ms | 2277.30 |

P50 không giảm mạnh khi có cache vì starter chỉ đưa latency lớn hơn 0 vào mẫu; cache hit latency 0 bị loại khỏi percentile. Lợi ích cache thể hiện rõ hơn ở availability, số lần circuit mở và estimated cost.

## 5. Cache comparison

Hai run dùng cùng cấu hình provider, breaker, ba scenario và 300 request; chỉ đổi `cache.enabled`.

| Metric | Without cache | With memory cache | Delta |
|---|---:|---:|---:|
| availability | 0.9600 | 0.9900 | +0.0300 |
| latency_p50_ms | 277.40 | 276.63 | -0.77 ms |
| latency_p95_ms | 314.51 | 313.48 | -1.03 ms |
| estimated_cost | 0.124144 | 0.041988 | -0.082156 (-66.2%) |
| cache_hit_rate | 0.0000 | 0.6500 | +0.6500 |
| circuit_open_count | 24 | 9 | -15 |

Cache giảm mạnh số lần gọi provider, chi phí và áp lực lên breaker. Do provider dùng random chưa cố định seed, chênh lệch availability có một phần nhiễu; kết luận mạnh nhất là cost và cache hit.

## 6. Redis shared cache

In-memory cache bị cô lập theo process: nhiều gateway pod sẽ gọi provider lặp lại và không thấy entry của nhau. `SharedRedisCache` lưu hash gồm cả `query` và `response`, đặt TTL bằng `EXPIRE`, cho phép exact lookup và semantic scan dùng chung giữa các instance. Guardrail privacy và false-hit vẫn chạy ở cả `get` và `set`.

### Evidence of shared state

Hai object cache dùng hai kết nối Redis riêng và cùng prefix:

```text
instance A: set("shared state proof", "visible from second instance")
instance B: get("shared state proof")
result: ('visible from second instance', 1.0)
```

### Redis CLI evidence

```text
redis-cli DBSIZE
13

redis-cli --scan --pattern "rl:cache:*"
rl:cache:fff10da1c72c
rl:cache:3dab98c0e49e
rl:cache:734852f3cf4a
... (13 keys total)
```

### Memory vs Redis run

| Metric | Memory | Redis | Notes |
|---|---:|---:|---|
| availability | 0.9900 | 0.9867 | Cả hai đều giữ hệ thống gần 99%. |
| latency_p50_ms | 276.63 | 272.53 | Cache-hit 0 ms không nằm trong mẫu latency. |
| latency_p95_ms | 313.48 | 312.82 | Redis overhead nhỏ so với provider latency. |
| cache_hit_rate | 0.6500 | 0.6967 | Redis dùng chung state giữa ba scenario trong cùng run. |
| estimated_cost | 0.041988 | 0.036316 | Redis run tiết kiệm thêm nhờ nhiều hit hơn. |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Result |
|---|---|---|---|
| primary_timeout_100 | Primary lỗi 100%, circuit mở và traffic chuyển backup | Backup phục vụ phần lớn request; có static fallback khi backup lỗi; circuit open được ghi log | Pass |
| primary_flaky_50 | Primary lỗi xen kẽ, có cả primary/fallback và recovery | Circuit dao động OPEN → HALF_OPEN → CLOSED; recovery trung bình gần reset timeout 2 s | Pass |
| all_healthy | Override rỗng dùng fail rate mặc định, phần lớn đi primary | Primary phục vụ phần lớn miss; cache xử lý truy vấn lặp và fallback xử lý lỗi ngẫu nhiên | Pass |

`recovery_time_ms` xấp xỉ 2000–2500 ms vì circuit phải chờ `reset_timeout_seconds=2`, rồi chỉ chuyển CLOSED khi một request sau timeout được phép làm probe và thành công. Độ trễ giữa 2 s và số đo là thời gian chờ request kế tiếp cộng latency provider.

## 8. Failure analysis

Điểm yếu còn lại quan trọng nhất là state circuit breaker vẫn nằm trong RAM. Trong triển khai ba pod, mỗi pod có failure counter và OPEN/CLOSED state riêng; một pod có thể tiếp tục gọi provider đang lỗi trong khi pod khác đã mở circuit. Cách sửa là lưu trạng thái và thời điểm mở trong Redis bằng thao tác atomic (Lua script hoặc transaction), kèm lease cho HALF_OPEN để chỉ một pod được gửi probe. Redis cũng là single dependency của cache; nếu Redis sập, gateway hiện có thể ném connection error trước khi tới provider. Production cần timeout ngắn và graceful degradation: bắt Redis error, chuyển sang in-memory cache hoặc coi là cache miss.

## 9. Next steps

1. Thêm seed cấu hình và truyền `random.Random(seed)` vào provider/query selection để hai chaos run tái lập bit-for-bit.
2. Chuyển circuit state sang Redis với atomic transition và distributed HALF_OPEN probe lease.
3. Thêm Redis timeout/circuit breaker riêng, fallback về in-memory, và chạy load đồng thời bằng `ThreadPoolExecutor` để đo contention thực tế.

## Verification evidence

```text
35 passed, 7 xpassed in 3.98s
```

Redis đang chạy nên không có test skipped. Bảy test yêu cầu hoàn thiện đã chuyển từ XFAIL thành XPASS.
