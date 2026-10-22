+++
date = '2026-10-22T02:30:00+07:00'
draft = false
slug = 'scalable-p01-5-api-deep-dive'
title = 'Scalable System Design P01.5: Thiết Kế Hệ Thống 1M RPS - Public API'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "scalability", "api", "golang", "redis"]
description = 'Ở 1M RPS, gọi Redis mỗi request là bất khả thi — cần 19,8 instance chỉ cho rate limit. Đo thật two-tier limiter nhanh hơn 57×, noisy neighbor, và chi phí auth.'
+++

> *Bài viết thuộc series **Scalable System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, kiến trúc, code và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên máy local (Go 1.24, Redis 7.4.2).*

# Mở đầu

[P01.3](/posts/scalable-p01-3-rps-deep-dive/) đào sâu hệ thống RPS-heavy dạng **burst** (flash sale 500.000 RPS trong 10 giây). Bài này là đầu còn lại: **1.000.000 RPS bền, liên tục, đa tenant** — một public API.

Khác biệt làm đổi hẳn kiến trúc:

| | Flash sale (P01.3) | **Public API (bài này)** |
|---|---|---|
| Hình dạng tải | Burst 50× trong 10 giây | **Bền, liên tục 24/7** |
| Tenant | 1 (shop của bạn) | **Hàng nghìn đến hàng triệu** |
| Hot resource | **Tồn kho** (1 dòng) | **Không có** — nút thắt là chính limiter |
| Lỗi tệ nhất | Oversell | **Noisy neighbor** — 1 tenant giết trải nghiệm của tenant khác |
| Giới hạn | Một con số toàn cục | **Theo tenant, theo tier, theo endpoint** |
| Thứ phải bảo toàn | Không bán âm tồn kho | **Công bằng giữa các tenant** |

Và một con số quyết định mọi thứ: **ở 1M RPS, bạn không thể gọi Redis mỗi request.** Phần lớn bài này là hệ quả của câu đó.

## Workload model

| Chỉ số | Giá trị |
|---|---:|
| RPS bền | **1.000.000** |
| Số tenant | 10.000 |
| Tenant lớn nhất | 30% tổng traffic |
| Tier | free (100 rps) · pro (5.000 rps) · enterprise (100.000 rps) |
| Mục tiêu p99 | < 50 ms |
| Mỗi request cần | verify API key + check quota + đếm usage |

Để ý dòng cuối: **mỗi request phải làm 3 việc trước khi vào nghiệp vụ.** Ở 1M RPS, đó là 3 triệu phép tra cứu mỗi giây — và chúng phải gần như miễn phí.

---

# Bức tường Redis

Đây là phát hiện quan trọng nhất của bài. Tôi đo token bucket phân tán trên Redis, 200.000 request, 120 worker:

| | check/s | p99 | Số lời gọi Redis |
|---|---:|---:|---:|
| **Redis mỗi request** | **50.574** | 5,12 ms | 200.000 |

**50.574 check/s.** Để đạt 1.000.000 RPS, bạn cần:

```
1.000.000 ÷ 50.574 ≈ 19,8 instance Redis
```

**Gần 20 instance Redis chỉ để làm rate limit** — chưa tính cache, session, hay bất cứ việc gì khác. Và p99 **5,12 ms** đã chiếm 10% ngân sách 50 ms, chỉ cho một phép kiểm tra.

Đây không phải vấn đề tối ưu Redis. Đây là ràng buộc kiến trúc: **một round-trip mạng cho mỗi request không scale tới hàng triệu RPS.**

## Lời giải: two-tier limiter

