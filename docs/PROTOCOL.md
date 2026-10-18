# OAP — Online Auction Protocol v0.1

Application-layer protocol for the Online Auction project (Topic 3, IT4062E).
Built directly on raw TCP sockets. No third-party socket library is used.

> DRAFT — nhóm phải đọc lại và sửa theo ý mình. Đừng dùng nguyên si.

---

## 1. Transport & framing

- Transport: **TCP**, một kết nối bền cho mỗi client.
- Server: C, Linux, một tiến trình, một luồng, dùng `select()`.
- Mã hóa: **UTF-8 text**.
- Một message = các field nối bằng `|`, kết thúc bằng `\n` (LF, 0x0A).
- Độ dài tối đa: **4096 byte** kể cả ký tự `\n` cuối.

### Quy tắc framing (bắt buộc)

TCP là dòng byte, không có ranh giới message. Mỗi bên giữ một buffer nhận:

```
1. recv() vào cuối buffer, cộng dồn độ dài
2. Quét buffer tìm ký tự LF
3. Tìm thấy  -> cắt ra 1 message, xử lý, dịch phần còn lại về đầu buffer
4. Lặp lại bước 2 (một lần recv() có thể chứa 0, 1 hoặc nhiều message)
5. Không thấy -> giữ nguyên buffer, chờ recv() lần sau
6. Buffer đầy mà vẫn chưa thấy LF -> message quá dài, đóng kết nối
```

`send()` cũng có thể gửi thiếu byte. Phải lặp cho đến khi gửi hết, hoặc giữ
phần còn lại trong write buffer và đẩy tiếp ở vòng `select()` sau.

### Escaping

Trong một field, các ký tự sau phải được escape:

| Ký tự gốc | Escape |
|---|---|
| backslash | `\\` |
| pipe | `\p` |
| LF | `\n` |

---

## 2. Phân loại message

| Loại | Hướng | Ghi chú |
|---|---|---|
| **Request** | C → S | Mỗi request nhận đúng 1 response |
| **Response** | S → C | `OK_*` hoặc `ERR` |
| **Event** | S → C | Server chủ động đẩy, không do client hỏi |

**Quan trọng:** client phải luôn sẵn sàng nhận event bất cứ lúc nào, kể cả khi
đang chờ response. Vì vậy client bắt buộc có một luồng đọc riêng.

### Response lỗi

```
ERR|<code>|<message>
```

| Code | Ý nghĩa |
|---|---|
| `E_SYNTAX` | Sai định dạng / thiếu field |
| `E_AUTH` | Chưa đăng nhập |
| `E_EXISTS` | Tên đăng nhập đã tồn tại |
| `E_CREDENTIAL` | Sai tài khoản hoặc mật khẩu |
| `E_ALREADY_LOGGED` | Tài khoản đang đăng nhập ở nơi khác |
| `E_NOT_FOUND` | Phòng / vật phẩm không tồn tại |
| `E_PERMISSION` | Không phải chủ phòng |
| `E_IN_ROOM` | Đang ở trong một phòng khác |
| `E_NOT_IN_ROOM` | Chưa vào phòng nào |
| `E_ROOM_STATE` | Sai trạng thái phòng (chưa bắt đầu / đã kết thúc) |
| `E_BID_TOO_LOW` | Giá trả thấp hơn giá hiện tại + 10000 |
| `E_SELF_BID` | Đang giữ giá cao nhất, không tự trả giá tiếp |
| `E_NO_ITEM` | Không có vật phẩm nào đang đấu |
| `E_INTERNAL` | Lỗi server |

---

## 3. Xác thực

```
C → S : REGISTER|<username>|<password>
S → C : OK_REGISTER
      | ERR|E_EXISTS|Username already taken

C → S : LOGIN|<username>|<password>
S → C : OK_LOGIN|<username>
      | ERR|E_CREDENTIAL|Invalid username or password
      | ERR|E_ALREADY_LOGGED|Account is already online

C → S : LOGOUT
S → C : OK_LOGOUT
```

Ràng buộc: username 3-31 ký tự `[A-Za-z0-9_]`, password không rỗng.
Mọi request khác `REGISTER` / `LOGIN` khi chưa đăng nhập → `ERR|E_AUTH`.

---

## 4. Quản lý phòng

### Trạng thái phòng

```
WAITING   chưa bắt đầu, cho phép vào phòng và thêm vật phẩm
RUNNING   đang đấu giá
FINISHED  đã hết hàng đợi, chỉ xem kết quả
```

### Danh sách phòng

```
C → S : LIST_ROOMS
S → C : OK_ROOMS|<count>
        ROOM|<room_id>|<name>|<owner>|<status>|<member_count>|<item_count>
        ROOM|...
        END_ROOMS
```

### Tạo / vào / rời phòng

