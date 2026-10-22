+++
date = '2026-10-22T01:30:00+07:00'
draft = false
slug = 'scalable-p01-3-rps-deep-dive'
title = 'Scalable System Design P01.3: Thiết Kế Hệ Thống 1M RPS — Flash Sale'
author = 'Huy Dang Quang'
categories = ["System Design"]
tags = ["system-design", "scalability", "performance", "golang", "redis"]
description = 'Flash sale burst 50×: từng tầng component và tradeoff, 6 cách giữ tồn kho đo thật (có cách oversell 60 suất), rate limit đúng/sai, và retry storm.'
+++

> *Bài viết thuộc series **Scalable System Design**, nội dung tham khảo từ các bài viết trong group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619). Phần diễn giải, kiến trúc, code và **toàn bộ số liệu benchmark** là tôi tự bổ sung, đo thật trên máy local (Go 1.24, MySQL 8.0.40, Redis 7.4.2).*

# Mở đầu

[P01.2](/posts/scalable-p01-2-ccu-deep-dive/) đào sâu hệ thống **CCU-heavy** (1M connection gaming), nơi ràng buộc là bandwidth và latency. Bài này làm điều tương tự cho đầu còn lại: hệ thống **RPS-heavy** với tải **dồn thành burst** — flash sale.

Khác biệt cốt lõi giữa hai bài toán:

| | CCU-heavy (P01.2) | **RPS-heavy (bài này)** |
|---|---|---|
| Tải đến | Đều, dự báo được | **Burst 50× trong 10 giây** |
| Ràng buộc | Bandwidth, latency p99 | **Hot resource, tính đúng đắn** |
| Lỗi tệ nhất | Reconnect storm | **Oversell + retry storm** |
| Hot path | Fan-out trong RAM | **Giữ suất tồn kho** |
| Thứ phải bảo toàn | State đồng bộ | **Không bán âm tồn kho** |

Ở hệ thống CCU, sai là *lag*. Ở đây, sai là **mất tiền thật** — và đó là lý do bài này nói về tính đúng đắn nhiều hơn về throughput.

## Workload model

Shop có **1.000 suất iPhone giảm 50%**, mở bán lúc 12:00:00. Dự kiến **500.000 người** chờ sẵn.

| Chỉ số | Giá trị | Ghi chú |
|---|---:|---|
| Người chờ | 500.000 | biết trước (đăng ký nhắc hẹn) |
| Suất bán | **1.000** | tỉ lệ 1/500 |
| RPS nền (bình thường) | 10.000 | |
| **RPS đỉnh (giây đầu)** | **500.000** | **burst 50×** |
| Thời gian burst | ~10 giây | |
| Người chắc chắn thất bại | **499.000 (99,8%)** | |

Dòng cuối là góc nhìn quyết định toàn bộ kiến trúc: **99,8% request sẽ thất bại dù hệ thống hoàn hảo.** Vậy thì việc đầu tiên không phải "làm sao xử lý 500.000 RPS", mà là **"làm sao từ chối 499.000 người càng rẻ càng tốt"**.

---