Thay vì xin **từng token** từ Redis, mỗi instance xin **một lô** rồi tự phát cục bộ:

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="tt-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="tt-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="tt-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Hai tầng: xin một LÔ token, rồi tự phát cục bộ</text>
<rect x="24" y="42" width="420" height="192" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#tt-sh)"/>
<text x="234" y="66" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">❌ Redis MỖI request</text>
<rect x="46" y="80" width="78" height="112" rx="7" fill="#FFFFFF" stroke="#DC2626" stroke-width="1.8"/>
<text x="85" y="100" text-anchor="middle" font-size="10" font-weight="700" fill="#9F1239">API node</text>
<rect x="56" y="110" width="58" height="16" rx="3" fill="#FFE9EC"/>
<rect x="56" y="132" width="58" height="16" rx="3" fill="#FFE9EC"/>
<rect x="56" y="154" width="58" height="16" rx="3" fill="#FFE9EC"/>
<text x="85" y="122" text-anchor="middle" font-size="8" fill="#BE123C">req 1</text>
<text x="85" y="144" text-anchor="middle" font-size="8" fill="#BE123C">req 2</text>
<text x="85" y="166" text-anchor="middle" font-size="8" fill="#BE123C">req N</text>
<text x="85" y="186" text-anchor="middle" font-size="8" fill="#A8A29E">...</text>
<rect x="300" y="100" width="124" height="72" rx="8" fill="#DC2626"/>
<text x="362" y="128" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">REDIS</text>
<text x="362" y="148" text-anchor="middle" font-size="10" fill="#FFD9DE">1 round-trip</text>
<text x="362" y="163" text-anchor="middle" font-size="10" fill="#FFD9DE">MỖI request</text>
<line x1="128" y1="118" x2="296" y2="118" stroke="#DC2626" stroke-width="1.6" marker-end="url(#tt-a)"/>
<line x1="128" y1="140" x2="296" y2="136" stroke="#DC2626" stroke-width="1.6" marker-end="url(#tt-a)"/>
<line x1="128" y1="162" x2="296" y2="154" stroke="#DC2626" stroke-width="1.6" marker-end="url(#tt-a)"/>
<rect x="46" y="200" width="378" height="26" rx="6" fill="#FFD9DE"/>
<text x="235" y="218" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">50.574 check/s · p99 5,12 ms · cần 19,8 Redis cho 1M RPS</text>
<rect x="456" y="42" width="420" height="192" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#tt-sh)"/>
<text x="666" y="66" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">✅ Two-tier — xin một LÔ</text>
<rect x="478" y="80" width="120" height="112" rx="7" fill="#FFFFFF" stroke="#0D9488" stroke-width="1.8"/>
<text x="538" y="98" text-anchor="middle" font-size="10" font-weight="700" fill="#134E4A">API node</text>
<rect x="488" y="106" width="100" height="34" rx="5" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.4"/>
<text x="538" y="120" text-anchor="middle" font-size="9" font-weight="800" fill="#134E4A">bucket CỤC BỘ</text>
<text x="538" y="133" text-anchor="middle" font-size="9" fill="#0F766E">10.000 token</text>
<rect x="488" y="148" width="30" height="14" rx="3" fill="#E6FBF9"/>
<rect x="523" y="148" width="30" height="14" rx="3" fill="#E6FBF9"/>
<rect x="558" y="148" width="30" height="14" rx="3" fill="#E6FBF9"/>
<text x="538" y="178" text-anchor="middle" font-size="9" fill="#0F766E">phát trong RAM, 0 mạng</text>
<rect x="736" y="100" width="124" height="72" rx="8" fill="#0D9488"/>
<text x="798" y="124" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">REDIS</text>
<text x="798" y="144" text-anchor="middle" font-size="10" fill="#CCFBF1">1 round-trip</text>
<text x="798" y="159" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">cho CẢ LÔ</text>
<line x1="602" y1="130" x2="732" y2="130" stroke="#0D9488" stroke-width="2.5" marker-end="url(#tt-t)"/>
<text x="667" y="122" text-anchor="middle" font-size="9" font-weight="700" fill="#0F766E">xin 10.000</text>
<rect x="478" y="200" width="382" height="26" rx="6" fill="#CCFBF1"/>
<text x="669" y="218" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">2.893.314 check/s · p99 338 µs · chỉ 20 lời gọi Redis</text>
<text x="450" y="266" text-anchor="middle" font-size="14" font-weight="800" fill="#374151">Đánh đổi: kích thước lô quyết định độ CHÍNH XÁC</text>
<rect x="30" y="282" width="840" height="124" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="48" y="304" font-size="11" font-weight="800" fill="#374151">Cấu hình</text>
<text x="290" y="304" font-size="11" font-weight="800" fill="#374151">check/s</text>
<text x="400" y="304" font-size="11" font-weight="800" fill="#374151">p99</text>
<text x="490" y="304" font-size="11" font-weight="800" fill="#374151">lời gọi Redis</text>
<text x="640" y="304" font-size="11" font-weight="800" fill="#374151">cho qua</text>
<line x1="48" y1="310" x2="852" y2="310" stroke="#E7E5E4" stroke-width="1.5"/>
<text x="48" y="328" font-size="10" font-family="monospace" fill="#9F1239">lô 100 · deny ngay</text>
<text x="290" y="328" font-size="10" font-family="monospace" fill="#57534E">3.551.081</text>
<text x="400" y="328" font-size="10" font-family="monospace" fill="#57534E">127 µs</text>
<text x="490" y="328" font-size="10" font-family="monospace" fill="#57534E">316</text>
<text x="640" y="328" font-size="10" font-family="monospace" font-weight="700" fill="#DC2626">15,8% ❌ deny oan</text>
<text x="48" y="348" font-size="10" font-family="monospace" fill="#9F1239">lô 1.000 · deny ngay</text>
<text x="290" y="348" font-size="10" font-family="monospace" fill="#57534E">3.573.661</text>
<text x="400" y="348" font-size="10" font-family="monospace" fill="#57534E">4 µs</text>
<text x="490" y="348" font-size="10" font-family="monospace" fill="#57534E">115</text>
<text x="640" y="348" font-size="10" font-family="monospace" font-weight="700" fill="#DC2626">57,0% ❌</text>
<text x="48" y="368" font-size="10" font-family="monospace" fill="#7C4A03">lô 10.000 · deny ngay</text>
<text x="290" y="368" font-size="10" font-family="monospace" fill="#57534E">3.257.018</text>
<text x="400" y="368" font-size="10" font-family="monospace" fill="#57534E">159 µs</text>
<text x="490" y="368" font-size="10" font-family="monospace" fill="#57534E">19</text>
<text x="640" y="368" font-size="10" font-family="monospace" fill="#7C4A03">90,0%</text>
<text x="48" y="390" font-size="10" font-family="monospace" font-weight="700" fill="#134E4A">lô 10.000 · CHỜ sync</text>
<text x="290" y="390" font-size="10" font-family="monospace" font-weight="700" fill="#134E4A">2.893.314</text>
<text x="400" y="390" font-size="10" font-family="monospace" font-weight="700" fill="#134E4A">338 µs</text>
<text x="490" y="390" font-size="10" font-family="monospace" font-weight="700" fill="#134E4A">20</text>
<text x="640" y="390" font-size="10" font-family="monospace" font-weight="800" fill="#0F766E">100% ✅ ĐÚNG</text>
<rect x="30" y="418" width="840" height="54" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="48" y="440" font-size="11" fill="#92400E">Lô nhỏ + deny ngay = nhanh nhưng <tspan font-weight="700">từ chối oan tới 84%</tspan> traffic hợp lệ (bucket cục bộ cạn giữa 2 lần sync).</text>
<text x="48" y="460" font-size="11" font-weight="800" fill="#7C4A03">Cấu hình đúng: lô LỚN + CHỜ lô đang xin → vẫn nhanh hơn Redis-mỗi-request 57×, và cho qua 100%.</text>
</svg>
</div>

**Cấu hình đúng (lô 10.000 + chờ sync) so với Redis mỗi request:**

| | Redis mỗi request | **Two-tier** | Cải thiện |
|---|---:|---:|---:|
| check/s | 50.574 | **2.893.314** | **57×** |
| p99 | 5,12 ms | **338 µs** | **15×** |
| Lời gọi Redis | 200.000 | **20** | **10.000×** |
| Cho qua đúng | 100% | **100%** | — |

Và quan trọng nhất: **2.893.314 check/s nghĩa là một instance đủ cho 1M RPS**, thay vì cần 19,8 instance Redis.

## Cái bẫy của two-tier: deny oan

Để ý cột "cho qua" trong sơ đồ. Với lô nhỏ và chính sách "hết token cục bộ thì từ chối ngay", hệ thống **từ chối tới 84% traffic hợp lệ** (lô 100 → cho qua 15,8%).

Nguyên nhân: bucket cục bộ cạn giữa hai lần sync, và trong lúc một goroutine đang đi xin lô mới thì **mọi goroutine khác bị từ chối** thay vì chờ. Đây là lỗi thiết kế, không phải đặc tính của two-tier.

