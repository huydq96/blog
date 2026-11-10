+++
date = '2026-09-04T10:00:00+07:00'
draft = false
title = 'Go Interview P2'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "interview"]
description = 'Một số câu hỏi phỏng vấn về Golang P2 — các loại bộ nhớ, Garbage Collector, memory leak, error wrapping'
+++

# Các loại bộ nhớ trong một chương trình Go là gì?

## Tổng quan các vùng nhớ

| Vùng nhớ | Chứa gì | Ai quản lý | Vòng đời |
|---|---|---|---|
| **Stack** | Local variable, tham số hàm, địa chỉ return — mỗi goroutine có 1 stack riêng | Go runtime, tự động | Theo từng lời gọi hàm — cấp khi gọi, thu khi return |
| **Heap** | Object "escape" khỏi function scope | Garbage Collector | Không xác định trước — sống tới khi unreachable |
| **Data / BSS segment** | Biến global đã khởi tạo (data) và chưa khởi tạo/zero value (BSS) | OS loader lúc khởi động | Suốt vòng đời chương trình |
| **Text / Code segment** | Machine code đã compile, read-only | OS | Suốt vòng đời chương trình, không đổi |

## Stack

- Mỗi goroutine có **1 stack riêng**, khác hẳn OS thread: kích thước ban đầu chỉ ~2KB (so với 1-8MB cố định của OS thread stack) — đây là lý do Go tạo hàng trăm nghìn goroutine vẫn nhẹ.
- Stack **grow động**: khi gần đầy, runtime cấp 1 vùng lớn hơn, copy toàn bộ nội dung sang và cập nhật lại mọi pointer trỏ vào stack cũ ("stack copying"). Mặc định tối đa 1GB (`debug.SetMaxStack`).
- Cấp phát/giải phóng cực rẻ — chỉ là di chuyển stack pointer, GC không cần quan tâm tới object nằm ở đây.

## Heap

- Bộ cấp phát heap của Go lấy cảm hứng từ **TCMalloc** (Thread-Caching Malloc): object được nhóm theo **size class** cố định để giảm phân mảnh. Mỗi P (processor) có cache riêng (`mcache`) cấp phát nhanh không cần lock; hết cache mới xin thêm từ `mcentral`/`mheap` (dùng chung, cần lock).
- Object lớn (> 32KB) cấp phát thẳng từ `mheap`, không qua size class.
- Đây là vùng duy nhất GC phải quét và thu hồi.

## Global / package-level (Data & BSS)

- `var x = 5` (có giá trị khởi tạo) nằm ở data segment; `var x int` (chưa gán, zero value) thuộc BSS về mặt khái niệm — Go không tách bạch rạch ròi như C, trình biên dịch/linker tự quyết định chỗ đặt.
- Cấp phát **một lần** khi chương trình khởi động (trước cả `main()`, qua `init()`), tồn tại suốt đời chương trình — đây cũng là lý do biến global luôn là GC root.

> ★ Hiểu đúng "biến này nằm ở đâu" là nền cho gần như mọi câu hỏi tối ưu hiệu năng phía sau: escape analysis quyết định stack hay heap, heap là thứ GC phải lo, global luôn là root.

# Garbage Collector (GC) trong Go là gì, hoạt động thế nào, và cần lưu ý gì để tối ưu?

## GC là gì?

Garbage Collector là cơ chế **tự động quản lý bộ nhớ** — tự tìm và giải phóng những vùng nhớ đã cấp phát (heap) mà chương trình không còn tham chiếu tới (unreachable) nữa, để lập trình viên không phải gọi `free()` thủ công như C/C++.

Go dùng thuật toán **concurrent, tri-color, mark-and-sweep, non-generational, non-compacting**:
- **Concurrent**: GC chạy song song với chương trình (mutator), không dừng hẳn toàn bộ ứng dụng để dọn rác.
- **Tri-color mark-and-sweep**: đánh dấu (mark) object còn sống bằng 3 màu, rồi quét (sweep) thu hồi phần không được đánh dấu.
- **Non-generational**: không chia heap theo "thế hệ" (young/old) như JVM — mọi object được quét như nhau mỗi chu kỳ.
- **Non-compacting**: không di chuyển object để dồn bộ nhớ liền khối — vùng nhớ trống bị phân mảnh, quản lý qua free list.

## Cách hoạt động — Tri-color mark-and-sweep

1. **Root**: điểm bắt đầu quét — biến global, biến trên stack của từng goroutine đang chạy. Mọi root được tô **grey**.
2. **Mark**: lấy 1 object grey, quét mọi con trỏ nó tham chiếu tới → tô các object đó thành grey, chính object vừa quét chuyển sang **black** (đã xử lý xong). Lặp lại tới khi hết object grey.
3. Object nào vẫn còn **white** sau khi hết grey = không có đường nào từ root tới được nó = rác.
4. **Sweep**: thu hồi toàn bộ vùng nhớ của các object white.

```
white (chưa xét) → grey (đã thấy, chưa quét con) → black (đã quét xong, chắc chắn sống)
Sau khi hết grey: white còn lại = rác → sweep thu hồi
```

### Từng bước trên một đồ thị object cụ thể

