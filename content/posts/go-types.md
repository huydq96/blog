+++
date = '2025-05-18T10:26:21+07:00'
draft = false
title = 'Go data types'
author = 'Huy Dang Quang'
categories = ["Golang"]
tags = ["golang", "coding"]
description = 'Một số kiểu dữ liệu trong Go'
+++


# Array

- [Array](https://go.dev/ref/spec#Array_types) là tập hợp của các phần tử có cùng một kiểu dữ liệu
- Kích thước của array là một phần của kiểu dữ liệu, xác định khi khai báo, cố định không thay đổi được
- Là dạng tham trị (**value types**) không phải là dạng tham chiếu (**reference types**). Điều này có nghĩa là khi chúng được gán cho một biến mới hoặc khi truyền array như là một tham số của hàm, chúng sẽ được xử lý toàn bộ, có thể hiểu là khi đó chúng được sao chép lại toàn bộ thành một bản sao rồi mới xử lý trên bản sao đó

# Slice

- [Slice](https://go.dev/ref/spec#Slice_types) là một tham chiếu đến Array, nó mô tả một phần (hoặc toàn bộ) Array
- Nếu thay đổi giá trị của Slice sẽ làm thay đổi giá trị của Array mà nó tham chiếu đến

## So sánh `Array` và `Slice`

| Tiêu chí | Array | Sclice |
| -------- | ------- | ------- |
| Kích thước | Cố định, xác định khi khai báo, không thay đổi được | Linh hoạt, có thể thay đổi kích thước bằng hàm `append` |
| Kiểu dữ liệu | Phần kích thước là một phần của kiểu dữ liệu | Không có kích thước cố định, chỉ là một “view” trên array |
| Kiểu giá trị/tham chiếu | Value type (tham trị): gán, truyền hàm sẽ copy toàn bộ array | Reference type (tham chiếu): gán, truyền hàm chỉ copy header, cùng trỏ về một backing array |
| Zero value | Mảng với các phần tử là zero value của kiểu dữ liệu | nil (không trỏ tới array nào, length/capacity = 0) |
| So sánh | Có thể dùng toán tử ==, != nếu phần tử so sánh được | Không thể so sánh trực tiếp, phải so sánh từng phần tử |

# Map

- [Map](https://go.dev/ref/spec#Map_types) là một cấu trúc dữ liệu cung cấp một bộ các cặp **key/value** không có thứ tự
- Mỗi lần lặp qua map, thứ tự các phần tử có thể khác nhau
- Mỗi khóa (key) trong map là duy nhất
- Khóa phải là kiểu dữ liệu có thể so sánh được bằng toán tử `==`, như: boolean, number, string, pointer, channel, interface, struct (nếu tất cả trường đều so sánh được), array (nếu phần tử so sánh được). Không thể dùng slice, map, function làm khóa
- Map là kiểu dữ liệu tham chiếu (reference type): Khi gán map này cho map khác hoặc truyền vào hàm, cả hai biến sẽ trỏ đến cùng một vùng nhớ
- Map được triển khai dựa trên bảng băm (hash table)
    - Bảng băm là một cấu trúc dữ liệu **key-value** sử dụng hàm băm (hash function) để ánh xạ khóa (**key**) vào vị trí lưu trữ (**bucket**), giúp truy xuất dữ liệu với độ phức tạp trung bình O(1)


# Rune

- [rune](https://go.dev/ref/spec#Rune_literals) là một "bí danh" của kiểu `int32` trong Go, dùng để biểu diễn **một điểm mã Unicode** (Unicode code point)
- Mỗi rune có thể lưu trữ giá trị số nguyên 32-bit, đủ để đại diện cho mọi ký tự trong bảng mã Unicode (bao gồm cả ký tự đặc biệt, ký tự đa ngôn ngữ, emoji...)

Sự khác biệt giữa rune và byte:
- byte (uint8): Đại diện cho một byte, thường dùng cho dữ liệu ASCII hoặc thao tác thô với chuỗi
- rune (int32): Đại diện cho một ký tự Unicode, có thể chiếm từ 1 đến 4 byte trong chuỗi UTF-8
- Khi làm việc với chuỗi có ký tự đặc biệt (ví dụ: tiếng Việt, tiếng Trung, emoji...), nên chuyển chuỗi thành slice các rune để thao tác chính xác từng ký tự.

Ví dụ in ra từng ký tự Unicode cùng vị trí và mã code point

```go
for index, runeValue := range "hello, 世界" {
    fmt.Printf("%d -> %#U\n", index, runeValue)
}

// 0 -> U+0068 'h'
// 1 -> U+0065 'e'
// 2 -> U+006C 'l'
// 3 -> U+006C 'l'
// 4 -> U+006F 'o'
// 5 -> U+002C ','
// 6 -> U+0020 ' '
// 7 -> U+4E16 '世'
// 10 -> U+754C '界'
```