```go
// internal/infrastructure/ratelimit/twotier.go

// Lua: lấy TỚI N token trong MỘT lời gọi. Đây là mấu chốt —
// gọi N lần để lấy N token thì mất hết ý nghĩa của two-tier.
var takeBatch = redis.NewScript(`
  local t = redis.call('TIME')
  local now = tonumber(t[1])*1000 + math.floor(tonumber(t[2])/1000)
  local d = redis.call('HMGET', KEYS[1], 'tk', 'ts')
  local tk, ts = tonumber(d[1]), tonumber(d[2])
  local rate, burst, want = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])
  if tk == nil then tk = burst; ts = now end
  tk = math.min(burst, tk + math.max(0, now - ts)/1000.0 * rate)
  local give = math.floor(math.min(tk, want))
  tk = tk - give
  redis.call('HSET', KEYS[1], 'tk', tk, 'ts', now)
  redis.call('PEXPIRE', KEYS[1], 120000)
  return give
`)

type bucket struct {
    mu      sync.Mutex
    tokens  int64
    syncing bool      // đang có goroutine đi xin lô
}

type TwoTier struct {
    rdb     *redis.Client
    buckets sync.Map    // tenant -> *bucket
    batch   int64       // kích thước lô — ĐẶT LỚN
    logger  *zap.Logger
}

func (t *TwoTier) Allow(ctx context.Context, tenant string, rate, burst int) bool {
    v, _ := t.buckets.LoadOrStore(tenant, &bucket{})
    b := v.(*bucket)

    b.mu.Lock()
    if b.tokens > 0 {
        b.tokens--
        b.mu.Unlock()
        return true            // ĐƯỜNG NÓNG: 0 round-trip, ~73 ns
    }
    if b.syncing {
        b.mu.Unlock()
        // CHỜ lô đang xin về rồi thử lại — ĐỪNG deny ngay.
        // Deny ngay là nguyên nhân của 84% false denial trong benchmark.
        return t.waitForTokens(ctx, b)
    }
    b.syncing = true
    b.mu.Unlock()

    // Gọi Redis NGOÀI lock, và chỉ MỘT lần cho cả lô
    give, err := takeBatch.Run(ctx, t.rdb,
        []string{"rl:" + tenant}, rate, burst, t.batch).Int64()

    b.mu.Lock()
    b.syncing = false
    if err != nil || give <= 0 {
        b.mu.Unlock()
        return false           // hết hạn mức toàn cục, hoặc Redis lỗi → fail closed
    }
    b.tokens += give - 1
    b.mu.Unlock()
    return true
}

func (t *TwoTier) waitForTokens(ctx context.Context, b *bucket) bool {
    // Chờ tối đa 10 ms — ngắn hơn nhiều so với p99 mục tiêu 50 ms
    deadline := time.Now().Add(10 * time.Millisecond)
    for time.Now().Before(deadline) {
        time.Sleep(50 * time.Microsecond)
        b.mu.Lock()
        if b.tokens > 0 {
            b.tokens--
            b.mu.Unlock()
            return true
        }
        still := b.syncing
        b.mu.Unlock()
        if !still {
            return false       // lô đã về nhưng không có token → thật sự hết hạn mức
        }
    }
    return false
}
```

Ba điểm thiết kế:

1. **Đường nóng không có round-trip nào.** Khi bucket cục bộ còn token, chi phí là một mutex + một phép trừ — khoảng 73 ns (đo ở phần auth bên dưới, cùng bậc độ lớn).
2. **Một lời gọi Redis cho cả lô.** Phiên bản đầu tôi viết gọi `takeBatch` một-token-một-lần trong vòng lặp 1.000 vòng **trong khi giữ mutex** — kết quả là 21.490 check/s, **chậm hơn** cả Redis-mỗi-request. Đó là lỗi tôi phải sửa, và nó cho thấy two-tier làm sai thì tệ hơn không làm.
3. **Chờ, đừng deny.** Chờ 10 ms rẻ hơn nhiều so với từ chối oan một request hợp lệ của khách hàng đang trả tiền.

### Đánh đổi phải chấp nhận

Two-tier **không chính xác tuyệt đối** khi có nhiều instance:

| | Redis mỗi request | Two-tier |
|---|---|---|
| Độ chính xác giới hạn | Tuyệt đối | **Có thể vượt tới `batch × số instance`** |
| Instance chết | Không ảnh hưởng | **Token trong lô bị mất** (tenant tạm bị thắt) |
| Scale | Cần thêm Redis | Gần như không giới hạn |

Với `batch = 10.000` và 20 instance, worst case bạn cho qua vượt 200.000 token trong một khoảnh khắc. Nên: **đặt `batch` theo tỉ lệ với hạn mức, không phải một số cố định** — ví dụ `batch = rate / 50`, tức mỗi lô bằng 20 ms hạn mức.

---

# Hot path #2 — Noisy neighbor

Đây là failure mode **riêng** của public API, không có ở flash sale (vì flash sale chỉ có một tenant).

**Tình huống**: một tenant gọi endpoint đắt (query nặng, export lớn) với volume cao. Nếu không có isolation, tenant đó **chiếm hết** connection pool / CPU / Redis, và mọi tenant khác chịu latency của nó.

Tôi đo: tenant "noisy" chiếm 90% traffic và gọi một operation đắt; tenant "quiet" chiếm 10% và gọi operation nhẹ.

| Cấu hình | noisy p99 | **quiet p99** |
|---|---:|---:|
| **Không isolate** | 28,35 ms | **28,33 ms** |
| **Per-tenant inflight cap** | 1,64 ms | **2,06 ms** |

Đọc dòng đầu: **quiet p99 = 28,33 ms, gần như y hệt noisy p99 = 28,35 ms.** Tenant nhẹ **thừa hưởng toàn bộ latency** của tenant nặng — đó chính là noisy neighbor, đo được.