```
C → S : CREATE_ROOM|<name>
S → C : OK_ROOM_CREATED|<room_id>

C → S : JOIN_ROOM|<room_id>
S → C : OK_JOINED|<room_id>|<name>|<status>|<member_count>
      | ERR|E_IN_ROOM|Leave your current room first
        (nếu phòng đang RUNNING, server gửi tiếp EVT_ITEM_START của món
         đang đấu để client dựng lại màn hình)

C → S : LEAVE_ROOM
S → C : OK_LEFT
```

Mỗi client chỉ ở **một** phòng tại một thời điểm.

---

## 5. Quản lý vật phẩm

Chỉ **chủ phòng** được thêm/xóa vật phẩm, và chỉ khi phòng ở `WAITING`.

```
C → S : ADD_ITEM|<room_id>|<name>|<desc>|<start_price>|<buynow_price>|<duration_sec>
S → C : OK_ITEM_ADDED|<item_id>
      | ERR|E_PERMISSION|Only the room owner can add items

C → S : DEL_ITEM|<room_id>|<item_id>
S → C : OK_ITEM_DELETED

C → S : LIST_ITEMS|<room_id>
S → C : OK_ITEMS|<count>
        ITEM|<item_id>|<name>|<status>|<start_price>|<buynow_price>|<final_price>|<winner>
        ITEM|...
        END_ITEMS
```

`status` của vật phẩm: `PENDING` (sắp đấu) | `ACTIVE` (đang đấu) | `SOLD` |
`UNSOLD` (hết giờ không ai trả giá).
Với `PENDING`: `final_price` = `0`, `winner` = `-`.

Giá là số nguyên VNĐ. `buynow_price = 0` nghĩa là không cho mua ngay.

---

## 6. Luồng đấu giá

```
C → S : START_AUCTION|<room_id>
S → C : OK_STARTED
      | ERR|E_PERMISSION|Only the room owner can start
      | ERR|E_ROOM_STATE|Room already started
        → broadcast EVT_ITEM_START cho cả phòng

C → S : BID|<amount>
S → C : OK_BID|<new_price>
      | ERR|E_BID_TOO_LOW|Minimum bid is <min>
      | ERR|E_SELF_BID|You are already the highest bidder
        → broadcast EVT_NEW_BID cho cả phòng

C → S : BUY_NOW
S → C : OK_BUY_NOW|<price>
      | ERR|E_NOT_FOUND|This item has no buy-now price
        → broadcast EVT_ITEM_SOLD (reason = BUYNOW)
```

### Luật trả giá

```
MIN_INCREMENT = 10000 VND
amount hợp lệ  <=>  amount >= current_price + MIN_INCREMENT
```

Với vật phẩm chưa có ai trả giá, `current_price = start_price`, nên lượt đầu
tiên phải >= `start_price + 10000`.

### Luật thời gian (chống sniping)

```
WARNING_THRESHOLD = 30 giây
EXTENSION         = 30 giây

Khi (deadline - now) <= 30 và chưa cảnh báo cho món này:
    → EVT_TIME_WARNING, đặt cờ warned = 1

Khi có BID hợp lệ và (deadline - now) <= 30:
    → deadline = now + 30
    → warned = 0
    → EVT_TIME_EXTENDED
```

### Khi hết giờ

```
Có người trả giá  → EVT_ITEM_SOLD (reason = AUCTION)
Không ai trả giá  → EVT_ITEM_UNSOLD

Còn vật phẩm PENDING → EVT_ITEM_START cho món kế tiếp
Hết hàng đợi         → EVT_AUCTION_END, phòng chuyển FINISHED
```

---

## 7. Danh sách event (S → C)

```
EVT_ITEM_START|<room_id>|<item_id>|<name>|<desc>|<start_price>|<buynow_price>|<deadline>
EVT_NEW_BID|<room_id>|<item_id>|<price>|<bidder>|<deadline>
EVT_TIME_WARNING|<room_id>|<item_id>|<seconds_left>
EVT_TIME_EXTENDED|<room_id>|<item_id>|<new_deadline>
EVT_ITEM_SOLD|<room_id>|<item_id>|<price>|<winner>|<reason>
EVT_ITEM_UNSOLD|<room_id>|<item_id>
EVT_AUCTION_END|<room_id>
EVT_USER_JOINED|<room_id>|<username>|<member_count>
EVT_USER_LEFT|<room_id>|<username>|<member_count>
EVT_KICKED|<reason>
```

`<deadline>` là **Unix timestamp** (giây). Client tự tính số giây còn lại và tự
đếm ngược trên giao diện — không bắt server gửi từng giây.

`<reason>` của `EVT_ITEM_SOLD`: `AUCTION` | `BUYNOW`.

---

## 8. Tìm kiếm & lịch sử