# Kiến trúc tổng thể

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 640" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="fs-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<marker id="fs-r" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="fs-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="640" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Phễu lọc: mỗi tầng phải rẻ hơn tầng sau nó hàng chục lần</text>
<rect x="60" y="42" width="780" height="60" rx="10" fill="#EEF2FF" stroke="#6366F1" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="80" y="64" font-size="12" font-weight="800" fill="#3730A3">① CDN / EDGE — chặn ngay tại biên, không chạm origin</text>
<text x="80" y="84" font-size="11" fill="#4338CA">Trang landing, ảnh, JS, countdown: 100% cache. Chặn bot. Giữ ~70% traffic không bao giờ vào.</text>
<rect x="700" y="52" width="128" height="40" rx="7" fill="#6366F1"/>
<text x="764" y="68" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">500.000 RPS</text>
<text x="764" y="84" text-anchor="middle" font-size="10" fill="#E0E7FF">→ còn ~150.000</text>
<rect x="110" y="118" width="680" height="60" rx="10" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="130" y="140" font-size="12" font-weight="800" fill="#134E4A">② WAITING ROOM — phát vé xếp hàng, KHÔNG xử lý nghiệp vụ</text>
<text x="130" y="160" font-size="11" fill="#0F766E">Cấp token có thứ tự. Client poll nhẹ. Chỉ nhả vào trong theo nhịp hệ thống chịu được.</text>
<rect x="650" y="128" width="128" height="40" rx="7" fill="#0D9488"/>
<text x="714" y="144" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">150.000 RPS</text>
<text x="714" y="160" text-anchor="middle" font-size="10" fill="#CCFBF1">→ nhả 20.000/s</text>
<rect x="160" y="194" width="580" height="60" rx="10" fill="#FFFAF0" stroke="#F5A623" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="180" y="216" font-size="12" font-weight="800" fill="#7C4A03">③ RATE LIMIT + ADMISSION — bảo vệ tầng sau khỏi quá tải</text>
<text x="180" y="236" font-size="11" fill="#92400E">Token bucket phân tán (server time!). Load shedding khi p99 vỡ ngưỡng.</text>
<rect x="600" y="204" width="128" height="40" rx="7" fill="#D97706"/>
<text x="664" y="220" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">20.000 RPS</text>
<text x="664" y="236" text-anchor="middle" font-size="10" fill="#FFF3DB">→ còn 20.000</text>
<rect x="210" y="270" width="480" height="60" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="3" filter="url(#fs-sh)"/>
<text x="230" y="292" font-size="12" font-weight="800" fill="#9F1239">④ RESERVE — giữ suất tồn kho ⚠️ HOT RESOURCE</text>
<text x="230" y="312" font-size="11" font-weight="700" fill="#BE123C">1 lệnh Redis Lua atomic. Đây là điểm tuần tự hoá DUY NHẤT của cả hệ thống.</text>
<rect x="646" y="280" width="118" height="40" rx="7" fill="#DC2626"/>
<text x="695" y="296" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">chỉ 1.000</text>
<text x="695" y="312" text-anchor="middle" font-size="10" fill="#FFD9DE">được qua</text>
<rect x="260" y="346" width="380" height="60" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5" filter="url(#fs-sh)"/>
<text x="280" y="368" font-size="12" font-weight="800" fill="#134E4A">⑤ CHECKOUT ASYNC — qua queue, không đồng bộ</text>
<text x="280" y="388" font-size="11" fill="#0F766E">Đẩy vào Kafka. Worker xử lý payment + tạo đơn với idempotency key.</text>
<rect x="24" y="424" width="852" height="96" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="446" font-size="12" font-weight="800" fill="#374151">Chi phí xử lý MỘT request ở mỗi tầng — đây là lý do thứ tự các tầng quan trọng</text>
<rect x="44" y="456" width="194" height="52" rx="7" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="250" y="456" width="194" height="52" rx="7" fill="#F2FCFB" stroke="#0D9488" stroke-width="1.6"/>
<rect x="456" y="456" width="194" height="52" rx="7" fill="#FFFAF0" stroke="#F5A623" stroke-width="1.6"/>
<rect x="662" y="456" width="194" height="52" rx="7" fill="#FFE9EC" stroke="#DC2626" stroke-width="1.6"/>
<text x="141" y="476" text-anchor="middle" font-size="11" font-weight="700" fill="#3730A3">CDN</text>
<text x="141" y="496" text-anchor="middle" font-size="11" font-weight="800" fill="#4338CA">~0 (không tới app)</text>
<text x="347" y="476" text-anchor="middle" font-size="11" font-weight="700" fill="#134E4A">Waiting room</text>
<text x="347" y="496" text-anchor="middle" font-size="11" font-weight="800" fill="#0F766E">1 Redis op</text>
<text x="553" y="476" text-anchor="middle" font-size="11" font-weight="700" fill="#7C4A03">Rate limit</text>
<text x="553" y="496" text-anchor="middle" font-size="11" font-weight="800" fill="#92400E">1 Lua call</text>
<text x="759" y="476" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">Reserve + checkout</text>
<text x="759" y="496" text-anchor="middle" font-size="11" font-weight="800" fill="#BE123C">Lua + Kafka + DB</text>
<line x1="450" y1="104" x2="450" y2="114" stroke="#57534E" stroke-width="2" marker-end="url(#fs-a)"/>
<line x1="450" y1="180" x2="450" y2="190" stroke="#57534E" stroke-width="2" marker-end="url(#fs-a)"/>
<line x1="450" y1="256" x2="450" y2="266" stroke="#57534E" stroke-width="2" marker-end="url(#fs-a)"/>
<line x1="450" y1="332" x2="450" y2="342" stroke="#57534E" stroke-width="2" marker-end="url(#fs-a)"/>
<rect x="24" y="536" width="852" height="94" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="44" y="560" font-size="13" font-weight="800" fill="#7C4A03">Nguyên tắc thiết kế chi phối tất cả: TỪ CHỐI SỚM VÀ RẺ</text>
<text x="44" y="584" font-size="11" fill="#92400E">99,8% request chắc chắn thất bại. Để chúng đi tới tầng ④ rồi mới từ chối là tự hại mình —</text>
<text x="44" y="602" font-size="11" fill="#92400E">mỗi request đi sâu một tầng là một lần tốn tài nguyên mà <tspan font-weight="700">kết quả vẫn là "hết hàng"</tspan>.</text>
<text x="44" y="622" font-size="11" font-weight="800" fill="#DC2626">Thiết kế sai phổ biến nhất: cho cả 500.000 request vào tới database rồi mới kiểm tra tồn kho.</text>
</svg>
</div>

---

# Lựa chọn component và tradeoff

## Tầng 2 — Waiting room

| Cách | Ưu | Nhược |
|---|---|---|
| **Không có** (ai nhanh người đó được) | Đơn giản nhất | 500.000 RPS đập thẳng vào app; trải nghiệm tệ (bấm mãi không biết sao) |
| **Token thứ tự + client poll** | Công bằng theo thứ tự đến, client biết vị trí, nhả tải theo nhịp chịu được | Cần Redis ZSET/Stream; client phải poll |
| **Virtual queue qua CDN** (Cloudflare Waiting Room, Fastly) | Chặn **ngay tại biên**, app không thấy gì | Phụ thuộc vendor, tốn phí, khó tuỳ biến |
| **Pre-token** (phát vé TRƯỚC giờ mở bán) | **Đỉnh biến mất hoàn toàn** — không còn burst | Chỉ áp dụng được khi biết trước thời điểm |

> 💡 **Khuyến nghị**: nếu biết trước giờ mở bán — **phát vé trước**. Đây là kỹ thuật mạnh nhất và rẻ nhất, vì nó **xoá bỏ burst** thay vì chịu đựng burst. Mở đăng ký 24 giờ trước, phát cho mỗi người một `slot_time` ngẫu nhiên trong cửa sổ 60 giây. 500.000 người rải trên 60 giây = **8.333 RPS** thay vì 500.000.
>
> Đây đúng là ý tưởng **jitter** ở [P01.2](/posts/scalable-p01-2-ccu-deep-dive/#hot-path-3--reconnect-storm-thundering-herd), áp vào phía nghiệp vụ: đo được ở đó là giảm đỉnh **65 lần**.

## Tầng 4 — Giữ tồn kho: 6 cách, và chỉ 5 cách đúng

Đây là hot path thật. Tôi đo cả **throughput** lẫn **tính đúng đắn**: 20.000 người bấm mua, **đúng 1.000 suất**, 120 worker đồng thời. Bất kỳ kết quả > 1.000 là **oversell** — tức bán hàng không có.

| Cách | Throughput | Cấp ra | Kết luận |
|---|---:|---:|---|
| ① MySQL `UPDATE ... WHERE qty > 0` | 6.026 ops/s | 1.000 | ✅ đúng, nhưng p99 **308 ms** |
| ② Redis `GET` rồi `DECR` (2 lệnh) | **83.852 ops/s** | **1.060** | ❌ **OVERSELL 60 suất** |
| ③ Redis `DECR` rồi kiểm tra, âm thì `INCR` bù | 49.116 ops/s | 1.000 | ✅ đúng, nhưng 2 round-trip |
| ④ **Redis Lua atomic check+DECR** | **83.313 ops/s** | 1.000 | ✅ **nhanh như ② và ĐÚNG** |
| ⑤ Redis 16 shard + Lua | 5.452 ops/s | 1.000 | ✅ đúng nhưng **chậm bất ngờ** |
| ⑥ **Redis `LPOP` từ túi token đúc sẵn** | **95.863 ops/s** | 1.000 | ✅ **nhanh nhất** |

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 440" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="iv-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="440" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">6 cách giữ 1.000 suất — throughput KHÔNG phải tiêu chí duy nhất</text>
<text x="450" y="48" text-anchor="middle" font-size="11" fill="#78716C">20.000 người bấm mua · 120 worker đồng thời · đo thật</text>
<line x1="150" y1="300" x2="810" y2="300" stroke="#57534E" stroke-width="2"/>
<line x1="150" y1="300" x2="150" y2="70" stroke="#57534E" stroke-width="2"/>
<rect x="172" y="286" width="72" height="14" rx="3" fill="#F5A623"/>
<rect x="276" y="98" width="72" height="202" rx="3" fill="#DC2626"/>
<rect x="380" y="182" width="72" height="118" rx="3" fill="#14B8A6"/>
<rect x="484" y="99" width="72" height="201" rx="3" fill="#0D9488"/>
<rect x="588" y="287" width="72" height="13" rx="3" fill="#F5A623"/>
<rect x="692" y="69" width="72" height="231" rx="3" fill="#0D9488"/>
<text x="208" y="278" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">6.026</text>
<text x="312" y="90" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">83.852</text>
<text x="416" y="174" text-anchor="middle" font-size="11" font-weight="800" fill="#0F766E">49.116</text>
<text x="520" y="91" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">83.313</text>
<text x="624" y="279" text-anchor="middle" font-size="11" font-weight="800" fill="#7C4A03">5.452</text>
<text x="728" y="61" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">95.863</text>
<text x="208" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">① MySQL</text>
<text x="208" y="334" text-anchor="middle" font-size="9" fill="#7C4A03">✅ p99 308ms</text>
<text x="312" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">② GET+DECR</text>
<text x="312" y="334" text-anchor="middle" font-size="9" font-weight="800" fill="#DC2626">❌ OVERSELL</text>
<text x="416" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">③ DECR+bù</text>
<text x="416" y="334" text-anchor="middle" font-size="9" fill="#0F766E">✅ 2 RTT</text>
<text x="520" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">④ Lua atomic</text>
<text x="520" y="334" text-anchor="middle" font-size="9" font-weight="800" fill="#0F766E">✅ TỐT NHẤT</text>
<text x="624" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">⑤ 16 shard</text>
<text x="624" y="334" text-anchor="middle" font-size="9" fill="#7C4A03">✅ nhưng chậm</text>
<text x="728" y="320" text-anchor="middle" font-size="10" font-weight="700" fill="#374151">⑥ LPOP token</text>
<text x="728" y="334" text-anchor="middle" font-size="9" font-weight="800" fill="#0F766E">✅ nhanh nhất</text>
<text x="116" y="185" text-anchor="middle" font-size="11" font-weight="700" fill="#57534E" transform="rotate(-90 116 185)">ops/giây</text>
<rect x="258" y="356" width="108" height="26" rx="6" fill="#DC2626"/>
<text x="312" y="374" text-anchor="middle" font-size="11" font-weight="800" fill="#FFFFFF">bán 1.060/1.000</text>
<rect x="24" y="394" width="852" height="38" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="450" y="418" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">Cách ② nhanh nhất nhóm Redis cơ bản — và bán 60 suất không tồn tại. Tốc độ mua bằng tính đúng đắn là không mua gì cả.</text>
<rect x="150" y="70" width="660" height="2" fill="#D6D3D1"/>
</svg>
</div>

### Vì sao ② oversell — và vì sao nó rất khó phát hiện

```go
// ❌ SAI — race window giữa GET và DECR
cur, _ := rdb.Get(ctx, "flash:stock").Int64()
if cur <= 0 {
    return errSoldOut
}
rdb.Decr(ctx, "flash:stock")   // ← giữa 2 lệnh này có 120 goroutine khác chen vào
return nil
```

Hai lệnh, mỗi lệnh atomic, nhưng **cặp lệnh thì không**. Khi `qty = 1`, có thể 61 goroutine cùng đọc được `1`, cả 61 đều thấy `> 0`, và cả 61 đều `DECR`. Kết quả đo: **bán 1.060 suất khi chỉ có 1.000**.

Điểm đáng sợ: **test với 1 user hay 10 user sẽ luôn PASS.** Lỗi chỉ xuất hiện ở mức đồng thời cao — tức là đúng lúc flash sale đang chạy.

### Cách đúng: một lệnh Lua atomic

```go
// Lua chạy như MỘT lệnh đơn trên Redis — không ai chen vào được.
var reserveScript = redis.NewScript(`
  local cur = tonumber(redis.call('GET', KEYS[1]) or '0')
  if cur <= 0 then
    return 0
  end
  redis.call('DECR', KEYS[1])
  return 1
`)

func (s *FlashSaleService) Reserve(ctx context.Context, userID, saleID string) error {
    ok, err := reserveScript.Run(ctx, s.rdb,
        []string{"flash:stock:" + saleID}).Int64()
    if err != nil {
        // Redis lỗi → TỪ CHỐI. Không bao giờ "cho qua cho chắc".
        return shared.NewDomainError(shared.ErrCodeUnavailable, "hệ thống đang quá tải")
    }
    if ok != 1 {
        return shared.NewDomainError(shared.ErrCodeSoldOut, "đã hết suất")
    }
    return nil
}
```

**Đo thật: ④ đạt 83.313 ops/s, gần y hệt ② (83.852 ops/s) — tính đúng đắn ở đây gần như MIỄN PHÍ.** Không có lý do nào để chọn ② nữa.

### Cách nhanh nhất: túi token đúc sẵn

Thay vì một bộ đếm, đúc sẵn **đúng 1.000 token** vào một list, rồi `LPOP`:

```go
// Lúc CHUẨN BỊ sale (trước giờ mở bán) — đúc đúng số suất
func (s *FlashSaleService) MintTokens(ctx context.Context, saleID string, qty int) error {
    key := "flash:tokens:" + saleID
    s.rdb.Del(ctx, key)
    toks := make([]interface{}, 0, qty)
    for i := 0; i < qty; i++ {
        toks = append(toks, fmt.Sprintf("%s-%d-%s", saleID, i, uuid.NewString()))
    }
    return s.rdb.RPush(ctx, key, toks...).Err()
}

// Lúc BÁN — 1 lệnh, atomic sẵn, và trả về MỘT TOKEN DUY NHẤT
func (s *FlashSaleService) Reserve(ctx context.Context, saleID string) (string, error) {
    tok, err := s.rdb.LPop(ctx, "flash:tokens:"+saleID).Result()
    if err == redis.Nil {
        return "", shared.NewDomainError(shared.ErrCodeSoldOut, "đã hết suất")
    }
    if err != nil {
        return "", shared.NewDomainError(shared.ErrCodeUnavailable, "hệ thống đang quá tải")
    }
    return tok, nil   // ← token này chính là idempotency key cho bước checkout
}
```

Ba lợi ích ngoài tốc độ (95.863 ops/s, nhanh nhất trong 6 cách):

1. **`LPOP` atomic sẵn** — không cần Lua, không thể viết sai.
2. **Token là bằng chứng giữ suất.** Nó đi cùng request xuống bước checkout và **chính là idempotency key** — giải quyết luôn bài toán ở phần sau.
3. **`LLEN` cho biết còn bao nhiêu suất** mà không cần đếm gì.

Đánh đổi: không co giãn được số lượng sau khi đúc (muốn thêm suất phải `RPUSH` thêm), và tốn RAM theo số suất — với 1.000 suất thì không đáng kể.

### Vì sao ⑤ sharding lại CHẬM — bài học phản trực giác

Sharding bộ đếm là lời khuyên kinh điển cho hot key. Nhưng đo thật nó **chậm nhất nhóm Redis: 5.452 ops/s**, chậm hơn Lua đơn 15 lần. Lý do:

```go
// Khi shard của mình hết, phải thăm dò các shard khác
for k := 0; k < nShards; k++ {
    sh := (i + k) % nShards
    if ok := tryShard(sh); ok {
        return true
    }
}
return false   // ← 19.000/20.000 người đi hết 16 shard rồi mới biết hết hàng
```

**19.000 trong 20.000 người đến khi mọi shard đã rỗng** — mỗi người phải trả **16 round-trip** để biết điều đó. Chi phí thăm dò lấn hết lợi ích giảm contention.

👉 **Bài học**: sharding bộ đếm chỉ đáng khi **bộ đếm thật sự là bottleneck và nó không cạn** (ví dụ đếm view, đếm like — tăng mãi không hết). Với tồn kho — thứ **cạn rất nhanh và phần lớn request đến khi đã cạn** — sharding làm hại nhiều hơn làm lợi.

### Và vì sao ① MySQL không tệ như tưởng

6.026 ops/s, cao hơn nhiều so với **387 ops/s** tôi đo ở [P01](/posts/scalable-p01-ccu-vs-rps/#hot-resource-chỗ-ccu-hoàn-toàn-không-nói-gì). Khác biệt: ở P01 tồn kho là 10 triệu nên **mọi** request đều lấy write lock; ở đây tồn kho chỉ 1.000 nên 19.000 request có `qty > 0` sai và **không lấy lock nào**.

Nghĩa là: **contention trên hot row chỉ tồn tại trong lúc còn hàng.** Với flash sale 1.000 suất, cửa sổ đau đớn chỉ là ~1.000 request đầu. Nhưng p99 **308 ms** vẫn quá cao cho burst 500.000 RPS — và đó là lý do vẫn cần Redis ở phía trước.

---

# Hot path #2 — Rate limit: đúng quan trọng hơn nhanh

Rate limit là tầng bảo vệ, nên **nó sai thì mọi tầng sau đều vô nghĩa**. Tôi đo 5 implementation với giới hạn 5.000 req/s:

| Implementation | Throughput | Cho qua | Kỳ vọng | Kết luận |
|---|---:|---:|---:|---|
| Fixed window (`INCR` + `EXPIRE`) | **88.477/s** | 5.000 | ~6.130 | ✅ đúng trong cửa sổ |
| Sliding window — pipeline | 71.911/s | 5.000 | ~6.391 | ✅ trong test này |
| Sliding window — Lua atomic | 63.237/s | 5.000 | ~6.581 | ✅ |
| **Token bucket — CLIENT time** | 63.112/s | **9.187** | ~6.584 | ❌ **vượt +40%** |
| **Token bucket — SERVER time** | 58.579/s | **6.700** | ~6.707 | ✅ **khớp gần tuyệt đối** |

## Lỗi #1: dùng thời gian của client

```lua
-- ❌ SAI: nowMs do CLIENT truyền vào
local nowMs = tonumber(ARGV[3])
tokens = math.min(burst, tokens + (nowMs - ts)/1000.0 * rate)
```

Với 120 client gọi đồng thời, timestamp chúng truyền vào **không đơn điệu** — request đến sau có thể mang timestamp cũ hơn. Mỗi lần như vậy bucket được bù thêm token không đáng có. Đo thật: **cho qua 9.187 thay vì ~6.584, vượt 40%**.

```lua
-- ✅ ĐÚNG: lấy thời gian từ CHÍNH Redis
local t = redis.call('TIME')
local nowMs = tonumber(t[1])*1000 + math.floor(tonumber(t[2])/1000)
```

Kết quả sau khi sửa: **6.700 vs kỳ vọng 6.707** — lệch 0,1%.

> ⚠️ **`redis.call('TIME')` khiến script thành non-deterministic.** Với Redis < 5 dùng replication kiểu command-propagation, điều này gây lệch giữa master và replica. Từ Redis 5 trở lên script được replicate theo **effect** nên an toàn. Nếu đang chạy Redis cũ, dùng `redis.replicate_commands()` ở đầu script.

## Lỗi #2: fixed window và lỗ hổng ranh giới

Fixed window nhanh nhất (88.477/s) nhưng có lỗi kinh điển. Tôi đo trực tiếp: gửi 1.000 request ở **cuối** giây N, rồi 1.000 request ở **đầu** giây N+1, hai burst cách nhau ~220 ms.

| Thuật toán | Burst 1 | Burst 2 | **Tổng trong ~0,4 giây** |
|---|---:|---:|---:|
| **Fixed window** | 1.000 | 1.000 | **2.000 — vượt 100% giới hạn** ❌ |
| Sliding window log | 1.000 | **0** | **1.000 — đúng** ✅ |

Fixed window cho qua **đúng gấp đôi** giới hạn trong một cửa sổ 1 giây, vì bộ đếm reset theo giây tròn. Kẻ tấn công biết điều này sẽ canh đúng ranh giới.

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<filter id="rw-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Lỗ hổng ranh giới cửa sổ — đo thật, giới hạn 1.000 req/s</text>
<rect x="24" y="46" width="832" height="150" rx="12" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#rw-sh)"/>
<text x="44" y="70" font-size="13" font-weight="800" fill="#9F1239">❌ FIXED WINDOW — bộ đếm reset theo giây tròn</text>
<line x1="120" y1="150" x2="790" y2="150" stroke="#57534E" stroke-width="2"/>
<line x1="400" y1="96" x2="400" y2="170" stroke="#DC2626" stroke-width="3"/>
<text x="400" y="88" text-anchor="middle" font-size="11" font-weight="800" fill="#DC2626">ranh giới giây</text>
<rect x="300" y="110" width="96" height="36" rx="5" fill="#DC2626"/>
<rect x="404" y="110" width="96" height="36" rx="5" fill="#DC2626"/>
<text x="348" y="133" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">1.000</text>
<text x="452" y="133" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">1.000</text>
<text x="348" y="166" text-anchor="middle" font-size="10" fill="#BE123C">cuối giây N</text>
<text x="452" y="166" text-anchor="middle" font-size="10" fill="#BE123C">đầu giây N+1</text>
<rect x="296" y="178" width="208" height="4" rx="2" fill="#DC2626"/>
<text x="400" y="192" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">~0,4 giây</text>
<rect x="540" y="104" width="296" height="80" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="688" y="128" text-anchor="middle" font-size="13" font-weight="800" fill="#9F1239">TỔNG 2.000 req</text>
<text x="688" y="150" text-anchor="middle" font-size="11" fill="#BE123C">trong 1 cửa sổ 1 giây bất kỳ</text>
<text x="688" y="172" text-anchor="middle" font-size="12" font-weight="800" fill="#DC2626">= VƯỢT 100% giới hạn</text>
<rect x="24" y="208" width="832" height="140" rx="12" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#rw-sh)"/>
<text x="44" y="232" font-size="13" font-weight="800" fill="#134E4A">✅ SLIDING WINDOW LOG — cửa sổ trượt theo từng request</text>
<line x1="120" y1="306" x2="790" y2="306" stroke="#57534E" stroke-width="2"/>
<rect x="300" y="268" width="96" height="34" rx="5" fill="#0D9488"/>
<rect x="404" y="296" width="96" height="6" rx="3" fill="#A8A29E"/>
<text x="348" y="290" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">1.000</text>
<text x="452" y="288" text-anchor="middle" font-size="12" font-weight="800" fill="#78716C">0</text>
<text x="348" y="322" text-anchor="middle" font-size="10" fill="#0F766E">cuối giây N</text>
<text x="452" y="322" text-anchor="middle" font-size="10" fill="#78716C">đầu giây N+1 — CHẶN HẾT</text>
<rect x="540" y="248" width="296" height="80" rx="9" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="688" y="272" text-anchor="middle" font-size="13" font-weight="800" fill="#134E4A">TỔNG 1.000 req</text>
<text x="688" y="294" text-anchor="middle" font-size="11" fill="#0F766E">Không có ranh giới để lợi dụng</text>
<text x="688" y="316" text-anchor="middle" font-size="12" font-weight="800" fill="#0F766E">= ĐÚNG giới hạn</text>
<text x="440" y="376" text-anchor="middle" font-size="12" font-weight="700" fill="#7C4A03">Đánh đổi: fixed window nhanh hơn 40% (88.477 vs 63.237 check/s) nhưng cho qua gấp đôi ở ranh giới.</text>
</svg>
</div>

**Chọn thuật toán nào?**

| | Fixed window | Sliding window log | **Token bucket** |
|---|---|---|---|
| Throughput đo được | 88.477/s | 63.237/s | 58.579/s |
| Bộ nhớ | 1 counter | **N entry/window** 💥 | 2 field |
| Chính xác ở ranh giới | ❌ vượt 2× | ✅ | ✅ |
| Cho phép burst có kiểm soát | ❌ | ❌ | ✅ (tham số `burst`) |
| Phù hợp | Chặn thô, chi phí thấp | Cần chính xác tuyệt đối | **Mặc định nên dùng** |

> 💡 **Khuyến nghị: token bucket với server time.** Nó chậm hơn fixed window 34% nhưng chính xác, cho phép burst có kiểm soát (quan trọng với traffic thật — người dùng không đến đều), và bộ nhớ cố định. Sliding window log tốn bộ nhớ tỉ lệ với số request trong cửa sổ — ở 500.000 RPS thì đó là 500.000 entry ZSET mỗi giây.

---

# Hot path #3 — Idempotency

Request đã giữ được suất nhưng checkout có thể thất bại giữa đường: client timeout rồi retry, worker crash trước khi ACK, payment gateway trả về chậm. Nếu không chống trùng, **một suất biến thành hai đơn hàng**.

Đo thật: 20.000 request trong đó mỗi order được gọi ~3 lần:

| | Kết quả |
|---|---|
| Throughput | **91.241 ops/s** |
| p50 / p99 | 1,24 ms / 2,59 ms |
| Số lần trùng bị chặn | **13.334** |

Chi phí thực tế: thêm **~1,2 ms** cho mỗi request. Với ngân sách burst, đó là cái giá rất rẻ cho việc không bán trùng.

```go
// internal/application/flashsale/service.go

func (s *Service) Checkout(ctx context.Context, cmd CheckoutCommand) (Result, error) {
    // cmd.ReserveToken chính là token LPOP được ở bước Reserve —
    // nó đã đảm bảo duy nhất ở mức Redis, nên dùng luôn làm idempotency key.
    key := "idem:checkout:" + cmd.ReserveToken

    ok, err := s.rdb.SetNX(ctx, key, "processing", 24*time.Hour).Result()
    if err != nil {
        return Result{}, shared.NewDomainError(shared.ErrCodeUnavailable, "hệ thống quá tải")
    }
    if !ok {
        // Đã có người xử lý token này. Trả về kết quả CŨ, không tạo đơn mới.
        if prev, err := s.rdb.Get(ctx, key+":result").Result(); err == nil && prev != "" {
            return decodeResult(prev), nil
        }
        // Đang xử lý dở — bảo client chờ, KHÔNG tạo đơn thứ hai.
        return Result{}, shared.NewDomainError(shared.ErrCodeInProgress, "đơn đang được xử lý")
    }

    res, err := s.createOrderAndCharge(ctx, cmd)
    if err != nil {
        // Thất bại → NHẢ khoá để lần sau làm lại được.
        // Quên bước này = token đó vĩnh viễn không bao giờ thành đơn.
        s.rdb.Del(ctx, key)
        return Result{}, err
    }

    s.rdb.Set(ctx, key+":result", encodeResult(res), 24*time.Hour)
    s.rdb.Set(ctx, key, "done", 24*time.Hour)
    return res, nil
}
```

Nhưng Redis chỉ là lớp nhanh. **Lớp bảo đảm cuối cùng phải ở database:**

```sql
CREATE TABLE orders (
  id             BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id        BIGINT       NOT NULL,
  sale_id        VARCHAR(64)  NOT NULL,
  reserve_token  VARCHAR(128) NOT NULL,
  amount         BIGINT       NOT NULL,          -- minor units, không dùng float
  created_at     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_orders_reserve_token (reserve_token)   -- ← lớp bảo đảm CUỐI
) ENGINE=InnoDB;
```

```go
// Lỗi duplicate key KHÔNG phải lỗi — nó là tín hiệu "đã làm rồi"
_, err := tx.ExecContext(ctx,
    `INSERT INTO orders (user_id, sale_id, reserve_token, amount) VALUES (?,?,?,?)`,
    cmd.UserID, cmd.SaleID, cmd.ReserveToken, cmd.Amount)
if err != nil {
    var me *mysql.MySQLError
    if errors.As(err, &me) && me.Number == 1062 { // ER_DUP_ENTRY
        existing, err := s.repo.GetByReserveToken(ctx, cmd.ReserveToken)
        if err != nil {
            return Result{}, err
        }
        return toResult(existing), nil   // trả đơn đã có, không tạo đơn mới
    }
    return Result{}, err
}
```

Vì sao cần **cả hai lớp**: Redis có thể mất khoá (restart, eviction, TTL hết); unique constraint trong DB thì không bao giờ. Redis cho tốc độ, DB cho bảo đảm.

---

# Failure mode tệ nhất: retry storm

Burst 500.000 RPS không phải thứ giết hệ thống. Thứ giết hệ thống là **vòng lặp phản hồi dương** giữa độ chậm và retry.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 480" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="rt-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<marker id="rt-t" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="rt-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="480" fill="#FFFDF7" rx="16"/>
<text x="450" y="28" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Retry storm — vòng lặp tự khuếch đại</text>
<circle cx="450" cy="170" r="104" fill="none" stroke="#DC2626" stroke-width="2.5" stroke-dasharray="8 6"/>
<rect x="356" y="52" width="188" height="52" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#rt-sh)"/>
<rect x="596" y="144" width="188" height="52" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#rt-sh)"/>
<rect x="356" y="236" width="188" height="52" rx="9" fill="#FFD9DE" stroke="#DC2626" stroke-width="2" filter="url(#rt-sh)"/>
<rect x="116" y="144" width="188" height="52" rx="9" fill="#FFE9EC" stroke="#DC2626" stroke-width="2" filter="url(#rt-sh)"/>
<text x="450" y="74" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">① Burst 500.000 RPS</text>
<text x="450" y="92" text-anchor="middle" font-size="10" fill="#BE123C">latency bắt đầu tăng</text>
<text x="690" y="166" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">② Client timeout</text>
<text x="690" y="184" text-anchor="middle" font-size="10" fill="#BE123C">p99 vượt timeout 2s</text>
<text x="450" y="258" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">③ Client RETRY</text>
<text x="450" y="276" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">tải tăng 2×, 3×...</text>
<text x="210" y="166" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">④ Càng chậm hơn</text>
<text x="210" y="184" text-anchor="middle" font-size="10" fill="#BE123C">pool cạn, queue dài</text>
<path d="M548,82 C600,96 648,118 676,140" stroke="#DC2626" stroke-width="2.2" fill="none" marker-end="url(#rt-a)"/>
<path d="M676,200 C650,224 600,244 548,258" stroke="#DC2626" stroke-width="2.2" fill="none" marker-end="url(#rt-a)"/>
<path d="M352,258 C300,244 250,224 224,200" stroke="#DC2626" stroke-width="2.2" fill="none" marker-end="url(#rt-a)"/>
<path d="M224,140 C250,118 300,96 352,82" stroke="#DC2626" stroke-width="2.2" fill="none" marker-end="url(#rt-a)"/>
<text x="450" y="166" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">TẢI THẬT</text>
<text x="450" y="186" text-anchor="middle" font-size="11" font-weight="700" fill="#9F1239">&gt; tải người dùng tạo ra</text>
<rect x="24" y="302" width="420" height="166" rx="10" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="44" y="326" font-size="12" font-weight="800" fill="#9F1239">Vì sao retry làm mọi thứ tệ hơn</text>
<text x="44" y="350" font-size="11" fill="#BE123C">• Request bị timeout vẫn ĐANG chạy ở server —</text>
<text x="44" y="368" font-size="11" fill="#BE123C">  client bỏ đi nhưng tài nguyên vẫn bị chiếm</text>
<text x="44" y="390" font-size="11" fill="#BE123C">• Retry đến khi server ĐANG yếu nhất</text>
<text x="44" y="412" font-size="11" fill="#BE123C">• 3 lần retry = tải ×4, không phải ×1</text>
<text x="44" y="436" font-size="11" font-weight="800" fill="#DC2626">• Hệ thống không tự hồi phục được —</text>
<text x="44" y="454" font-size="11" font-weight="800" fill="#DC2626">  phải NGẮT từ bên ngoài mới dừng vòng lặp</text>
<rect x="458" y="302" width="418" height="166" rx="10" fill="#CCFBF1" stroke="#0D9488" stroke-width="2.5"/>
<text x="478" y="326" font-size="12" font-weight="800" fill="#134E4A">Bốn cách phá vòng lặp</text>
<text x="478" y="350" font-size="11" fill="#0F766E"><tspan font-weight="700">1. Retry budget</tspan> — chỉ cho retry tối đa 10% tổng</text>
<text x="478" y="368" font-size="11" fill="#0F766E">   request, hết budget thì fail thẳng</text>
<text x="478" y="390" font-size="11" fill="#0F766E"><tspan font-weight="700">2. Circuit breaker</tspan> — mở mạch là DỪNG retry,</text>
<text x="478" y="408" font-size="11" fill="#0F766E">   không phải retry nhanh hơn</text>
<text x="478" y="430" font-size="11" fill="#0F766E"><tspan font-weight="700">3. Load shedding</tspan> — chủ động trả 503 khi p99 vỡ</text>
<text x="478" y="452" font-size="11" fill="#0F766E"><tspan font-weight="700">4. Retry-After có jitter</tspan> do SERVER chỉ định</text>
</svg>
</div>

## Load shedding: chủ động từ chối để sống sót

Nghịch lý: **cách duy nhất để phục vụ được nhiều người nhất là chủ động từ chối một số người.** Hệ thống nhận 100% rồi chết phục vụ được 0 người; hệ thống nhận 20% và sống phục vụ được 20%.

```go
// internal/presentation/middleware/shed.go

// Shedder theo dõi p99 thật và từ chối khi vượt ngưỡng.
// Khác rate limit: rate limit dựa trên SỐ LƯỢNG, shedder dựa trên SỨC KHOẺ.
type Shedder struct {
    p99       *atomic.Int64       // nanosecond, cập nhật bởi goroutine lấy mẫu
    threshold time.Duration
    inflight  atomic.Int64
    maxInflight int64
    logger    *zap.Logger
}

func (s *Shedder) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Lớp 1: giới hạn số request đang xử lý (Little's Law:
        // inflight = throughput × latency, nên chặn inflight là chặn cả hai)
        cur := s.inflight.Add(1)
        defer s.inflight.Add(-1)
        if cur > s.maxInflight {
            s.reject(c, "max_inflight")
            return
        }

        // Lớp 2: nếu p99 đã vỡ ngưỡng, từ chối một phần theo xác suất.
        // Từ chối NGẪU NHIÊN quan trọng: từ chối theo thứ tự sẽ
        // luôn loại cùng một nhóm người dùng.
        if p99 := time.Duration(s.p99.Load()); p99 > s.threshold {
            over := float64(p99-s.threshold) / float64(s.threshold)
            dropRate := math.Min(0.9, over)   // tối đa shed 90%, luôn giữ 10% để đo
            if rand.Float64() < dropRate {
                s.reject(c, "p99_breach")
                return
            }
        }
        c.Next()
    }
}

func (s *Shedder) reject(c *gin.Context, reason string) {
    metrics.Shed.WithLabelValues(reason).Inc()
    // Retry-After có JITTER do server sinh — không để client tự quyết.
    c.Header("Retry-After", "2")
    c.AbortWithStatusJSON(503, gin.H{
        "success": false,
        "error": gin.H{
            "code":    "SERVICE_OVERLOADED",
            "message": "hệ thống đang quá tải, vui lòng thử lại",
        },
        "meta": gin.H{"retry_after_ms": 1000 + rand.Intn(4000)},
    })
}
```

Ba điểm thiết kế:

1. **Giới hạn `inflight` là lớp bảo vệ mạnh nhất và đơn giản nhất.** Theo định luật Little (`inflight = throughput × latency`), chặn inflight tự động chặn cả throughput lẫn latency — không cần đo gì.
2. **Shed theo xác suất, không theo thứ tự.** Shed tuần tự sẽ luôn loại đúng những client kết nối chậm nhất — tức là phân biệt đối xử theo chất lượng mạng.
3. **Luôn giữ lại ≥10%.** Nếu shed 100%, bạn mất luôn tín hiệu để biết hệ thống đã hồi phục chưa.

## Retry budget — thứ hay bị thiếu nhất

Circuit breaker bảo vệ *một* dependency. Retry budget bảo vệ **toàn hệ thống** khỏi chính các retry của nó:

```go
// Retry budget: tổng số retry không được vượt X% tổng số request.
// Google SRE gọi đây là "retry amplification guard".
type RetryBudget struct {
    mu       sync.Mutex
    requests int64
    retries  int64
    ratio    float64       // 0.1 = retry tối đa 10% tổng request
    window   time.Duration
    lastReset time.Time
}

func (b *RetryBudget) AllowRetry() bool {
    b.mu.Lock()
    defer b.mu.Unlock()
    if time.Since(b.lastReset) > b.window {
        b.requests, b.retries, b.lastReset = 0, 0, time.Now()
    }
    if float64(b.retries) >= float64(b.requests)*b.ratio {
        return false   // hết budget → FAIL NGAY, không retry
    }
    b.retries++
    return true
}
```

Không có retry budget, một sự cố nhỏ ở dependency sẽ khiến **mọi** request retry 3 lần, biến tải 1× thành 4× — đúng lúc dependency đang yếu nhất.

---

# Stress test và monitoring

## Cách tìm breaking point

Load test cho hệ thống burst **khác** load test tải đều. Trình tự:

1. **Tìm công suất bền (sustained capacity) trước.** Tăng RPS từng bậc, mỗi bậc giữ 5 phút, tới khi p99 vỡ ngưỡng. Đây là baseline — con số bạn sẽ dùng để cấu hình rate limit.
2. **Rồi mới test burst.** Từ tải nền, nhảy thẳng lên 50× trong 10 giây. Đo: p99 trong burst, tỉ lệ lỗi, **thời gian hồi phục sau burst**. Thời gian hồi phục quan trọng hơn p99 trong burst — nó cho biết hệ thống có tự hồi phục được hay không.
3. **Test oversell.** Đây là bước **không thể thiếu** với flash sale, và là bước hay bị bỏ qua nhất. Đặt tồn kho = 100, bắn 10.000 request đồng thời, rồi **đếm số đơn tạo ra**. Khác 100 là có bug. Như số đo trên, cách viết "nhanh" oversell 6%.
4. **Test retry storm.** Cấu hình client retry 3 lần, rồi làm chậm dependency một cách nhân tạo. Nếu không có retry budget, bạn sẽ thấy tải tăng 4×.
5. **Test idempotency.** Gửi cùng một `reserve_token` 10 lần song song. Phải ra **đúng một** đơn.

> ⚠️ **Lưu ý về công cụ**: khi tạo tải từ một máy, bạn sẽ chạm trần **ephemeral port (~28.000 connection mỗi source IP)** và **`max_connections` của MySQL** trước khi chạm trần ứng dụng. Tôi gặp đúng lỗi thứ hai: pool 200 connection vượt `max_connections=151` mặc định, và benchmark báo "thành công 344/20000" trong khi thực tế là lỗi `too many connections` — không phải lỗi contention. **Luôn phân loại lỗi trong load test**, đừng chỉ đếm thành công/thất bại.

## Metric bắt buộc

| Nhóm | Metric | Vì sao |
|---|---|---|
| **Phễu lọc** | Số request **còn lại sau từng tầng** | Thấy ngay tầng nào không chặn được |
| | `shed_total` theo `reason` | Load shedding có đang hoạt động không |
| **Hot path** | `reserve_duration` p99 | Điểm tuần tự hoá duy nhất — phải theo dõi sát |
| | **`oversell_count`** | **Phải luôn bằng 0.** Alert ngay nếu > 0 |
| | `reserve_rejected` vs `reserve_granted` | Tỉ lệ đúng với tồn kho còn lại không |
| **Retry** | `retry_ratio` = retries / requests | **> 10% = đang có retry storm** |
| | `retry_budget_exhausted` | Budget đã kịch chưa |
| **Saturation** | `inflight_requests` | Tín hiệu quá tải sớm nhất |
| | `db_pool_wait_duration` | Pool cạn trước khi DB chậm |
| | Kafka consumer lag | Checkout async có theo kịp không |
| **Nghiệp vụ** | `orders_created` vs `tokens_minted` | Phải khớp. Lệch = mất đơn hoặc trùng đơn |

**Metric quan trọng nhất: `oversell_count`.** Nó không phải metric hiệu năng mà là **metric đúng đắn** — và nó phải luôn bằng 0. Cách tính đơn giản: `COUNT(orders) - tokens_minted`, chạy mỗi phút. Nếu dương, bạn đang bán hàng không có.

Metric cảnh báo sớm nhất: **`inflight_requests`**. Nó tăng trước cả latency, vì latency chỉ tăng *sau khi* queue đã dài.

## Checklist

```txt
TÍNH ĐÚNG ĐẮN (quan trọng hơn hiệu năng)
□ Reserve có atomic trong MỘT lệnh không? (Lua hoặc LPOP, KHÔNG phải GET rồi DECR)
□ Đã test oversell với 10.000 request đồng thời chưa? Kết quả có khớp tồn kho?
□ Idempotency key dùng token nghiệp vụ hay message ID? (phải là nghiệp vụ)
□ Có unique constraint trong DB làm lớp bảo đảm cuối chưa?
□ Khi checkout thất bại, khoá idempotency có được NHẢ ra không?
□ Rate limiter dùng server time hay client time? (client time = vượt 40%)
□ Dùng fixed window? Đã biết nó cho qua 2× ở ranh giới chưa?

CHỐNG QUÁ TẢI
□ Có giới hạn inflight request không? (lớp bảo vệ mạnh nhất)
□ Load shedding theo p99 có chưa? Shed NGẪU NHIÊN không?
□ Có giữ lại ≥10% để đo sức khoẻ không?
□ Có retry budget không? (không có = tải ×4 khi sự cố)
□ Retry-After do SERVER sinh và có jitter chưa?
□ Mọi lỗi tra Redis/DB đều FAIL CLOSED chưa?

GIẢM BURST TỪ GỐC
□ Biết trước giờ mở bán? → phát vé TRƯỚC, rải trên cửa sổ 60s
□ CDN đã chặn được bao nhiêu % traffic? Đã đo chưa?
□ Trang landing + countdown có 100% cache được không?

LOAD TEST
□ Đã tìm sustained capacity trước khi test burst chưa?
□ Đã đo THỜI GIAN HỒI PHỤC sau burst chưa?
□ Load test có PHÂN LOẠI lỗi không? (too many connections ≠ contention)
□ Đã test retry storm với dependency chậm giả tạo chưa?
```

---

# Tóm tắt

| Câu hỏi | Trả lời |
|---|---|
| Nguyên tắc kiến trúc số 1? | **Từ chối sớm và rẻ.** 99,8% request chắc chắn thất bại — đừng để chúng đi sâu |
| Cách giảm burst tốt nhất? | **Phát vé trước giờ mở bán.** Xoá bỏ burst thay vì chịu đựng nó |
| Cách giữ tồn kho nào sai? | `GET` rồi `DECR` — đo thật: **oversell 60/1.000 suất (6%)** |
| Cách nào tốt nhất? | **`LPOP` từ túi token đúc sẵn** (95.863 ops/s) hoặc **Lua atomic** (83.313 ops/s) |
| Tính đúng đắn có đắt không? | **Gần như miễn phí**: Lua atomic 83.313 vs cách sai 83.852 ops/s |
| Sharding bộ đếm có giúp? | **Không, với tồn kho.** Đo thật chậm nhất (5.452 ops/s) vì 19.000 người thăm dò 16 shard rồi mới biết hết hàng |
| Lỗi rate limit hay gặp nhất? | Dùng **client time** → cho qua **vượt 40%**. Phải dùng `redis.call('TIME')` |
| Fixed window có an toàn? | **Không.** Đo thật: cho qua **2.000/1.000 ở ranh giới cửa sổ** |
| Failure mode tệ nhất? | **Retry storm** — vòng lặp tự khuếch đại, hệ thống không tự hồi phục được |
| Lớp bảo vệ mạnh nhất? | **Giới hạn inflight request** (định luật Little) |
| Metric phải luôn bằng 0? | **`oversell_count`** |

---

# Tham khảo

- [Google SRE — Handling Overload](https://sre.google/sre-book/handling-overload/) · [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — nguồn của retry budget và load shedding
- [Redis — Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/) · [`TIME`](https://redis.io/docs/latest/commands/time/)
- [Stripe — Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters) — 4 loại limiter trong production thật
- [Marc Brooker — Little's Law and capacity](https://brooker.co.za/blog/2021/04/19/latency.html)
- Group Facebook [Viet Tran Premium Lab & Playground](https://www.facebook.com/groups/2512930739174619) — nguồn nội dung gốc
- Series: [P01 — CCU không phải RPS](/posts/scalable-p01-ccu-vs-rps/) · [P01.2 — 1M CCU gaming](/posts/scalable-p01-2-ccu-deep-dive/) · [P01.4 — Chat 1M CCU](/posts/scalable-p01-4-chat-deep-dive/) · [P01.5 — Public API triệu RPS](/posts/scalable-p01-5-api-deep-dive/) · [P02 — Cache Stampede](/posts/system-design-p02-cache-stampede/)