Sau khi cap inflight riêng cho mỗi tenant, quiet về **2,06 ms — tốt hơn 13,7 lần**, chỉ nhờ việc giới hạn *tenant khác*.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="nn-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="420" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Noisy neighbor — tenant nhẹ thừa hưởng latency của tenant nặng</text>
<rect x="24" y="48" width="408" height="200" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#nn-sh)"/>
<text x="228" y="72" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">❌ Không isolate — pool DÙNG CHUNG</text>
<rect x="48" y="86" width="104" height="44" rx="7" fill="#DC2626"/>
<rect x="48" y="140" width="104" height="44" rx="7" fill="#14B8A6"/>
<text x="100" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">noisy</text>
<text x="100" y="120" text-anchor="middle" font-size="9" fill="#FFD9DE">90% · nặng</text>
<text x="100" y="158" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">quiet</text>
<text x="100" y="174" text-anchor="middle" font-size="9" fill="#CCFBF1">10% · nhẹ</text>
<rect x="190" y="98" width="86" height="74" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="233" y="128" text-anchor="middle" font-size="10" font-weight="800" fill="#9F1239">1 POOL</text>
<text x="233" y="144" text-anchor="middle" font-size="9" fill="#BE123C">dùng chung</text>
<line x1="156" y1="108" x2="186" y2="120" stroke="#DC2626" stroke-width="2"/>
<line x1="156" y1="162" x2="186" y2="150" stroke="#14B8A6" stroke-width="2"/>
<rect x="300" y="86" width="116" height="44" rx="7" fill="#FFD9DE" stroke="#DC2626" stroke-width="1.8"/>
<rect x="300" y="140" width="116" height="44" rx="7" fill="#FFD9DE" stroke="#DC2626" stroke-width="1.8"/>
<text x="358" y="104" text-anchor="middle" font-size="10" fill="#9F1239">noisy p99</text>
<text x="358" y="122" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">28,35 ms</text>
<text x="358" y="158" text-anchor="middle" font-size="10" fill="#9F1239">quiet p99</text>
<text x="358" y="176" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">28,33 ms</text>
<rect x="48" y="196" width="368" height="40" rx="8" fill="#FFD9DE"/>
<text x="232" y="214" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">Hai con số GẦN BẰNG NHAU</text>
<text x="232" y="230" text-anchor="middle" font-size="10" fill="#BE123C">→ tenant nhẹ không hưởng lợi gì từ việc nó nhẹ</text>
<rect x="448" y="48" width="408" height="200" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#nn-sh)"/>
<text x="652" y="72" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">✅ Per-tenant inflight cap</text>
<rect x="472" y="86" width="104" height="44" rx="7" fill="#DC2626"/>
<rect x="472" y="140" width="104" height="44" rx="7" fill="#14B8A6"/>
<text x="524" y="104" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">noisy</text>
<text x="524" y="120" text-anchor="middle" font-size="9" fill="#FFD9DE">cap 8 inflight</text>
<text x="524" y="158" text-anchor="middle" font-size="10" font-weight="700" fill="#FFFFFF">quiet</text>
<text x="524" y="174" text-anchor="middle" font-size="9" fill="#CCFBF1">cap 40 inflight</text>
<rect x="608" y="86" width="64" height="44" rx="6" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.6"/>
<rect x="608" y="140" width="64" height="44" rx="6" fill="#CCFBF1" stroke="#0D9488" stroke-width="1.6"/>
<text x="640" y="112" text-anchor="middle" font-size="9" font-weight="700" fill="#9F1239">pool A</text>
<text x="640" y="166" text-anchor="middle" font-size="9" font-weight="700" fill="#134E4A">pool B</text>
<line x1="580" y1="108" x2="604" y2="108" stroke="#DC2626" stroke-width="2"/>
<line x1="580" y1="162" x2="604" y2="162" stroke="#0D9488" stroke-width="2"/>
<rect x="700" y="86" width="132" height="44" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.8"/>
<rect x="700" y="140" width="132" height="44" rx="7" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="766" y="104" text-anchor="middle" font-size="10" fill="#9F1239">noisy p99</text>
<text x="766" y="122" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">1,64 ms</text>
<text x="766" y="158" text-anchor="middle" font-size="10" fill="#134E4A">quiet p99</text>
<text x="766" y="176" text-anchor="middle" font-size="13" font-weight="800" fill="#0F766E">2,06 ms</text>
<rect x="472" y="196" width="360" height="40" rx="8" fill="#CCFBF1"/>
<text x="652" y="214" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">quiet tốt hơn 13,7×</text>
<text x="652" y="230" text-anchor="middle" font-size="10" fill="#0F766E">chỉ nhờ việc giới hạn TENANT KHÁC</text>
<rect x="24" y="264" width="832" height="144" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="288" font-size="13" font-weight="800" fill="#374151">Bốn tầng isolation — rate limit MỘT MÌNH là không đủ</text>
<text x="44" y="312" font-size="11" fill="#57534E"><tspan font-weight="700" fill="#0F766E">① Rate limit theo tenant</tspan> — chặn SỐ LƯỢNG request. Không chặn được endpoint ĐẮT.</text>
<text x="44" y="334" font-size="11" fill="#57534E"><tspan font-weight="700" fill="#0F766E">② Inflight cap theo tenant</tspan> — chặn SỐ request đang xử lý. Đây là tầng hiệu quả nhất (đo được 13,7×).</text>
<text x="44" y="356" font-size="11" fill="#57534E"><tspan font-weight="700" fill="#0F766E">③ Cost-based limiting</tspan> — endpoint đắt trừ nhiều token hơn. Chặn được "ít request nhưng mỗi cái rất nặng".</text>
<text x="44" y="378" font-size="11" fill="#57534E"><tspan font-weight="700" fill="#0F766E">④ Tách hạ tầng theo tier</tspan> — enterprise có pool/node riêng. Đắt nhất, nhưng isolation thật sự.</text>
<text x="44" y="400" font-size="11" font-weight="800" fill="#DC2626">Chỉ có ① là sai lầm phổ biến nhất: tenant gửi 100 request/s nhưng mỗi cái mất 2 giây vẫn giết hệ thống.</text>
</svg>
</div>

## Code: inflight cap theo tenant

```go
// internal/presentation/middleware/tenant_isolation.go

type TenantIsolation struct {
    sems    sync.Map        // tenant -> chan struct{}
    byTier  map[string]int  // tier -> số inflight tối đa
    logger  *zap.Logger
}

func NewTenantIsolation(logger *zap.Logger) *TenantIsolation {
    return &TenantIsolation{
        byTier: map[string]int{"free": 4, "pro": 32, "enterprise": 256},
        logger: logger,
    }
}

func (t *TenantIsolation) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tenant := c.GetString(auth.ContextTenantID)
        tier := c.GetString(auth.ContextTier)

        cap, ok := t.byTier[tier]
        if !ok {
            cap = t.byTier["free"]
        }
        v, _ := t.sems.LoadOrStore(tenant, make(chan struct{}, cap))
        sem := v.(chan struct{})

        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        default:
            // Tenant này đã dùng hết hạn mức ĐỒNG THỜI của nó.
            // Quan trọng: đây là 429, KHÔNG phải 503 —
            // lỗi ở phía tenant, không phải hệ thống quá tải.
            metrics.TenantThrottled.WithLabelValues(tenant, tier).Inc()
            c.Header("Retry-After", "1")
            c.AbortWithStatusJSON(429, gin.H{
                "success": false,
                "error": gin.H{
                    "code":    "TOO_MANY_CONCURRENT_REQUESTS",
                    "message": fmt.Sprintf("tối đa %d request đồng thời cho tier %s", cap, tier),
                },
            })
            return
        }
        c.Next()
    }
}
```