Dưới đây là đúng đồ thị mà đoạn code ở phần sau tạo ra: `globalCache → A → B`, `C` được giữ bởi biến local của `main`, `E` được giữ bởi một goroutine đang block, và `D` không có ai tham chiếu.

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 600" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="gc-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#78716C"/></marker>
<marker id="gc-arg" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#0D9488"/></marker>
<filter id="gc-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="600" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Tri-color mark-and-sweep — chạy trên đồ thị object thật</text>
<circle cx="316" cy="44" r="9" fill="#FFFFFF" stroke="#A8A29E" stroke-width="2"/>
<text x="332" y="48" font-size="11" fill="#57534E">white: chưa thấy</text>
<circle cx="452" cy="44" r="9" fill="#D6D3D1" stroke="#78716C" stroke-width="2"/>
<text x="468" y="48" font-size="11" fill="#57534E">grey: đã thấy, chưa quét con</text>
<circle cx="668" cy="44" r="9" fill="#374151"/>
<text x="684" y="48" font-size="11" fill="#57534E">black: quét xong, chắc chắn sống</text>
<rect x="20" y="62" width="420" height="218" rx="11" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2" filter="url(#gc-sh)"/>
<text x="230" y="84" text-anchor="middle" font-size="12" font-weight="800" fill="#374151">① Quét ROOT — mọi object khác là white</text>
<rect x="32" y="96" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="32" y="138" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="32" y="180" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<text x="78" y="112" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT global</text>
<text x="78" y="154" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack main</text>
<text x="78" y="196" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack goroutine</text>
<line x1="128" y1="108" x2="166" y2="108" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="128" y1="150" x2="166" y2="150" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="128" y1="192" x2="166" y2="192" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="208" y1="108" x2="264" y2="108" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<circle cx="188" cy="108" r="19" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="286" cy="108" r="19" fill="#FFFFFF" stroke="#A8A29E" stroke-width="2"/>
<circle cx="188" cy="150" r="19" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="188" cy="192" r="19" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="368" cy="170" r="19" fill="#FFFFFF" stroke="#A8A29E" stroke-width="2"/>
<text x="188" y="113" text-anchor="middle" font-size="13" font-weight="800" fill="#1C1917">A</text>
<text x="286" y="113" text-anchor="middle" font-size="13" font-weight="800" fill="#57534E">B</text>
<text x="188" y="155" text-anchor="middle" font-size="13" font-weight="800" fill="#1C1917">C</text>
<text x="188" y="197" text-anchor="middle" font-size="13" font-weight="800" fill="#1C1917">E</text>
<text x="368" y="175" text-anchor="middle" font-size="13" font-weight="800" fill="#57534E">D</text>
<text x="368" y="204" text-anchor="middle" font-size="9" fill="#A8A29E">không ai trỏ tới</text>
<text x="230" y="250" text-anchor="middle" font-size="10.5" fill="#57534E">Root trỏ thẳng tới A, C, E → ba object này thành grey.</text>
<text x="230" y="266" text-anchor="middle" font-size="10.5" fill="#57534E">B và D vẫn white vì GC chưa &quot;nhìn thấy&quot; chúng.</text>
<rect x="460" y="62" width="420" height="218" rx="11" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2" filter="url(#gc-sh)"/>
<text x="670" y="84" text-anchor="middle" font-size="12" font-weight="800" fill="#374151">② MARK — lấy từng grey, quét con của nó</text>
<rect x="472" y="96" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="472" y="138" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="472" y="180" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<text x="518" y="112" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT global</text>
<text x="518" y="154" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack main</text>
<text x="518" y="196" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack goroutine</text>
<line x1="568" y1="108" x2="606" y2="108" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="568" y1="150" x2="606" y2="150" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="568" y1="192" x2="606" y2="192" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="648" y1="108" x2="704" y2="108" stroke="#0D9488" stroke-width="2.5" marker-end="url(#gc-arg)"/>
<circle cx="628" cy="108" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="726" cy="108" r="19" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="628" cy="150" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="628" cy="192" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="808" cy="170" r="19" fill="#FFFFFF" stroke="#A8A29E" stroke-width="2"/>
<text x="628" y="113" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">A</text>
<text x="726" y="113" text-anchor="middle" font-size="13" font-weight="800" fill="#1C1917">B</text>
<text x="628" y="155" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">C</text>
<text x="628" y="197" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">E</text>
<text x="808" y="175" text-anchor="middle" font-size="13" font-weight="800" fill="#57534E">D</text>
<text x="686" y="142" text-anchor="middle" font-size="9" font-weight="700" fill="#0D9488">đi theo con trỏ</text>
<text x="670" y="250" text-anchor="middle" font-size="10.5" fill="#57534E">Quét A → thấy B, tô B grey, A chuyển black. C và E không có</text>
<text x="670" y="266" text-anchor="middle" font-size="10.5" fill="#57534E">con trỏ ra ngoài nên quét xong là black luôn. D không ai chạm tới.</text>
<rect x="20" y="296" width="420" height="218" rx="11" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2" filter="url(#gc-sh)"/>
<text x="230" y="318" text-anchor="middle" font-size="12" font-weight="800" fill="#374151">③ Hết grey — white còn lại chính là rác</text>
<rect x="32" y="330" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="32" y="372" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="32" y="414" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<text x="78" y="346" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT global</text>
<text x="78" y="388" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack main</text>
<text x="78" y="430" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack goroutine</text>
<line x1="128" y1="342" x2="166" y2="342" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="128" y1="384" x2="166" y2="384" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="128" y1="426" x2="166" y2="426" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="208" y1="342" x2="264" y2="342" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<circle cx="188" cy="342" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="286" cy="342" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="188" cy="384" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="188" cy="426" r="19" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="368" cy="404" r="19" fill="#FFFFFF" stroke="#DC2626" stroke-width="2.5" stroke-dasharray="4 3"/>
<text x="188" y="347" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">A</text>
<text x="286" y="347" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">B</text>
<text x="188" y="389" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">C</text>
<text x="188" y="431" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">E</text>
<text x="368" y="409" text-anchor="middle" font-size="13" font-weight="800" fill="#DC2626">D</text>
<text x="368" y="438" text-anchor="middle" font-size="9" font-weight="700" fill="#DC2626">RÁC</text>
<text x="230" y="484" text-anchor="middle" font-size="10.5" fill="#57534E">Không còn object grey nào → mark kết thúc. A, B, C, E đều black.</text>
<text x="230" y="500" text-anchor="middle" font-size="10.5" fill="#57534E">D vẫn white: không có đường đi nào từ root tới nó.</text>
<rect x="460" y="296" width="420" height="218" rx="11" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#gc-sh)"/>
<text x="670" y="318" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">④ SWEEP — thu hồi vùng nhớ của object white</text>
<rect x="472" y="330" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="472" y="372" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<rect x="472" y="414" width="92" height="24" rx="5" fill="#EEF2FF" stroke="#6366F1" stroke-width="1.6"/>
<text x="518" y="346" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT global</text>
<text x="518" y="388" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack main</text>
<text x="518" y="430" text-anchor="middle" font-size="8" font-weight="700" fill="#3730A3">ROOT stack goroutine</text>
<line x1="568" y1="342" x2="606" y2="342" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="568" y1="384" x2="606" y2="384" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="568" y1="426" x2="606" y2="426" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<line x1="648" y1="342" x2="704" y2="342" stroke="#78716C" stroke-width="1.8" marker-end="url(#gc-ar)"/>
<circle cx="628" cy="342" r="19" fill="#0D9488" stroke="#0F766E" stroke-width="2.5"/>
<circle cx="726" cy="342" r="19" fill="#0D9488" stroke="#0F766E" stroke-width="2.5"/>
<circle cx="628" cy="384" r="19" fill="#0D9488" stroke="#0F766E" stroke-width="2.5"/>
<circle cx="628" cy="426" r="19" fill="#0D9488" stroke="#0F766E" stroke-width="2.5"/>
<text x="628" y="347" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">A</text>
<text x="726" y="347" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">B</text>
<text x="628" y="389" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">C</text>
<text x="628" y="431" text-anchor="middle" font-size="13" font-weight="800" fill="#FFFFFF">E</text>
<text x="808" y="409" text-anchor="middle" font-size="22" font-weight="800" fill="#DC2626">✕</text>
<text x="808" y="434" text-anchor="middle" font-size="10" font-weight="700" fill="#DC2626">D đã thu hồi</text>
<text x="670" y="484" text-anchor="middle" font-size="10.5" fill="#0F766E">1MB của D trả về free list. Heap giảm đúng 1MB — xem số đo ở dưới.</text>
<text x="670" y="500" text-anchor="middle" font-size="10.5" fill="#0F766E">Chu kỳ sau, A B C E được đặt lại thành white và quét lại từ đầu.</text>
<rect x="20" y="528" width="860" height="58" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="40" y="552" font-size="12" font-weight="800" fill="#7C4A03">Điểm mấu chốt: GC không hỏi &quot;object này còn được dùng không&quot;</text>
<text x="40" y="572" font-size="11" fill="#92400E">Nó chỉ hỏi <tspan font-weight="700">&quot;có đường đi nào từ root tới object này không&quot;</tspan>. D chết không phải vì không ai dùng, mà vì không ai <tspan font-style="italic">trỏ tới</tspan>.</text>
</svg>
</div>

### Ví dụ bằng code — chạy thật và đo từng bước

Chương trình dưới đây dựng đúng đồ thị trên, rồi lần lượt cắt từng cạnh và gọi `runtime.GC()` để thấy heap tụt đúng 1MB mỗi lần:

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
)

// ╔═══════════════════════════════════════════════════════════════╗
// ║ DATA / BSS SEGMENT — cấp phát 1 lần lúc khởi động,            ║
// ║ sống suốt đời chương trình => LUÔN là GC root.                ║
// ╚═══════════════════════════════════════════════════════════════╝
var globalCache *Node // BSS (chưa khởi tạo → zero value nil)

// Node sẽ nằm trên HEAP. Mỗi Node nặng đúng 1MB để nhìn rõ số liệu.
type Node struct {
	name string
	next *Node        // con trỏ → GC phải đi theo cạnh này khi mark
	data [1 << 20]byte // payload 1MB
}

func newNode(name string) *Node {
	// n là biến LOCAL, nhưng ta trả về con trỏ → escape analysis đẩy lên HEAP.
	// Kiểm chứng: go build -gcflags="-m"  →  "&Node{...} escapes to heap"
	n := &Node{name: name}
	return n
}

// ╔═══════════════════════════════════════════════════════════════╗
// ║ STACK — x không escape, nằm trên stack của goroutine gọi hàm. ║
// ║ GC KHÔNG tốn một chút chi phí nào cho x.                      ║
// ╚═══════════════════════════════════════════════════════════════╝
func sumOnStack(a, b int) int {
	x := a + b // stack: thu hồi tự động khi hàm return
	return x   // chỉ copy giá trị ra ngoài, không con trỏ nào thoát
}

func heapMB() float64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return float64(m.HeapAlloc) / (1 << 20)
}

func main() {
	// Tắt GC tự động: mọi lần GC dưới đây đều do ta CHỦ ĐỘNG gọi,
	// để số liệu phản ánh đúng từng bước, không lẫn chu kỳ bất chợt.
	debug.SetGCPercent(-1)
	fmt.Printf("heap ban đầu                            : %.1f MB\n\n", heapMB())

	// ─── Dựng đồ thị object trên heap ───
	// globalCache ──► A ──► B      (reachable từ root GLOBAL)
	a := newNode("A")
	a.next = newNode("B")
	globalCache = a

	// C: chỉ được giữ bởi biến local của main → root STACK của goroutine main
	c := newNode("C")

	// D: tạo xong bỏ luôn, không ai tham chiếu → WHITE → bị sweep
	_ = newNode("D")

	// E: bị giữ bởi một goroutine block vĩnh viễn.
	// MỖI GOROUTINE ĐANG SỐNG LÀ 1 GC ROOT → E không bao giờ được thu hồi,
	// dù logic chương trình không còn cần nó. Đây chính là goroutine leak.
	ready := make(chan struct{})
	go func() {
		e := newNode("E")
		close(ready)          // báo đã cấp phát xong rồi mới block
		<-make(chan struct{}) // block mãi mãi → goroutine sống mãi → e sống mãi
		runtime.KeepAlive(e)
	}()
	<-ready

	fmt.Printf("tạo xong A,B,C,D,E (5 × 1MB)            : %.1f MB\n", heapMB())

	// ─── GC #1 ───
	// Mark: A,B (từ global) · C (từ stack main) · E (từ stack goroutine) → BLACK
	// D không có đường nào từ root → WHITE → sweep thu hồi
	runtime.GC()
	fmt.Printf("GC #1 — D là rác, bị thu hồi            : %.1f MB   (A,B,C,E sống)\n", heapMB())
	fmt.Printf("        C vẫn sống vì main còn dùng: %q\n", c.name) // chính dòng này giữ C reachable

	// ─── GC #2: buông tham chiếu tới C ───
	c = nil
	runtime.GC()
	fmt.Printf("GC #2 — c = nil, C thành rác            : %.1f MB   (A,B,E sống)\n", heapMB())

	// ─── GC #3: cắt cạnh A ──► B ───
	// B mất đường đi duy nhất từ root. A vẫn sống vì global còn giữ.
	globalCache.next = nil
	runtime.GC()
	fmt.Printf("GC #3 — A.next = nil, B thành rác       : %.1f MB   (A,E sống)\n", heapMB())

	// ─── GC #4: buông cả biến global ───
	globalCache = nil
	runtime.GC()
	fmt.Printf("GC #4 — globalCache = nil, A thành rác  : %.1f MB   (chỉ còn E)\n", heapMB())

	// E vẫn nằm đó. GC làm đúng việc của nó — E thật sự còn reachable.
	fmt.Printf("\ngoroutine đang sống: %d → E (1MB) vẫn reachable từ stack của nó.\n",
		runtime.NumGoroutine())
	fmt.Println("GC không thu hồi thứ \"không dùng nữa\", nó thu hồi thứ KHÔNG CÒN REACHABLE.")
	fmt.Printf("\nstack không tốn chi phí GC: sumOnStack(2,3) = %d\n", sumOnStack(2, 3))
}
```

Kết quả chạy thật (Go 1.24.4, linux/amd64):

```
heap ban đầu                            : 0.2 MB

tạo xong A,B,C,D,E (5 × 1MB)            : 5.2 MB
GC #1 — D là rác, bị thu hồi            : 4.2 MB   (A,B,C,E sống)
        C vẫn sống vì main còn dùng: "C"
GC #2 — c = nil, C thành rác            : 3.2 MB   (A,B,E sống)
GC #3 — A.next = nil, B thành rác       : 2.2 MB   (A,E sống)
GC #4 — globalCache = nil, A thành rác  : 1.2 MB   (chỉ còn E)

goroutine đang sống: 2 → E (1MB) vẫn reachable từ stack của nó.
GC không thu hồi thứ "không dùng nữa", nó thu hồi thứ KHÔNG CÒN REACHABLE.

stack không tốn chi phí GC: sumOnStack(2,3) = 5
```

| Thao tác | Object trở thành rác | Vì sao | Heap |
|---|---|---|---:|
| — | — | vừa cấp phát 5 object | **5.2 MB** |
| `runtime.GC()` | **D** | không một root nào trỏ tới | **4.2 MB** |
| `c = nil` | **C** | mất root `stack main` — đường duy nhất | **3.2 MB** |
| `globalCache.next = nil` | **B** | cạnh `A → B` bị cắt, hết đường từ root | **2.2 MB** |
| `globalCache = nil` | **A** | mất root `global` | **1.2 MB** |
| *(kết thúc)* | **không bao giờ là E** | goroutine đang block vẫn là root | **1.2 MB** |

Ba điều đọc được từ bảng này:

- **Heap giảm đúng 1MB mỗi bước** — tương ứng chính xác một object bị cắt khỏi đồ thị. GC không "đoán", nó đi theo con trỏ.
- **Cắt cạnh giữa hai object cũng đủ giết object con** (`A.next = nil` làm B chết), không nhất thiết phải buông biến gốc.
- **1MB cuối không bao giờ về 0.2MB.** Goroutine kia chỉ đang `block`, nhưng với GC nó là root hợp lệ — đây chính là goroutine leak, và GC hoàn toàn "làm đúng".

### Bốn pha của một chu kỳ GC và hai điểm Stop-The-World

<div class="zoomable-diagram" style="max-width:900px;margin:2rem auto;">
<svg viewBox="0 0 900 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="ph-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#57534E"/></marker>
<filter id="ph-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="900" height="400" fill="#FFFDF7" rx="16"/>
<text x="450" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Một chu kỳ GC — chương trình chỉ bị dừng ở hai vạch đỏ hẹp</text>
<text x="450" y="46" text-anchor="middle" font-size="11" fill="#78716C">Số liệu lấy từ GODEBUG=gctrace=1 của chính chương trình ở trên</text>
<text x="24" y="92" font-size="11" font-weight="700" fill="#57534E">GC</text>
<rect x="60" y="74" width="26" height="34" rx="4" fill="#DC2626"/>
<rect x="90" y="74" width="330" height="34" rx="4" fill="#0D9488"/>
<rect x="424" y="74" width="20" height="34" rx="4" fill="#DC2626"/>
<rect x="448" y="74" width="412" height="34" rx="4" fill="#14B8A6"/>
<text x="73" y="96" text-anchor="middle" font-size="9" font-weight="800" fill="#FFFFFF">STW</text>
<text x="255" y="96" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">CONCURRENT MARK</text>
<text x="434" y="96" text-anchor="middle" font-size="9" font-weight="800" fill="#FFFFFF">STW</text>
<text x="654" y="96" text-anchor="middle" font-size="12" font-weight="800" fill="#FFFFFF">CONCURRENT SWEEP</text>
<text x="24" y="152" font-size="11" font-weight="700" fill="#57534E">App</text>
<rect x="60" y="134" width="26" height="34" rx="4" fill="#E7E5E4" stroke="#DC2626" stroke-width="1.5" stroke-dasharray="3 2"/>
<rect x="90" y="134" width="330" height="34" rx="4" fill="#374151"/>
<rect x="424" y="134" width="20" height="34" rx="4" fill="#E7E5E4" stroke="#DC2626" stroke-width="1.5" stroke-dasharray="3 2"/>
<rect x="448" y="134" width="412" height="34" rx="4" fill="#374151"/>
<text x="73" y="156" text-anchor="middle" font-size="9" font-weight="800" fill="#DC2626">■</text>
<text x="255" y="156" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">mutator chạy bình thường</text>
<text x="434" y="156" text-anchor="middle" font-size="9" font-weight="800" fill="#DC2626">■</text>
<text x="654" y="156" text-anchor="middle" font-size="11" font-weight="700" fill="#FFFFFF">mutator chạy bình thường</text>
<rect x="60" y="184" width="384" height="22" rx="5" fill="#FFF3DB" stroke="#F5A623" stroke-width="1.8"/>
<text x="252" y="199" text-anchor="middle" font-size="10.5" font-weight="800" fill="#7C4A03">WRITE BARRIER BẬT trong suốt khoảng này</text>
<line x1="73" y1="112" x2="73" y2="130" stroke="#DC2626" stroke-width="1.5"/>
<line x1="434" y1="112" x2="434" y2="130" stroke="#DC2626" stroke-width="1.5"/>
<rect x="24" y="222" width="190" height="62" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="119" y="242" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">① Sweep Termination</text>
<text x="119" y="258" text-anchor="middle" font-size="10" fill="#BE123C">STW · bật write barrier,</text>
<text x="119" y="272" text-anchor="middle" font-size="10" fill="#BE123C">quét root ban đầu</text>
<rect x="226" y="222" width="190" height="62" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="321" y="242" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">② Mark</text>
<text x="321" y="258" text-anchor="middle" font-size="10" fill="#0F766E">chạy SONG SONG với app,</text>
<text x="321" y="272" text-anchor="middle" font-size="10" fill="#0F766E">tô màu theo tri-color</text>
<rect x="428" y="222" width="190" height="62" rx="8" fill="#FFE9EC" stroke="#DC2626" stroke-width="2"/>
<text x="523" y="242" text-anchor="middle" font-size="11" font-weight="800" fill="#9F1239">③ Mark Termination</text>
<text x="523" y="258" text-anchor="middle" font-size="10" fill="#BE123C">STW · chốt kết quả mark,</text>
<text x="523" y="272" text-anchor="middle" font-size="10" fill="#BE123C">tắt write barrier</text>
<rect x="630" y="222" width="190" height="62" rx="8" fill="#CCFBF1" stroke="#0D9488" stroke-width="2"/>
<text x="725" y="242" text-anchor="middle" font-size="11" font-weight="800" fill="#134E4A">④ Sweep</text>
<text x="725" y="258" text-anchor="middle" font-size="10" fill="#0F766E">trả vùng nhớ white về free</text>
<text x="725" y="272" text-anchor="middle" font-size="10" fill="#0F766E">list, cũng chạy song song</text>
<rect x="24" y="298" width="852" height="86" rx="10" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2"/>
<text x="44" y="320" font-size="12" font-weight="800" fill="#374151">Đọc thẳng từ dòng gctrace đo được</text>
<rect x="44" y="330" width="812" height="20" rx="4" fill="#F5F5F4"/>
<text x="52" y="344" font-size="10" font-family="monospace" fill="#1C1917">gc 1 @0.002s 2%: 0.051+0.11+0.003 ms clock, 5-&gt;5-&gt;4 MB, 12 P (forced)</text>
<text x="44" y="366" font-size="10.5" fill="#57534E"><tspan font-weight="700" fill="#DC2626">0.051</tspan> ms STW đầu  +  <tspan font-weight="700" fill="#0D9488">0.11</tspan> ms mark chạy song song  +  <tspan font-weight="700" fill="#DC2626">0.003</tspan> ms STW cuối</text>
<text x="44" y="380" font-size="10.5" fill="#57534E">Tổng thời gian app bị dừng: <tspan font-weight="700">0.054 ms</tspan> — dưới 1/18 của một mili-giây. Và <tspan font-family="monospace">5-&gt;5-&gt;4 MB</tspan> chính là D bị thu hồi.</text>
</svg>
</div>

Bật `gctrace` để tự nhìn số của chương trình mình:

```bash
GODEBUG=gctrace=1 ./myapp
```

```
gc 1 @0.002s 2%: 0.051+0.11+0.003 ms clock, ... 5->5->4 MB, ... 12 P (forced)
gc 2 @0.002s 2%: 0.006+0.072+0.003 ms clock, ... 4->4->3 MB, ... 12 P (forced)
gc 3 @0.003s 3%: 0.031+0.12+0.009 ms clock,  ... 3->3->2 MB, ... 12 P (forced)
gc 4 @0.003s 4%: 0.012+0.086+0.012 ms clock, ... 2->2->1 MB, ... 12 P (forced)
```

Cách đọc một dòng:

| Phần | Ý nghĩa |
|---|---|
| `gc 1` | chu kỳ GC thứ mấy kể từ lúc chương trình chạy |
| `@0.002s` | thời điểm chu kỳ này bắt đầu, tính từ lúc khởi động |
| `2%` | **tổng %** CPU mà GC đã dùng từ đầu chương trình tới giờ |
| `0.051+0.11+0.003 ms` | STW đầu **+** mark concurrent **+** STW cuối — hai số ngoài cùng là phần app thật sự bị dừng |
| `5->5->4 MB` | heap lúc bắt đầu → lúc kết thúc mark → **heap còn sống sau sweep** |
| `12 P` | số processor runtime đang dùng |
| `(forced)` | chu kỳ này do `runtime.GC()` gọi tay, không phải do `GOGC` kích hoạt |

> ★ Số thứ ba trong `5->5->4 MB` là con số đáng quan tâm nhất: đó là **heap thực sự còn sống**, và cũng là mốc để `GOGC` tính chu kỳ tiếp theo. `GOGC=100` nghĩa là GC sẽ chạy lại khi heap chạm `4 × 2 = 8 MB`.


**Write barrier**: vì mark chạy *đồng thời* với chương trình, nếu mutator đổi con trỏ giữa lúc GC đang quét (ví dụ gán 1 object black trỏ tới 1 object white), GC có thể bỏ sót và thu hồi nhầm object đang sống. Go dùng **hybrid write barrier** (từ Go 1.8) để chặn đúng tình huống này mà không cần dừng hẳn chương trình.

Tình huống cụ thể mà write barrier chặn — cần **đồng thời hai thao tác** của mutator thì object sống mới bị bỏ sót:

<div class="zoomable-diagram" style="max-width:880px;margin:2rem auto;">
<svg viewBox="0 0 880 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;height:auto;">
<defs>
<marker id="wb-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#78716C"/></marker>
<marker id="wb-arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#DC2626"/></marker>
<filter id="wb-sh" x="-20%" y="-20%" width="140%" height="140%"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#00000022"/></filter>
</defs>
<rect x="0" y="0" width="880" height="400" fill="#FFFDF7" rx="16"/>
<text x="440" y="26" text-anchor="middle" font-size="16" font-weight="800" fill="#374151">Vì sao cần write barrier — kịch bản object sống bị bỏ sót</text>
<text x="440" y="46" text-anchor="middle" font-size="11" fill="#78716C">Mutator ghi con trỏ ngay giữa lúc GC đang mark</text>
<rect x="16" y="60" width="278" height="212" rx="11" fill="#FFFFFF" stroke="#D6D3D1" stroke-width="2" filter="url(#wb-sh)"/>
<text x="155" y="82" text-anchor="middle" font-size="12" font-weight="800" fill="#374151">① Đang mark dở</text>
<circle cx="76" cy="122" r="21" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="76" cy="200" r="21" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="216" cy="200" r="21" fill="#FFFFFF" stroke="#A8A29E" stroke-width="2"/>
<text x="76" y="127" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">A</text>
<text x="76" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#1C1917">C</text>
<text x="216" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#57534E">B</text>
<text x="76" y="156" text-anchor="middle" font-size="9" font-weight="700" fill="#374151">black: quét xong</text>
<text x="76" y="234" text-anchor="middle" font-size="9" font-weight="700" fill="#78716C">grey: chưa quét con</text>
<text x="216" y="234" text-anchor="middle" font-size="9" font-weight="700" fill="#A8A29E">white</text>
<line x1="99" y1="200" x2="191" y2="200" stroke="#78716C" stroke-width="2" marker-end="url(#wb-ar)"/>
<text x="155" y="192" text-anchor="middle" font-size="9" fill="#57534E">đường sống duy nhất của B</text>
<text x="155" y="258" text-anchor="middle" font-size="10" fill="#57534E">B còn sống nhờ cạnh C → B.</text>
<rect x="302" y="60" width="278" height="212" rx="11" fill="#FFF7F7" stroke="#DC2626" stroke-width="2.5" filter="url(#wb-sh)"/>
<text x="441" y="82" text-anchor="middle" font-size="12" font-weight="800" fill="#9F1239">② KHÔNG có barrier → mất object</text>
<circle cx="362" cy="122" r="21" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="362" cy="200" r="21" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="502" cy="200" r="21" fill="#FFFFFF" stroke="#DC2626" stroke-width="2.5" stroke-dasharray="4 3"/>
<text x="362" y="127" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">A</text>
<text x="362" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#1C1917">C</text>
<text x="502" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#DC2626">B</text>
<path d="M383,133 C430,150 470,172 486,186" stroke="#DC2626" stroke-width="2.5" fill="none" marker-end="url(#wb-arr)"/>
<text x="452" y="146" text-anchor="middle" font-size="9" font-weight="800" fill="#DC2626">A.next = B</text>
<line x1="385" y1="200" x2="477" y2="200" stroke="#D6D3D1" stroke-width="2" stroke-dasharray="4 3"/>
<line x1="412" y1="188" x2="450" y2="212" stroke="#DC2626" stroke-width="2.5"/>
<line x1="450" y1="188" x2="412" y2="212" stroke="#DC2626" stroke-width="2.5"/>
<text x="431" y="232" text-anchor="middle" font-size="9" font-weight="800" fill="#DC2626">C.next = nil</text>
<text x="441" y="258" text-anchor="middle" font-size="10" fill="#BE123C">A đã black nên không quét lại. C quét thì B đã mất cạnh.</text>
<rect x="588" y="60" width="278" height="212" rx="11" fill="#F2FCFB" stroke="#0D9488" stroke-width="2.5" filter="url(#wb-sh)"/>
<text x="727" y="82" text-anchor="middle" font-size="12" font-weight="800" fill="#134E4A">③ CÓ write barrier → B được cứu</text>
<circle cx="648" cy="122" r="21" fill="#374151" stroke="#1F2937" stroke-width="2.5"/>
<circle cx="648" cy="200" r="21" fill="#D6D3D1" stroke="#78716C" stroke-width="2.5"/>
<circle cx="788" cy="200" r="21" fill="#D6D3D1" stroke="#0D9488" stroke-width="3"/>
<text x="648" y="127" text-anchor="middle" font-size="14" font-weight="800" fill="#FFFFFF">A</text>
<text x="648" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#1C1917">C</text>
<text x="788" y="205" text-anchor="middle" font-size="14" font-weight="800" fill="#1C1917">B</text>
<path d="M669,133 C716,150 756,172 772,186" stroke="#0D9488" stroke-width="2.5" fill="none" marker-end="url(#wb-ar)"/>
<text x="738" y="146" text-anchor="middle" font-size="9" font-weight="800" fill="#0F766E">A.next = B</text>
<text x="788" y="236" text-anchor="middle" font-size="9" font-weight="800" fill="#0D9488">bị tô GREY ngay</text>
<text x="727" y="258" text-anchor="middle" font-size="10" fill="#0F766E">Barrier chen vào lúc ghi con trỏ, đẩy B vào hàng đợi mark.</text>
<rect x="16" y="286" width="850" height="98" rx="10" fill="#FFF3DB" stroke="#F5A623" stroke-width="2"/>
<text x="36" y="310" font-size="12" font-weight="800" fill="#7C4A03">Bất biến mà GC phải giữ: KHÔNG được tồn tại một object black trỏ thẳng tới một object white</text>
<text x="36" y="332" font-size="11" fill="#92400E">Nếu bất biến này vỡ, GC sẽ sweep nhầm một object đang được dùng → chương trình đọc phải vùng nhớ đã thu hồi.</text>
<text x="36" y="352" font-size="11" fill="#92400E">Go dùng <tspan font-weight="700">hybrid write barrier</tspan> (từ Go 1.8): tô màu cả giá trị cũ bị ghi đè lẫn giá trị mới được ghi vào.</text>
<text x="36" y="372" font-size="11" fill="#92400E">Cái giá: mỗi lệnh ghi con trỏ trong lúc mark tốn thêm vài nhịp CPU. Đổi lại, <tspan font-weight="700">không phải dừng chương trình để quét lại stack</tspan>.</text>
</svg>
</div>


**Stop-The-World (STW)**: GC của Go chỉ dừng chương trình ở 2 thời điểm rất ngắn (thường dưới 1ms): lúc bắt đầu (bật write barrier) và lúc kết thúc mark phase (mark termination). Phần lớn thời gian mark + sweep chạy concurrent.

**GOGC** (mặc định `100`): GC chu kỳ tiếp theo được kích hoạt khi heap tăng thêm 100% so với heap còn sống sau lần GC trước (tức tăng gấp đôi). Giảm `GOGC` → GC chạy thường xuyên hơn, tốn CPU hơn nhưng RAM đỉnh thấp hơn. Từ Go 1.19 có thêm **`GOMEMLIMIT`** — đặt giới hạn cứng cho tổng bộ nhớ, hữu ích khi chạy trong container có memory limit (tránh bị OOM-killed).

## Stack vs Heap — Escape Analysis

Go tự quyết định 1 biến nằm ở **stack** hay **heap** ngay lúc compile, qua **escape analysis** — không phải cứ khai báo bằng `new`/`&` là chắc chắn lên heap.

- Biến ở **stack**: tự dọn khi function return, **không tốn chi phí GC gì cả** — rẻ nhất có thể.
- Biến **escape lên heap**: GC phải track và thu hồi khi không còn reachable.

```go
func noEscape() int {
    x := 42
    return x // x ở stack — value được copy ra ngoài, x gốc mất khi hàm return
}

func escape() *int {
    x := 42
    return &x // x escape lên heap — con trỏ sống lâu hơn function, stack không đủ an toàn
}
```

Kiểm tra thật escape analysis quyết định gì (không đoán):

```bash
go build -gcflags="-m" ./...
```

Nguyên nhân phổ biến khiến biến escape: trả về pointer trỏ tới biến local, closure giữ biến rồi bản thân closure "thoát" ra khỏi hàm (return closure, hoặc truyền vào goroutine), gán biến vào `interface{}`, hoặc kích thước không xác định lúc compile (slice tăng trưởng động).

## Lưu ý khi dùng biến để tối ưu GC

### Biến global (package scope)

Biến global sống **suốt vòng đời chương trình** → luôn là 1 GC root → GC luôn phải quét qua nó ở **mọi** chu kỳ, không có ngoại lệ.

Nguy hiểm hơn: nếu biến global trỏ tới dữ liệu lớn (map/slice cache) mà không có cơ chế dọn, GC sẽ **không bao giờ thu hồi được** — vì object đó luôn reachable từ root, đúng theo định nghĩa. Đây không phải "memory leak" kiểu C (không có con trỏ mồ côi), mà là leak kiểu Go: dữ liệu sống mãi vì vẫn có người tham chiếu, dù logic không cần nữa (xem thêm câu hỏi Memory Leak phía dưới).

### Biến local (function scope)

Biến local **không escape** → nằm ở stack → gần như miễn phí với GC. Nguyên tắc tối ưu: giữ vòng đời biến càng ngắn, càng cục bộ càng tốt — đừng "nâng" biến lên phạm vi rộng hơn mức cần thiết (đừng biến local thành field của struct, đừng đưa lên global) nếu không thực sự cần chia sẻ.

## Lưu ý với Goroutine để tối ưu GC

Đây là chỗ hay bị bỏ qua nhất: **mỗi goroutine đang tồn tại (kể cả đang block) là 1 GC root.**

- Goroutine leak (channel không ai nhận, context không cancel...) không chỉ là leak logic. Goroutine đó vẫn "sống" trong runtime, nên **stack của nó và mọi object nó đang giữ tham chiếu đều là reachable** → GC nhìn vào thấy hoàn toàn hợp lệ, không thu hồi gì cả. GC "làm đúng việc của nó", chỉ là chương trình đang giữ một thứ lẽ ra phải chết.
- Đây là lý do goroutine leak thường **khó phát hiện qua GC/heap profile thông thường** — object không "mất tích", nó vẫn nằm trong tay 1 goroutine đang treo vô ích.
- Số lượng goroutine sống càng lớn (worker pool không giới hạn, spawn vô tội vạ) → càng nhiều GC root cần quét mỗi chu kỳ mark → tốn CPU hơn dù GC vẫn chạy concurrent.

Khuyến nghị:
- Luôn có đường thoát rõ ràng cho goroutine (context/channel signal).
- Giới hạn số goroutine đồng thời bằng semaphore/worker pool kích thước cố định, không spawn không giới hạn theo số request.
- Theo dõi bằng `pprof`: `/debug/pprof/goroutine` (số lượng & stack trace từng goroutine) và `/debug/pprof/heap` (ai đang giữ bộ nhớ) — số goroutine tăng đều không giảm gần như luôn là dấu hiệu leak.

## Giảm áp lực lên GC (GC pressure) trong hot path

| Kỹ thuật | Vì sao giúp GC |
|---|---|
| `sync.Pool` tái dùng object tạm (buffer, struct lớn) | Giảm số lần allocate mới → giảm số object GC phải track |
| Preallocate `make([]T, 0, n)` khi biết trước kích thước | Tránh nhiều lần realloc + copy khi `append` vượt cap |
| Giữ biến ở scope hẹp nhất có thể | Tăng khả năng biến ở stack thay vì escape lên heap |
| Batch xử lý thay vì tạo object nhỏ lẻ trong vòng lặp lớn | Giảm tổng số allocation trong hot path |
| Đặt `GOGC`/`GOMEMLIMIT` phù hợp môi trường chạy | Cân bằng CPU dùng cho GC và RAM đỉnh, tránh OOM trong container |

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf) // trả lại pool để tái sử dụng, không cấp phát mới mỗi lần gọi
    }()
    buf.Write(data)
    // ...
}
```

## Cách quan sát GC thực tế

- `GODEBUG=gctrace=1 ./myapp` — in log mỗi lần GC chạy: thời gian, heap size trước/sau, % CPU dành cho GC.
- `runtime.ReadMemStats(&m)` — đọc số liệu heap, số lần GC, pause time ngay trong code.
- `pprof` heap profile (`go tool pprof http://localhost:6060/debug/pprof/heap`) — tìm chính xác dòng code nào allocate nhiều nhất.

> ★ Tóm tắt 1 câu: GC của Go không thu hồi thứ "không dùng nữa" theo ý định của bạn — nó thu hồi thứ **không còn reachable từ root**. Biến global và goroutine đang sống đều là root, nên kiểm soát vòng đời của 2 thứ đó chính là cách tối ưu GC hiệu quả nhất, hơn hẳn việc cố "giúp" GC bằng micro-optimization.

# Memory Leak trong Go — vì sao vẫn xảy ra dù có GC, và những dạng thường gặp?

## Vì sao có GC mà vẫn leak được?

GC chỉ thu hồi những gì **không còn reachable từ root** (xem câu hỏi GC phía trên). Nếu code vẫn giữ 1 tham chiếu tới object không cần dùng nữa — dù vô tình — GC coi nó đang sống và không bao giờ đụng tới. "Leak" trong Go, khác C, không phải quên `free()`, mà là **quên "buông" một tham chiếu**.

## Các dạng leak thường gặp

### 1. Goroutine leak

Channel không ai nhận/gửi, context không cancel khiến goroutine treo vĩnh viễn. Nguy hiểm gấp đôi: vừa leak chính goroutine, vừa leak mọi object nó đang giữ tham chiếu (stack + heap object) vì goroutine đang sống luôn là GC root.

### 2. Global cache không có eviction

```go
var userCache = map[string]*User{}

func GetUser(id string) *User {
    if u, ok := userCache[id]; ok { return u }
    u := fetchFromDB(id)
    userCache[id] = u // không bao giờ xoá → phình vô hạn
    return u
}
```

Fix: TTL/LRU eviction, hoặc giới hạn kích thước map cố định.

### 3. `time.Ticker` / `time.Timer` không `Stop()`

```go
func leak() {
    ticker := time.NewTicker(time.Second)
    go func() {
        for range ticker.C { doWork() }
    }() // không Stop() → ticker + goroutine sống mãi, dù hàm leak() đã return
}
```

`time.NewTicker` giữ 1 goroutine nội bộ trong runtime để bắn tick — quên `Stop()` là leak chắc chắn, không phụ thuộc GC.

### 4. Slice giữ nguyên backing array lớn

```go
func extractHeader(data []byte) []byte {
    return data[:10] // slice mới NHƯNG dùng chung backing array với data
    // nếu data là 50MB, cả 50MB đó không được GC thu hồi dù chỉ cần 10 byte
}
```

Fix: copy ra slice độc lập nếu phần giữ lại nhỏ hơn nhiều so với phần gốc.

```go
func extractHeaderFixed(data []byte) []byte {
    header := make([]byte, 10)
    copy(header, data[:10])
    return header // backing array gốc của data giờ có thể được GC thu hồi
}
```

### 5. `context.WithCancel` quên gọi `cancel()`

Mỗi context con giữ 1 tham chiếu ngược tới cha (để propagate cancel); quên gọi `cancel()` khiến context không được gỡ khỏi cây, giữ luôn goroutine/resource liên quan. Đây là lý do quy tắc "luôn `defer cancel()`" quan trọng hơn nhiều người nghĩ.

### 6. Closure giữ biến lớn không cần thiết

```go
func handler(bigData []byte) func() {
    result := process(bigData) // chỉ cần result
    return func() {
        fmt.Println(result)
        // closure này vẫn "nhìn thấy" bigData trong scope —
        // compiler có thể giữ cả bigData sống nếu không cẩn thận
    }
}
```

## Cách phát hiện leak trong thực tế

- So sánh heap profile theo thời gian: `go tool pprof -diff_base=heap1.prof heap2.prof`.
- Theo dõi `runtime.NumGoroutine()` và RSS theo thời gian — tăng đều không giảm là dấu hiệu rõ nhất.
- `net/http/pprof` bật sẵn trong service production để lấy profile bất cứ lúc nào không cần restart.

> ★ Câu trả lời phỏng vấn tốt: không nói "Go có GC nên không leak" — nói đúng: "GC giải quyết đúng 1 bài toán là thu hồi bộ nhớ unreachable; leak trong Go luôn là do code vô tình giữ 1 reference sống lâu hơn cần thiết — goroutine, cache, ticker, hoặc slice là 4 chỗ hay gặp nhất."

# Error wrapping trong Go — `%w`, `errors.Is`, `errors.As` hoạt động thế nào?

## Trước Go 1.13

Error chỉ là 1 giá trị implement interface `error` (có method `Error() string`). Muốn giữ "nguyên nhân gốc" khi bọc thêm ngữ cảnh, phải tự làm thủ công (custom struct chứa field lỗi gốc) — không có chuẩn chung, mỗi codebase tự nghĩ 1 kiểu.

## Go 1.13: wrapping chuẩn hoá

`fmt.Errorf` thêm verb `%w`: tạo ra error mới, nhưng vẫn giữ liên kết tới error gốc qua method ngầm `Unwrap() error`.

```go
var ErrNotFound = errors.New("not found")

func findUser(id string) error {
    return fmt.Errorf("findUser %s: %w", id, ErrNotFound)
}
```

## 3 hàm cốt lõi trong package `errors`

| Hàm | Làm gì | Dùng khi |
|---|---|---|
| `errors.Unwrap(err)` | Lấy error bị bọc bên trong (1 lớp) | Ít dùng trực tiếp — `Is`/`As` dùng ngầm bên dưới |
| `errors.Is(err, target)` | Duyệt cả **chuỗi wrap** (theo Unwrap liên tiếp), true nếu gặp `target` | So sánh với **sentinel error** đã biết trước (`sql.ErrNoRows`, `io.EOF`, `ErrNotFound`...) |
| `errors.As(err, &target)` | Duyệt chuỗi wrap, tìm error nào cùng **type** với `target`, gán vào nếu thấy | Cần lấy field/method riêng của 1 **custom error type** cụ thể |

```go
err := findUser("123")

// Is: so sánh GIÁ TRỊ, xuyên qua toàn bộ chuỗi wrap
fmt.Println(errors.Is(err, ErrNotFound)) // true — dù message đã đổi khác hẳn

// As: tìm đúng TYPE, lấy được field riêng
type ValidationError struct {
    Field string
    Err   error
}
func (e *ValidationError) Error() string { return fmt.Sprintf("field %s: %v", e.Field, e.Err) }
func (e *ValidationError) Unwrap() error  { return e.Err } // bắt buộc để Is/As đi xuyên qua được

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field)
}
```

## `%v` vs `%w` — khác biệt sống còn

- `%v`: format error thành **string phẳng**, mất hoàn toàn liên kết — `errors.Is`/`errors.As` phía sau sẽ **không** xuyên qua được.
- `%w`: giữ liên kết qua `Unwrap()` — `errors.Is`/`errors.As` xuyên qua bình thường.

Dùng nhầm `%v` khi lẽ ra cần `%w` là lỗi rất phổ biến — kiểm tra lỗi bằng `errors.Is` phía trên cứ trả `false` mà không hiểu tại sao, vì chuỗi wrap đã bị cắt đứt ngay từ chỗ dùng `%v`.

## Go 1.20+: wrap nhiều error cùng lúc

Từ Go 1.20, `fmt.Errorf` cho phép nhiều `%w` trong cùng 1 lời gọi (tạo thành cây thay vì chuỗi), và có thêm `errors.Join(err1, err2, ...)` để gộp nhiều error độc lập thành 1 error duy nhất — `errors.Is`/`errors.As` vẫn duyệt được qua tất cả nhánh.

```go
err := errors.Join(errValidation, errTimeout)
fmt.Println(errors.Is(err, errTimeout)) // true
```

## Best practice

- Sentinel error (`var ErrX = errors.New(...)`) → check bằng `errors.Is`.
- Custom error type cần lấy thêm dữ liệu → check bằng `errors.As`.
- **Đừng bao giờ** so sánh lỗi bằng `err.Error() == "..."` hoặc `strings.Contains` — message có thể đổi bất cứ lúc nào, cực kỳ dễ vỡ.
- Luôn wrap kèm ngữ cảnh khi truyền lỗi lên tầng trên: `fmt.Errorf("tên hàm: %w", err)` — giúp trace được lỗi xảy ra ở đâu mà không mất nguyên nhân gốc.

# Xem thêm

<div style="margin:1.5rem 0;text-align:left;">
<a href="/pages/golang-interview-prep.html" target="_blank" rel="noopener" style="">📋 Senior Golang Interview Prepare<thêm/a>
</div>