```
C → S : SEARCH_ITEMS|<keyword>
S → C : OK_SEARCH|<count>
        RESULT|<room_id>|<room_name>|<room_status>|<item_id>|<item_name>|<status>|<price>
        RESULT|...
        END_SEARCH

C → S : HISTORY
S → C : OK_HISTORY|<count>
        HIST|<room_id>|<room_name>|<item_id>|<item_name>|<final_price>|<won>|<ended_at>
        HIST|...
        END_HISTORY

C → S : ROOM_RESULT|<room_id>
S → C : OK_ROOM_RESULT|<count>
        RESULT|<item_id>|<item_name>|<final_price>|<winner>|<reason>
        END_ROOM_RESULT
```

`<won>` = `1` nếu người dùng thắng món đó, `0` nếu chỉ tham gia.

---

## 9. Nâng cao (v0.2 — làm sau khi lõi chạy ổn)

```
C → S : CHAT|<text>              → EVT_CHAT|<room_id>|<user>|<text>
C → S : AUTO_BID|<max_amount>    → OK_AUTO_BID|<max_amount>
C → S : AUTO_BID_CANCEL          → OK_AUTO_BID_CANCEL
S → C : EVT_OUTBID|<room_id>|<item_id>|<price>|<new_leader>   (gửi riêng)
```

Auto-bid: khi người khác trả giá, server tự trả hộ `current + 10000` cho tới khi
chạm trần `max_amount`. Xử lý ngay trong lượt bid đó, tuần tự.

---

## 10. Xử lý mất kết nối

- Client mất kết nối khi đang trong phòng: xóa khỏi danh sách thành viên,
  broadcast `EVT_USER_LEFT`. **Các lượt trả giá đã đặt vẫn có hiệu lực** —
  người mất kết nối vẫn có thể thắng.
- Chủ phòng mất kết nối khi phòng đang `RUNNING`: phiên **vẫn tiếp tục**,
  server tự chạy hết hàng đợi.
- Server đóng: gửi `EVT_KICKED|server shutdown` cho tất cả rồi `close()`.

---

## 11. Ví dụ một phiên hoàn chỉnh

```
C → S   LOGIN|binh|1234
S → C   OK_LOGIN|binh
C → S   LIST_ROOMS
S → C   OK_ROOMS|1
S → C   ROOM|1|Do co Ha Noi|an|WAITING|2|2
S → C   END_ROOMS
C → S   JOIN_ROOM|1
S → C   OK_JOINED|1|Do co Ha Noi|WAITING|3

        ... chủ phòng bấm bắt đầu ...

S → C   EVT_ITEM_START|1|1|Binh gom|Gom Bat Trang|500000|3000000|1757400000
C → S   BID|505000
S → C   ERR|E_BID_TOO_LOW|Minimum bid is 510000
C → S   BID|510000
S → C   OK_BID|510000
S → C   EVT_NEW_BID|1|1|510000|binh|1757400000

        ... Chi trả giá ...

S → C   EVT_NEW_BID|1|1|550000|chi|1757400000
S → C   EVT_TIME_WARNING|1|1|30
C → S   BID|850000
S → C   OK_BID|850000
S → C   EVT_NEW_BID|1|1|850000|binh|1757400030
S → C   EVT_TIME_EXTENDED|1|1|1757400030
S → C   EVT_ITEM_SOLD|1|1|850000|binh|AUCTION
S → C   EVT_ITEM_START|1|2|Dong ho cu|Dong ho co|200000|1000000|1757400150
S → C   EVT_ITEM_SOLD|1|2|1000000|dung|BUYNOW
S → C   EVT_AUCTION_END|1
```

---

## 12. Đối chiếu thang điểm

| Chức năng | Điểm | Message liên quan |
|---|---|---|
| Stream handling | 1 | mục 1 |
| Socket I/O trên server | 2 | vòng lặp `select()` |
| Đăng ký, quản lý tài khoản | 2 | `REGISTER` |
| Đăng nhập, quản lý phiên | 2 | `LOGIN` `LOGOUT` |
| Kiểm soát truy cập phòng | 1 | `E_PERMISSION` `E_IN_ROOM` |
| Tạo phòng | 1 | `CREATE_ROOM` |
| Liệt kê phòng | 1 | `LIST_ROOMS` |
| Tạo vật phẩm | 2 | `ADD_ITEM` |
| Xóa vật phẩm | 1 | `DEL_ITEM` |
| Vào phòng | 2 | `JOIN_ROOM` |
| Trả giá | 2 | `BID` + `EVT_NEW_BID` |
| Mua ngay | 1 | `BUY_NOW` |
| Ghi log | 1 | `server.log` |
| Cảnh báo 30s + gia hạn | 2 | `EVT_TIME_WARNING` `EVT_TIME_EXTENDED` |
| Nâng cao | 2-10 | mục 8, 9 |
| GUI | 3 | client |