Chi tiết phân biệt mã lỗi đáng nói: **429 khi tenant vượt hạn mức của chính nó, 503 khi hệ thống quá tải.** Trả 503 cho trường hợp đầu khiến tenant nghĩ lỗi ở bạn và... retry mạnh hơn. Trả 429 kèm `Retry-After` là tín hiệu đúng, và SDK tử tế sẽ tự backoff.

## Cost-based limiting

Tầng ① (đếm request) không chặn được "ít request nhưng mỗi cái rất nặng". Giải pháp: mỗi endpoint trừ **số token khác nhau**:

```go
// Chi phí theo endpoint — đo từ production, không phải đoán
var endpointCost = map[string]int64{
    "GET /v1/transactions":        1,
    "GET /v1/transactions/{id}":   1,
    "POST /v1/transactions":       2,
    "GET /v1/reports/monthly":    50,   // aggregation nặng
    "POST /v1/exports":           500,  // sinh file, chạy phút
}

// Lua trừ N token một lần, atomic
var takeCost = redis.NewScript(`
  local t = redis.call('TIME')
  local now = tonumber(t[1])*1000 + math.floor(tonumber(t[2])/1000)
  local d = redis.call('HMGET', KEYS[1], 'tk','ts')
  local tk, ts = tonumber(d[1]), tonumber(d[2])
  local rate, burst, cost = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])
  if tk == nil then tk = burst; ts = now end
  tk = math.min(burst, tk + math.max(0, now-ts)/1000.0*rate)
  if tk < cost then
    redis.call('HSET', KEYS[1], 'tk', tk, 'ts', now)
    return 0
  end
  redis.call('HSET', KEYS[1], 'tk', tk - cost, 'ts', now)
  redis.call('PEXPIRE', KEYS[1], 120000)
  return 1
`)
```

Nguyên tắc đặt `cost`: **đo p50 latency thật của endpoint rồi chuẩn hoá về endpoint nhẹ nhất.** Nếu `GET /transactions` mất 2 ms và `GET /reports/monthly` mất 100 ms, cost là 1 và 50. Đừng đặt theo cảm tính — và đừng quên cập nhật khi endpoint đổi.

---

# Hot path #3 — Chi phí xác thực

Ở 1M RPS, **mỗi request phải verify API key**. Nếu phép verify tốn 1 ms, bạn mất 1.000 core chỉ cho việc đó. Tôi đo 3 cách, 50.000 lần:

| Cách | **verify/s** | p50 | p99 |
|---|---:|---:|---:|
| ① **Redis `GET` mỗi request** | 93.697 | **1,20 ms** | 2,51 ms |
| ② HMAC-SHA256 (stateless) | **2.682.917** | 853 ns | 4 µs |
| ③ **Cache cục bộ (`sync.Map`)** | **2.773.244** | **73 ns** | ~0 |

Chênh lệch **~29 lần** giữa Redis và hai cách còn lại, và p50 chênh **16.400 lần** (1,20 ms vs 73 ns).

Ở 1M RPS:
- Redis mỗi request: cần **10,7 instance Redis** chỉ cho auth (1.000.000 ÷ 93.697)
- Cache cục bộ: **0 round-trip**, chi phí gần như không đo được

## Thiết kế auth đúng cho 1M RPS

```go
// internal/infrastructure/auth/keycache.go

type KeyInfo struct {
    TenantID string
    Tier     string
    Scopes   []string
    Revoked  bool
    cachedAt time.Time
}

type KeyCache struct {
    local     sync.Map          // hash(apiKey) -> *KeyInfo
    rdb       *redis.Client
    sf        singleflight.Group // chống cache stampede — xem P02
    ttl       time.Duration      // 60s: đủ ngắn để revoke kịp, đủ dài để rẻ
    logger    *zap.Logger
}

func (c *KeyCache) Lookup(ctx context.Context, apiKey string) (*KeyInfo, error) {
    h := sha256.Sum256([]byte(apiKey))
    k := hex.EncodeToString(h[:])

    // ĐƯỜNG NÓNG: ~73 ns, không mạng, không allocate
    if v, ok := c.local.Load(k); ok {
        info := v.(*KeyInfo)
        if time.Since(info.cachedAt) < c.ttl {
            if info.Revoked {
                return nil, ErrKeyRevoked
            }
            return info, nil
        }
    }

    // MISS: singleflight để 10.000 request cùng miss không tạo 10.000 query.
    // Đây đúng là bài toán cache stampede ở P02.
    v, err, _ := c.sf.Do(k, func() (any, error) {
        raw, err := c.rdb.Get(ctx, "apikey:"+k).Result()
        if err == redis.Nil {
            return nil, ErrKeyNotFound
        }
        if err != nil {
            return nil, err
        }
        var info KeyInfo
        if err := json.Unmarshal([]byte(raw), &info); err != nil {
            return nil, err
        }
        info.cachedAt = time.Now()
        c.local.Store(k, &info)
        return &info, nil
    })
    if err != nil {
        return nil, err
    }
    return v.(*KeyInfo), nil
}
```

### Cái giá của cache cục bộ: độ trễ thu hồi

TTL 60 giây nghĩa là **một API key bị thu hồi vẫn dùng được tới 60 giây** trên mỗi instance. Đây là đánh đổi thật và phải quyết định có ý thức — giống hệt bài toán thu hồi JWT ở [bài JWT](/posts/jwt/#23-bốn-cách-vá-và-giá-của-từng-cách).

Ba cách thu hẹp:

| Cách | Độ trễ thu hồi | Chi phí |
|---|---|---|
| Giảm TTL xuống 5 giây | ≤ 5 giây | Tăng 12× số lần tra Redis |
| **Pub/Sub thông báo thu hồi** | **~1 ms** | Thêm 1 subscriber/instance; mất tin nếu instance offline |
| Version key toàn cục | ≤ TTL, nhưng invalidate hàng loạt được | 1 lần tra version (cache được) |

> 💡 **Khuyến nghị: TTL 60 giây + Redis Pub/Sub để thu hồi tức thì.** Pub/Sub mất tin ([P03](/posts/system-design-p03-pubsub-vs-streams/)) nhưng ở đây không sao: TTL là lưới an toàn — tin thu hồi bị lỡ thì 60 giây sau cache cũng hết hạn.

```go
// Lắng nghe thông báo thu hồi — xoá ngay khỏi cache cục bộ
func (c *KeyCache) WatchRevocations(ctx context.Context) {
    sub := c.rdb.Subscribe(ctx, "apikey:revoked")
    defer sub.Close()
    for msg := range sub.Channel() {
        c.local.Delete(msg.Payload)   // payload = hash của key
        metrics.KeyRevocationApplied.Inc()
    }
}
```

---

# Kiến trúc tổng thể

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 540" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ap-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ap-sh2" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="540" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">1M RPS — mọi thứ trong hot path phải đo bằng nanosecond</text>
<rect x="24" y="42" width="852" height="62" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2"/>
<text x="44" y="64" font-size="12" font-weight="800" fill="#3730A3">① EDGE — TLS, chống DDoS, cache response công khai</text>
<text x="44" y="84" font-size="11" fill="#4338CA">Endpoint đọc dữ liệu công khai cache tại biên. Mỗi request chặn ở đây là một request không tốn gì.</text>
<rect x="690" y="52" width="170" height="42" rx="7" fill="#6366F1"/>
<text x="775" y="78" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">0 chi phí app</text>
<rect x="24" y="116" width="852" height="80" rx="10" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5"/>
<text x="44" y="138" font-size="12" font-weight="800" fill="#134E4A">② AUTH — cache cục bộ + singleflight khi miss</text>
<text x="44" y="158" font-size="11" fill="#0F766E">Cache hit: <tspan font-family="monospace" font-weight="700">73 ns</tspan>, không mạng. Miss: 1 lần Redis, gộp bằng singleflight.</text>
<text x="44" y="180" font-size="11" font-weight="700" fill="#134E4A">Nếu gọi Redis mỗi request: 93.697/s → cần 10,7 Redis chỉ cho auth.</text>
<rect x="690" y="130" width="170" height="52" rx="7" fill="#0D9488"/>
<text x="775" y="150" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">73 ns</text>
<text x="775" y="168" text-anchor="middle" font-size="10" fill="#CCFBF1">vs 1,20 ms (Redis)</text>
<rect x="24" y="208" width="852" height="80" rx="10" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5"/>
<text x="44" y="230" font-size="12" font-weight="800" fill="#7C4A03">③ RATE LIMIT — two-tier: bucket cục bộ + sync lô với Redis</text>
<text x="44" y="250" font-size="11" fill="#92400E">Đường nóng 0 round-trip. Hết lô → 1 lời gọi Lua lấy 10.000 token.</text>
<text x="44" y="272" font-size="11" font-weight="700" fill="#7C4A03">Nếu gọi Redis mỗi request: 50.574/s → cần 19,8 Redis chỉ cho rate limit.</text>
<rect x="690" y="222" width="170" height="52" rx="7" fill="#F5A623"/>
<text x="775" y="242" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">2.893.314/s</text>
<text x="775" y="260" text-anchor="middle" font-size="10" fill="#FFF3DB">20 lời gọi Redis</text>
<rect x="24" y="300" width="852" height="80" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2.5"/>
<text x="44" y="322" font-size="12" font-weight="800" fill="#9F1239">④ ISOLATION — inflight cap theo tenant + cost-based limiting</text>
<text x="44" y="342" font-size="11" fill="#BE123C">Chặn SỐ request đang xử lý, không chỉ số request/giây. Endpoint đắt trừ nhiều token hơn.</text>
<text x="44" y="364" font-size="11" font-weight="700" fill="#DC2626">Không có tầng này: tenant nhẹ có p99 = 28,33 ms, y hệt tenant nặng 28,35 ms.</text>
<rect x="690" y="314" width="170" height="52" rx="7" fill="#DC2626"/>
<text x="775" y="334" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">quiet tốt hơn</text>
<text x="775" y="352" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">13,7×</text>
<rect x="24" y="392" width="852" height="62" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="414" font-size="12" font-weight="800" fill="#374151">⑤ USAGE ACCOUNTING — đếm để tính tiền, KHÔNG nằm trong hot path</text>
<text x="44" y="434" font-size="11" fill="#57534E">Đếm cục bộ, flush định kỳ (1–10s) sang store bền. Mất 10 giây usage khi crash là chấp nhận được;</text>
<text x="44" y="450" font-size="11" fill="#57534E">thêm 1 ms vào mỗi request của 1M RPS thì không.</text>
<line x1="450" y1="106" x2="450" y2="112" stroke="#57534E" stroke-width="2" marker-end="url(#ap-a)"/>
<line x1="450" y1="198" x2="450" y2="204" stroke="#57534E" stroke-width="2" marker-end="url(#ap-a)"/>
<line x1="450" y1="290" x2="450" y2="296" stroke="#57534E" stroke-width="2" marker-end="url(#ap-a)"/>
<line x1="450" y1="382" x2="450" y2="388" stroke="#57534E" stroke-width="2" marker-end="url(#ap-a)"/>
<rect x="24" y="466" width="852" height="62" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="490" font-size="13" font-weight="800" fill="#7C4A03">Nguyên tắc chi phối: ở 1M RPS, MỖI round-trip mạng trong hot path là một ràng buộc kiến trúc</text>
<text x="44" y="514" font-size="11" fill="#92400E">Cộng dồn auth (19,8) + rate limit (10,7) = <tspan font-weight="700">30 instance Redis</tspan> nếu làm theo cách "đúng chuẩn sách". Two-tier + cache cục bộ đưa về 1–2.</text>
</svg>
</div>

## Usage accounting: đếm để tính tiền

Một chi tiết riêng của public API: bạn phải **đếm usage để tính tiền**, và nó không được nằm trong hot path.

```go
// internal/infrastructure/usage/counter.go

// Đếm cục bộ rồi flush theo lô. Mất tối đa `flushEvery` dữ liệu khi crash.
type Counter struct {
    mu      sync.Mutex
    counts  map[string]int64   // tenant:endpoint -> số request
    rdb     *redis.Client
    flushEvery time.Duration
}

func (c *Counter) Record(tenant, endpoint string, cost int64) {
    c.mu.Lock()
    c.counts[tenant+":"+endpoint] += cost
    c.mu.Unlock()           // ~50 ns, không mạng
}

func (c *Counter) Run(ctx context.Context) {
    t := time.NewTicker(c.flushEvery)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            c.flush(ctx)     // flush lần cuối khi shutdown
            return
        case <-t.C:
            c.flush(ctx)
        }
    }
}

func (c *Counter) flush(ctx context.Context) {
    c.mu.Lock()
    batch := c.counts
    c.counts = make(map[string]int64, len(batch))
    c.mu.Unlock()

    if len(batch) == 0 {
        return
    }
    // HINCRBY gộp — 1 pipeline cho hàng nghìn tenant
    pipe := c.rdb.Pipeline()
    hour := time.Now().Format("2006010215")
    for k, v := range batch {
        parts := strings.SplitN(k, ":", 2)
        pipe.HIncrBy(ctx, "usage:"+hour+":"+parts[0], parts[1], v)
        pipe.Expire(ctx, "usage:"+hour+":"+parts[0], 90*24*time.Hour)
    }
    if _, err := pipe.Exec(ctx); err != nil {
        // Thất bại → TRẢ LẠI vào counts để lần sau flush tiếp, không mất usage
        c.mu.Lock()
        for k, v := range batch {
            c.counts[k] += v
        }
        c.mu.Unlock()
    }
}
```

Hai chi tiết: **`Record` không bao giờ chạm mạng** (~50 ns), và khi flush thất bại thì **trả lại vào bộ đếm** chứ không bỏ — vì đây là dữ liệu tính tiền.

Đánh đổi phải nói rõ với bên business: **crash làm mất tối đa `flushEvery` giây usage.** Với `flushEvery = 5s` và 1M RPS, đó là 5 triệu request không được tính tiền. Nếu không chấp nhận được, phải đưa accounting vào hot path — và trả giá bằng latency.

---

# Failure mode tệ nhất

| Failure mode | Cơ chế | Phòng vệ |
|---|---|---|
| **Noisy neighbor** | 1 tenant chiếm pool → mọi tenant chịu p99 của nó (đo: 28,33 ms) | Inflight cap theo tenant (đo: tốt hơn 13,7×) |
| **Retry storm theo tenant** | SDK của tenant retry 3× khi chậm → tải ×4 từ đúng tenant đang gây vấn đề | Retry budget + trả **429** chứ không 503 |
| **Cache stampede khi deploy** | Instance mới khởi động, cache auth rỗng → mọi request miss | Singleflight ([P02](/posts/system-design-p02-cache-stampede/)) + warm cache trước khi nhận traffic |
| **Token lô bị mất khi instance chết** | Instance giữ 10.000 token chưa dùng thì chết → tenant bị thắt oan | Lô nhỏ hơn · TTL trên bucket Redis để token tự hồi |
| **Usage drift** | Flush thất bại lặp lại → số liệu tính tiền sai | Trả lại vào counter · đối chiếu định kỳ với log |
| **Key revoke không kịp** | Cache cục bộ TTL 60s → key bị thu hồi vẫn dùng được | Pub/Sub thu hồi tức thì, TTL là lưới an toàn |

Failure mode đáng lo nhất là **retry storm theo tenant**, vì nó là vòng lặp tự khuếch đại giống [P01.3](/posts/scalable-p01-3-rps-deep-dive/#failure-mode-tệ-nhất-retry-storm) nhưng tập trung vào **đúng tenant đang có vấn đề** — nên nó vừa làm hệ thống nặng hơn vừa làm trải nghiệm của tenant đó tệ hơn nữa.

Chặn nó bằng cách trả **mã lỗi đúng**:

```go
// Phân biệt rõ ràng, vì SDK của client hành xử khác nhau theo mã lỗi
switch {
case tenantOverRateLimit:
    // 429 + Retry-After: SDK tử tế sẽ backoff. Lỗi ở phía TENANT.
    c.Header("Retry-After", "1")
    c.AbortWithStatusJSON(429, errBody("RATE_LIMIT_EXCEEDED", quotaMsg))

case tenantOverConcurrency:
    c.Header("Retry-After", "1")
    c.AbortWithStatusJSON(429, errBody("TOO_MANY_CONCURRENT_REQUESTS", concMsg))

case systemOverloaded:
    // 503: lỗi ở phía HỆ THỐNG. Jitter do server sinh.
    c.Header("Retry-After", "2")
    c.AbortWithStatusJSON(503, gin.H{
        "error": gin.H{"code": "SERVICE_OVERLOADED"},
        "meta":  gin.H{"retry_after_ms": 1000 + rand.Intn(4000)},
    })

case badRequest:
    // 400: KHÔNG BAO GIỜ retry được. Nói rõ để SDK đừng thử lại.
    c.AbortWithStatusJSON(400, errBody("INVALID_REQUEST", detail))
}
```

Trả 503 cho lỗi vượt quota là sai lầm tốn kém: tenant nghĩ lỗi ở bạn, SDK retry mạnh hơn, và bạn tự tạo retry storm từ chính thông điệp lỗi của mình.

---

# Stress test và monitoring

## Cách test

1. **Test từng tầng riêng trước.** Ở 1M RPS, bottleneck có thể là auth, limiter, hay nghiệp vụ — và nếu test cả cùng lúc bạn sẽ không biết là cái nào. Benchmark riêng: auth (đo được 2,77M/s), limiter (2,89M/s), rồi mới ghép.
2. **Test với phân bố tenant thật.** Đừng test 10.000 tenant đều nhau. Lấy phân bố thật: thường là power law — tenant lớn nhất chiếm 30%, top 10 chiếm 80%. Chính tenant lớn nhất quyết định isolation.
3. **Test noisy neighbor có chủ đích.** Cho một tenant gọi endpoint đắt nhất với volume cao, rồi đo **p99 của tenant khác**. Nếu hai con số gần nhau, bạn chưa có isolation — như số đo 28,33 vs 28,35 ms.
4. **Test độ chính xác của limiter, không chỉ throughput.** Đây là bước hay bị bỏ qua nhất. Đặt limit 5.000/s, bắn 20.000 request, **đếm số được cho qua**. Tôi tìm ra lỗi two-tier deny oan 84% chính bằng bước này — nó không lộ ra ở phép đo throughput.
5. **Test cold start.** Deploy instance mới vào lúc tải đỉnh. Cache auth rỗng = mọi request miss. Nếu không có singleflight, bạn sẽ thấy ngay.
6. **Test revoke.** Thu hồi một key rồi đo sau bao lâu nó thật sự bị chặn trên **mọi** instance.

## Metric bắt buộc

| Nhóm | Metric | Vì sao |
|---|---|---|
| **Theo tenant** | p99 latency **theo tenant** (top N) | Trung bình toàn cục che hết noisy neighbor |
| | `tenant_throttled_total` theo tenant + lý do | Phân biệt rate limit vs concurrency vs quota |
| | `tenant_share` (% traffic) | Phát hiện tenant đang phình |
| **Limiter** | `limiter_local_hit_ratio` | **Phải > 99%.** Thấp = lô quá nhỏ |
| | `limiter_redis_calls/s` | So với RPS để biết tỉ lệ giảm thật |
| | **`limiter_false_deny`** | Deny khi hạn mức toàn cục CÒN. Phải ~0 |
| **Auth** | `auth_cache_hit_ratio` | < 99% ở steady state = TTL quá ngắn |
| | `auth_redis_calls/s` | Tăng vọt = cold start hoặc cache bị xoá |
| | `key_revocation_lag` | Thời gian từ revoke tới khi chặn thật |
| **Usage** | `usage_flush_failures` | Dữ liệu tính tiền đang bị kẹt |
| | `usage_pending_count` | Bao nhiêu bản ghi chưa flush |
| **Saturation** | `inflight_requests` theo tenant | Tín hiệu sớm nhất |

**Metric quan trọng nhất và hay thiếu nhất: `p99 theo tenant`.** p99 toàn cục có thể rất đẹp trong khi một tenang cụ thể đang chịu 10× latency — và họ sẽ là người gửi ticket. Đo theo tenant (ít nhất là top 20 theo volume) là cách duy nhất thấy được điều đó.

**Metric phải ~0: `limiter_false_deny`** — số lần từ chối trong khi hạn mức toàn cục vẫn còn. Nó là metric đúng đắn của two-tier limiter, tương đương `oversell_count` ở [P01.3](/posts/scalable-p01-3-rps-deep-dive/).

## Checklist

```
BỨC TƯỜNG REDIS
□ Đã tính số instance Redis cần nếu gọi mỗi request chưa? (đo được: 19,8 cho limiter + 10,7 cho auth)
□ Auth có cache cục bộ chưa? (73 ns vs 1,20 ms — chênh 16.400×)
□ Rate limit có two-tier chưa? Lấy cả LÔ trong MỘT lời gọi Lua?
□ Khi bucket cục bộ cạn: CHỜ lô đang xin, hay deny ngay? (deny ngay = false deny tới 84%)
□ batch có tỉ lệ với rate không? (gợi ý: rate/50 ≈ 20 ms hạn mức)
□ Cache miss có singleflight chưa? (cold start sẽ lộ ngay nếu không)

ISOLATION ĐA TENANT
□ Có inflight cap THEO TENANT chưa? (không chỉ rate limit theo số lượng)
□ Có cost-based limiting cho endpoint đắt chưa? Cost dựa trên ĐO hay đoán?
□ Đã test noisy neighbor và so p99 giữa các tenant chưa?
□ Tier enterprise có cần hạ tầng riêng không?

MÃ LỖI
□ 429 cho vượt quota của TENANT, 503 cho hệ thống quá tải — có phân biệt chưa?
□ Retry-After có jitter do SERVER sinh chưa?
□ 400 có nói rõ "không retry được" chưa?

ĐÚNG ĐẮN
□ limiter_false_deny có được đo không? Có ~0 không?
□ Usage flush thất bại có TRẢ LẠI vào counter không?
□ Đã nói rõ với business "crash mất tối đa N giây usage" chưa?
□ Revoke key có Pub/Sub tức thì, hay chỉ chờ TTL?

MONITORING
□ p99 THEO TENANT (top 20), không chỉ p99 toàn cục
□ limiter_local_hit_ratio > 99%
□ auth_cache_hit_ratio > 99%
□ inflight_requests theo tenant
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| Ràng buộc lớn nhất ở 1M RPS? | **Mỗi round-trip mạng trong hot path.** Gọi Redis mỗi request cần **19,8 instance** chỉ cho rate limit |
| Lời giải? | **Two-tier limiter**: bucket cục bộ + xin lô. Đo thật **2.893.314 check/s**, nhanh hơn **57×**, chỉ **20 lời gọi Redis** thay vì 200.000 |
| Two-tier làm sai thì sao? | Tệ hơn không làm. Phiên bản đầu của tôi gọi Redis 1.000 lần/sync trong khi giữ mutex → **21.490 check/s**, chậm hơn cả Redis-mỗi-request |
| Cái bẫy của two-tier? | **Deny oan.** Lô nhỏ + "deny ngay" từ chối tới **84%** traffic hợp lệ. Phải CHỜ lô đang xin |
| Chi phí auth? | Redis **1,20 ms** vs cache cục bộ **73 ns** — chênh **16.400×**. Ở 1M RPS, Redis-mỗi-request cần 10,7 instance |
| Noisy neighbor nghiêm trọng thế nào? | Đo thật: tenant nhẹ có p99 **28,33 ms**, y hệt tenant nặng **28,35 ms** — không hưởng lợi gì từ việc nó nhẹ |
| Cách sửa hiệu quả nhất? | **Inflight cap theo tenant** → tenant nhẹ tốt hơn **13,7×** |
| Rate limit theo số lượng đủ chưa? | **Không.** Tenant gửi 100 req/s nhưng mỗi cái 2 giây vẫn giết hệ thống → cần cost-based |
| 429 hay 503? | **429** khi tenant vượt hạn mức của mình, **503** khi hệ thống quá tải. Trả nhầm = tự tạo retry storm |
| Metric phải ~0? | **`limiter_false_deny`** |

---

# Tham khảo

- [Stripe — Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters) — 4 loại limiter trong production thật
- [Google SRE — Handling Overload](https://sre.google/sre-book/handling-overload/) · [Load Shedding](https://sre.google/sre-book/addressing-cascading-failures/)
- [Cloudflare — How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) — two-tier ở quy mô thật
- [`golang.org/x/time/rate`](https://pkg.go.dev/golang.org/x/time/rate) — token bucket cục bộ chuẩn của Go
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — CCU không phải RPS](/posts/scalable-p01-ccu-vs-rps/) · [P01.2 — 1M CCU gaming](/posts/scalable-p01-2-ccu-deep-dive/) · [P01.3 — Triệu RPS flash sale](/posts/scalable-p01-3-rps-deep-dive/) · [P01.4 — Chat 1M CCU](/posts/scalable-p01-4-chat-deep-dive/)
