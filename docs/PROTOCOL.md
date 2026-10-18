# OAP — Online Auction Protocol v1.0.1

Application-layer protocol for the Online Auction project (Topic 3, IT4062E).
Built directly on raw TCP sockets. No third-party socket library is used.

> **Trạng thái: ĐÃ CHỐT** ngày 18/10/2026, nhóm trưởng Đồng Đại Huy chốt.
> Dữ liệu và lưu trữ phía server: xem `DATA_MODEL.md`.
> Bản v0.1 nằm trong commit đầu tiên của repo (`git log -- docs/PROTOCOL.md`). Các thay đổi so với v0.1 xem mục 15.
> Mã `P1`…`P17` trong tài liệu này trỏ tới trang rà soát v0.1.

---

## 0. Quy tắc sửa tài liệu này

1. **Sửa tài liệu trước, sửa code sau.** Muốn đổi message nào thì sửa file này trước.
2. Tăng số phiên bản (v1.0 → v1.1) và ghi một dòng vào mục 15.
3. Báo cả nhóm. Thay đổi đụng tới phần của ai thì người đó phải biết trước khi merge.

---

## 1. Transport & framing

- Transport: **TCP**, một kết nối bền cho mỗi client.
- Server: C, Linux, một tiến trình, một luồng, dùng `select()`.
- Mã hóa: **UTF-8 text**.
- Một dòng = các field nối bằng `|`, kết thúc bằng `\n` (LF, 0x0A).
- Độ dài tối đa: **4096 byte** mỗi dòng, kể cả `\n` cuối. Áp dụng cho từng dòng
  của response nhiều dòng.
- Số là số nguyên hệ 10, không có dấu phân cách hàng nghìn.
- Giá trị "không có" ghi bằng `-` (ví dụ chưa có người dẫn giá). Username không
  thể là `-` vì chỉ gồm `[A-Za-z0-9_]`.

### Quy tắc framing (bắt buộc)

TCP là dòng byte, không có ranh giới message. Mỗi bên giữ một buffer nhận:

```
1. recv() vào cuối buffer, cộng dồn độ dài
2. Quét buffer tìm ký tự LF
3. Tìm thấy  -> cắt ra 1 dòng, xử lý, dịch phần còn lại về đầu buffer
4. Lặp lại bước 2 (một lần recv() có thể chứa 0, 1 hoặc nhiều dòng)
5. Không thấy -> giữ nguyên buffer, chờ recv() lần sau
6. Buffer đầy (4096 byte) mà vẫn chưa thấy LF
   -> gửi ERR|E_SYNTAX|Message too long rồi đóng kết nối
```

`send()` cũng có thể gửi thiếu byte. Client (Python) dùng `sendall()`. Server
giữ phần còn lại trong send buffer của client đó và gửi tiếp khi `select()` báo
socket ghi được.

### Dữ liệu vào không chuẩn (P15)

- Dòng kết thúc bằng `\r\n`: bỏ `\r` ở cuối rồi xử lý bình thường.
- Dòng rỗng: bỏ qua, không trả lời.
- Escape không có trong bảng dưới (ví dụ `\x`): `ERR|E_SYNTAX`.

### Escaping

Trong một field, các ký tự sau phải được escape. Bên gửi escape từng field rồi
mới nối bằng `|`. Bên nhận tách theo `|` rồi mới bỏ escape từng field.

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
| **Event** | S → C | Server chủ động đẩy, luôn bắt đầu bằng `EVT_` |

**Quan trọng:** client phải luôn sẵn sàng nhận event bất cứ lúc nào, kể cả khi
đang chờ response. Vì vậy client bắt buộc có một luồng đọc riêng.

### Thứ tự (P6)

1. Server xử lý request của mỗi kết nối **lần lượt theo thứ tự nhận**.
2. Client chỉ gửi request tiếp theo **sau khi đã nhận xong response** của
   request trước. Nhờ vậy response luôn khớp với request đang chờ.
3. Response luôn được gửi **trước** các event do chính request đó gây ra.
   Ví dụ: `OK_BID` trước `EVT_NEW_BID`, `OK_JOINED` trước `EVT_ITEM_START`.
4. Response nhiều dòng luôn liền một khối, **không có event chen giữa**:
   ```
   OK_<X>|<count>
   <count> dòng dữ liệu
   END_<X>
   ```
   Server ghi cả khối vào send buffer trong một lần.
5. Client phân loại dòng nhận được: bắt đầu bằng `EVT_` là event, còn lại thuộc
   response đang chờ.

### Response lỗi

```
ERR|<code>|<message>
```

`<message>` là câu tiếng Anh để đọc và ghi log. **Client xử lý theo `<code>`**,
không dựa vào `<message>`.

| Code | Ý nghĩa |
|---|---|
| `E_UNKNOWN_CMD` | Lệnh không có trong protocol |
| `E_SYNTAX` | Sai định dạng, sai số field, escape lạ, dòng quá dài |
| `E_INVALID` | Giá trị ngoài giới hạn ở mục 3 (độ dài, khoảng giá, thời lượng…) |
| `E_AUTH` | Chưa đăng nhập |
| `E_EXISTS` | Tên đăng nhập đã tồn tại |
| `E_CREDENTIAL` | Sai tài khoản hoặc mật khẩu |
| `E_ALREADY_LOGGED` | Kết nối này đã đăng nhập rồi |
| `E_NOT_FOUND` | Phòng / vật phẩm không tồn tại |
| `E_PERMISSION` | Không có quyền (không phải chủ phòng, hoặc chủ phòng tự trả giá) |
| `E_IN_ROOM` | Đang ở trong một phòng |
| `E_NOT_IN_ROOM` | Chưa vào phòng nào |
| `E_ROOM_STATE` | Sai trạng thái phòng (chưa bắt đầu / đã bắt đầu / đã kết thúc) |
| `E_NO_ITEM` | Phòng không có vật phẩm nào để đấu |
| `E_ITEM_CLOSED` | Vật phẩm không còn đang đấu (đã đóng hoặc chưa lên sàn) |
| `E_BID_TOO_LOW` | Giá trả thấp hơn giá hiện tại + 10000 |
| `E_SELF_BID` | Đang giữ giá cao nhất, không tự trả giá tiếp |
| `E_NO_BUYNOW` | Vật phẩm không có giá mua ngay, hoặc giá đấu đã đạt giá mua ngay |
| `E_INTERNAL` | Lỗi server |

### Thứ tự kiểm tra lỗi

Server kiểm tra theo thứ tự sau và trả **lỗi đầu tiên** gặp phải:

```
E_UNKNOWN_CMD → E_SYNTAX → E_AUTH → E_INVALID → các lỗi riêng của từng lệnh
```

Lỗi riêng của từng lệnh được kiểm tra theo **đúng thứ tự liệt kê** ở mục của lệnh đó.

---

## 3. Giới hạn giá trị (P7)

Vượt giới hạn → `ERR|E_INVALID|<mô tả>`. Độ dài tính bằng **byte UTF-8** sau khi
bỏ escape.

| Giá trị | Giới hạn |
|---|---|
| username | 3–31 ký tự `[A-Za-z0-9_]`, phân biệt hoa thường |
| password | 1–64 byte |
| tên phòng, tên vật phẩm | 1–100 byte |
| mô tả vật phẩm | 0–1000 byte |
| từ khóa tìm kiếm | 1–100 byte |
| `start_price` | 1 … 1 000 000 000 000 (10¹²) VND |
| `buynow_price` | `0` (không cho mua ngay), hoặc từ `start_price + 10000` tới 10¹² |
| `amount` của BID | 1 … 10¹² |
| `duration_sec` | 60–3600 giây |

Giá dùng kiểu `long long` (64 bit) trong C. `int` chỉ chứa được tới khoảng 2,1 tỷ.

---

## 4. Xác thực

```
C → S : REGISTER|<username>|<password>
S → C : OK_REGISTER
      | ERR|E_EXISTS|Username already taken
      | ERR|E_INVALID|...
      | ERR|E_ALREADY_LOGGED|Log out first

C → S : LOGIN|<username>|<password>
S → C : OK_LOGIN|<username>
      | ERR|E_CREDENTIAL|Invalid username or password
      | ERR|E_ALREADY_LOGGED|This connection is already logged in

C → S : LOGOUT
S → C : OK_LOGOUT
```

- `REGISTER` không tự đăng nhập. Client gửi `LOGIN` sau đó.
- Mọi request khác `REGISTER` / `LOGIN` khi chưa đăng nhập → `ERR|E_AUTH`.
- **Đăng nhập ở nơi khác (P10):** nếu tài khoản đang online trên một kết nối
  khác, lần `LOGIN` mới **thắng**. Server gửi `EVT_KICKED|logged in elsewhere`
  cho kết nối cũ, cho kết nối cũ rời phòng (như `LOGOUT`), đóng kết nối cũ, rồi
  trả `OK_LOGIN` cho kết nối mới. Cách này cũng giải phóng các phiên "ma" của
  client bị treo hoặc mất mạng.
- **`LOGOUT` khi đang ở trong phòng (P10):** server tự làm `LEAVE_ROOM` trước
  (gửi `EVT_USER_LEFT` cho phòng). Các lượt giá đã trả vẫn có hiệu lực. Kết nối
  TCP vẫn giữ, client có thể `LOGIN` lại.
- **Mật khẩu (P16):** server không lưu mật khẩu nguyên văn mà lưu dạng băm kèm
  salt, bằng `crypt()` kiểu SHA-512 (`DATA_MODEL.md` mục 7). Mật khẩu đi trên mạng ở
  dạng rõ, chấp nhận được trong phạm vi môn học và sẽ ghi rõ trong báo cáo.

---

## 5. Quản lý phòng

### Trạng thái phòng

```
WAITING   chưa bắt đầu: vào phòng được, chủ phòng thêm / xóa vật phẩm được
RUNNING   đang đấu giá: vào phòng được
FINISHED  đã hết hàng đợi: không vào được, xem kết quả bằng ROOM_RESULT
```

Phòng không bị xóa. Quyền chủ phòng gắn với **username**, không gắn với kết nối.

### Danh sách phòng

```
C → S : LIST_ROOMS
S → C : OK_ROOMS|<count>
        ROOM|<room_id>|<name>|<owner>|<status>|<member_count>|<item_count>
        ROOM|...
        END_ROOMS
```

Liệt kê mọi phòng, kể cả `FINISHED`, theo `room_id` tăng dần. `item_count` là
tổng số vật phẩm của phòng.

### Tạo / vào / rời phòng

```
C → S : CREATE_ROOM|<name>
S → C : OK_ROOM_CREATED|<room_id>
      | ERR|E_IN_ROOM|Leave your current room first
      | ERR|E_INVALID|...

C → S : JOIN_ROOM|<room_id>
S → C : OK_JOINED|<room_id>|<name>|<owner>|<status>|<member_count>
      | ERR|E_NOT_FOUND|Room not found
      | ERR|E_IN_ROOM|Leave your current room first
      | ERR|E_ROOM_STATE|Room has finished

C → S : LEAVE_ROOM
S → C : OK_LEFT
      | ERR|E_NOT_IN_ROOM|You are not in a room
```

- **Mỗi client chỉ ở một phòng** tại một thời điểm. Đang ở phòng nào (kể cả
  chính phòng đó) mà gửi `JOIN_ROOM` → `E_IN_ROOM`.
- **Người tạo phòng tự vào phòng (P9).** Sau `OK_ROOM_CREATED`, người tạo đã ở
  trong phòng, không cần gửi `JOIN_ROOM`.
- Sau `OK_JOINED`:
  - nếu phòng đang `RUNNING`, server gửi riêng cho người vừa vào một
    `EVT_ITEM_START` mang **trạng thái hiện tại** của món đang đấu (P2);
  - server gửi `EVT_USER_JOINED` cho **các thành viên khác** trong phòng.
- Sau `OK_LEFT`, server gửi `EVT_USER_LEFT` cho các thành viên còn lại.
- **Chủ phòng rời phòng hoặc mất kết nối khi phòng còn `WAITING` (P9):** phòng
  vẫn còn. Chủ phòng vào lại được và vẫn quản lý được.
- **Phòng chuyển `FINISHED`:** thành viên vẫn ở trong phòng cho tới khi gửi
  `LEAVE_ROOM`.

---

## 6. Quản lý vật phẩm

Chỉ **chủ phòng** được thêm / xóa vật phẩm, và chỉ khi phòng ở `WAITING`.
Chủ phòng không cần đang ở trong phòng (lệnh đã mang `room_id`).

```
C → S : ADD_ITEM|<room_id>|<name>|<desc>|<start_price>|<buynow_price>|<duration_sec>
S → C : OK_ITEM_ADDED|<item_id>
      | ERR|E_NOT_FOUND|Room not found
      | ERR|E_PERMISSION|Only the room owner can add items
      | ERR|E_ROOM_STATE|Room has already started
      | ERR|E_INVALID|...

C → S : DEL_ITEM|<room_id>|<item_id>
S → C : OK_ITEM_DELETED
      | ERR|E_NOT_FOUND|Room or item not found
      | ERR|E_PERMISSION|Only the room owner can delete items
      | ERR|E_ROOM_STATE|Room has already started

C → S : LIST_ITEMS|<room_id>
S → C : OK_ITEMS|<count>
        ITEM|<item_id>|<name>|<desc>|<status>|<start_price>|<buynow_price>|<duration_sec>|<price>|<leader>
        ITEM|...
        END_ITEMS
      | ERR|E_NOT_FOUND|Room not found
```

- `item_id` là số duy nhất trong toàn hệ thống.
- **Thứ tự hàng đợi** = thứ tự thêm vào (`item_id` tăng dần).
- `LIST_ITEMS` dùng được với phòng ở mọi trạng thái, không cần vào phòng. Đề bài
  yêu cầu xem được món đã đấu, đang đấu và sắp đấu.
- `buynow_price = 0` nghĩa là không cho mua ngay.

`status` của vật phẩm và ý nghĩa của `<price>`, `<leader>` (P13):

| `status` | Nghĩa | `<price>` | `<leader>` |
|---|---|---|---|
| `PENDING` | đang chờ trong hàng đợi | `start_price` | `-` |
| `ACTIVE` | đang đấu | giá hiện tại | người đang dẫn, `-` nếu chưa ai trả |
| `SOLD` | đã bán | giá bán | người thắng |
| `UNSOLD` | hết giờ, không ai trả giá | `start_price` | `-` |

---

## 7. Luồng đấu giá

```
C → S : START_AUCTION|<room_id>
S → C : OK_STARTED
      | ERR|E_NOT_FOUND|Room not found
      | ERR|E_PERMISSION|Only the room owner can start
      | ERR|E_ROOM_STATE|Room already started
      | ERR|E_NO_ITEM|Room has no items
        → phòng chuyển RUNNING, broadcast EVT_ITEM_START của món đầu tiên

C → S : BID|<item_id>|<amount>
S → C : OK_BID|<item_id>|<new_price>
      | OK_BUY_NOW|<item_id>|<buynow_price>     (khi amount ≥ buynow_price, xem dưới)
      | ERR|E_NOT_IN_ROOM|You are not in a room
      | ERR|E_ROOM_STATE|Auction is not running
      | ERR|E_ITEM_CLOSED|Item is no longer on the floor
      | ERR|E_PERMISSION|Room owner cannot bid
      | ERR|E_SELF_BID|You are already the highest bidder
      | ERR|E_BID_TOO_LOW|Minimum bid is <min>
        → broadcast EVT_NEW_BID (và EVT_TIME_EXTENDED nếu được gia hạn)

C → S : BUY_NOW|<item_id>
S → C : OK_BUY_NOW|<item_id>|<price>
      | ERR|E_NOT_IN_ROOM|You are not in a room
      | ERR|E_ROOM_STATE|Auction is not running
      | ERR|E_ITEM_CLOSED|Item is no longer on the floor
      | ERR|E_PERMISSION|Room owner cannot buy
      | ERR|E_NO_BUYNOW|Buy-now is not available for this item
        → broadcast EVT_ITEM_SOLD (reason = BUYNOW), rồi món kế tiếp
```

`BID` và `BUY_NOW` áp dụng cho phòng mà client đang ở. `item_id` phải đúng là
món đang `ACTIVE` của phòng đó, nếu không → `E_ITEM_CLOSED` (P1). Client lấy
`item_id` từ `EVT_ITEM_START`.

### Luật trả giá

```
MIN_INCREMENT = 10000 VND
amount hợp lệ  <=>  amount >= current_price + MIN_INCREMENT
```

- Chưa ai trả giá thì `current_price = start_price`, nên lượt đầu tiên phải
  ≥ `start_price + 10000`.
- **Chủ phòng không được trả giá hay mua ngay** trong phòng của mình (P11).
- Người đang giữ giá cao nhất không được trả tiếp (`E_SELF_BID`), nhưng **được**
  mua ngay.

### Luật mua ngay (P3)

- `BUY_NOW` chỉ hợp lệ khi `buynow_price > 0` **và** `current_price < buynow_price`.
  Ngược lại → `E_NO_BUYNOW`.
- Một lượt `BID` hợp lệ có `amount ≥ buynow_price` (với `buynow_price > 0`) được
  xử lý **như `BUY_NOW`**: món bán ngay với giá `buynow_price`, server trả
  `OK_BUY_NOW|<item_id>|<buynow_price>` thay cho `OK_BID`.

### Luật thời gian (chống sniping)

```
WARNING_THRESHOLD = 30 giây
EXTENSION         = 30 giây

Khi món lên sàn:
    deadline = now + duration_sec, warned = 0

Mỗi vòng lặp, khi warned = 0 và (deadline - now) <= 30:
    → EVT_TIME_WARNING, warned = 1

Khi có BID hợp lệ và (deadline - now) <= 30:
    → deadline = now + 30
    → warned = 1                (P5: không gửi lại EVT_TIME_WARNING)
    → EVT_NEW_BID rồi EVT_TIME_EXTENDED
```

- **BID chỉ hợp lệ nếu server đọc được nó trước deadline (P12).** Khi xử lý
  `BID` hoặc `BUY_NOW`, server kiểm tra `now >= deadline` trước tiên. Nếu đã quá
  giờ, server đóng món (gửi event như mục "Khi hết giờ"), rồi trả
  `E_ITEM_CLOSED` cho lượt đó.
- `seconds_left` trong event = số giây còn lại, làm tròn lên, tính theo đồng hồ
  server (P4). Client tự tính deadline = giờ máy client + `seconds_left`.

### Khi hết giờ

```
Có người trả giá  → EVT_ITEM_SOLD (reason = AUCTION)
Không ai trả giá  → EVT_ITEM_UNSOLD

Còn vật phẩm PENDING → EVT_ITEM_START cho món kế tiếp, ngay lập tức
Hết hàng đợi         → EVT_AUCTION_END, phòng chuyển FINISHED
```

---

## 8. Danh sách event (S → C)

```
EVT_ITEM_START|<room_id>|<item_id>|<name>|<desc>|<start_price>|<buynow_price>|<current_price>|<leader>|<seconds_left>
EVT_NEW_BID|<room_id>|<item_id>|<price>|<bidder>|<seconds_left>
EVT_TIME_WARNING|<room_id>|<item_id>|<seconds_left>
EVT_TIME_EXTENDED|<room_id>|<item_id>|<seconds_left>
EVT_ITEM_SOLD|<room_id>|<item_id>|<price>|<winner>|<reason>
EVT_ITEM_UNSOLD|<room_id>|<item_id>
EVT_AUCTION_END|<room_id>
EVT_USER_JOINED|<room_id>|<username>|<member_count>
EVT_USER_LEFT|<room_id>|<username>|<member_count>
EVT_KICKED|<reason>
```

| Event | Gửi cho |
|---|---|
| `EVT_ITEM_START` | Cả phòng khi món lên sàn. Riêng người vừa vào khi vào phòng đang `RUNNING` |
| `EVT_NEW_BID`, `EVT_TIME_WARNING`, `EVT_TIME_EXTENDED` | Cả phòng, kể cả người vừa trả giá |
| `EVT_ITEM_SOLD`, `EVT_ITEM_UNSOLD`, `EVT_AUCTION_END` | Cả phòng |
| `EVT_USER_JOINED`, `EVT_USER_LEFT` | Các thành viên khác, không gửi cho chính người vào / rời |
| `EVT_KICKED` | Một kết nối. Server đóng kết nối ngay sau đó |

- Khi món mới lên sàn: `current_price = start_price`, `leader = -`,
  `seconds_left = duration_sec`.
- `<reason>` của `EVT_ITEM_SOLD`: `AUCTION` | `BUYNOW`.
- `<reason>` của `EVT_KICKED`: `logged in elsewhere` | `server shutdown`.

---

## 9. Tìm kiếm & lịch sử

```
C → S : SEARCH_ITEMS|<keyword>
S → C : OK_SEARCH|<count>
        FOUND|<room_id>|<room_name>|<room_status>|<item_id>|<item_name>|<status>|<price>
        FOUND|...
        END_SEARCH

C → S : HISTORY
S → C : OK_HISTORY|<count>
        HIST|<room_id>|<room_name>|<item_id>|<item_name>|<final_price>|<won>|<ended_at>
        HIST|...
        END_HISTORY

C → S : ROOM_RESULT|<room_id>
S → C : OK_ROOM_RESULT|<count>
        RESULT|<item_id>|<item_name>|<status>|<final_price>|<winner>|<reason>
        RESULT|...
        END_ROOM_RESULT
      | ERR|E_NOT_FOUND|Room not found
```

- **SEARCH_ITEMS:** tìm chuỗi con trong tên và mô tả vật phẩm, không phân biệt
  hoa thường với chữ cái ASCII. Trả tối đa 50 kết quả, theo `item_id` tăng dần.
  `<price>` có nghĩa như bảng ở mục 6.
- **HISTORY:** các món đã kết thúc mà người dùng **đã tham gia**, tức là đã có ít
  nhất một lượt BID hợp lệ hoặc đã mua ngay. `<won>` = `1` nếu người dùng thắng
  món đó, `0` nếu chỉ tham gia. `<ended_at>` là Unix timestamp theo giờ server,
  chỉ dùng để hiển thị. Mới nhất ở trên.
- **ROOM_RESULT:** các món **đã kết thúc** của phòng (`SOLD` hoặc `UNSOLD`), dùng
  được khi phòng `RUNNING` hoặc `FINISHED`. Với `UNSOLD`: `final_price = 0`,
  `winner = -`, `reason = -`.
- Tìm theo khung giờ (time slot) chưa chốt, xem mục 14.

---

## 10. Nâng cao (sẽ chốt ở v1.1, sau khi phần lõi chạy ổn)

```
C → S : CHAT|<text>                      → EVT_CHAT|<room_id>|<user>|<text>
C → S : AUTO_BID|<item_id>|<max_amount>  → OK_AUTO_BID|<item_id>|<max_amount>
C → S : AUTO_BID_CANCEL|<item_id>        → OK_AUTO_BID_CANCEL|<item_id>
S → C : EVT_OUTBID|<room_id>|<item_id>|<price>|<new_leader>   (gửi riêng)
```

Auto-bid: khi người khác trả giá, server tự trả hộ `current + 10000` cho tới khi
chạm trần `max_amount`. Xử lý ngay trong lượt bid đó, tuần tự. Luật chi tiết (hai
người cùng bật auto-bid, tương tác với gia hạn 30 giây) chốt ở v1.1.

---

## 11. Mất kết nối & khởi động lại

- **Client mất kết nối khi đang trong phòng:** xóa khỏi danh sách thành viên,
  broadcast `EVT_USER_LEFT`. **Các lượt trả giá đã đặt vẫn có hiệu lực**, người
  mất kết nối vẫn có thể thắng và thấy kết quả trong `HISTORY`.
- **Chủ phòng mất kết nối khi phòng đang `RUNNING`:** phiên **vẫn tiếp tục**,
  server tự chạy hết hàng đợi.
- **Server đóng:** gửi `EVT_KICKED|server shutdown` cho tất cả rồi `close()`.
- **Server khởi động lại (P17):** phòng nào đang `RUNNING` chuyển thành
  `FINISHED`, và mọi món `ACTIVE` hoặc `PENDING` của phòng đó chuyển thành
  `UNSOLD`. Server ghi việc này vào log. Chi tiết ở `DATA_MODEL.md` mục 6.

---

## 12. Ví dụ một phiên hoàn chỉnh

Góc nhìn của client Bình. An là chủ phòng, Chi và Dũng là người tham gia khác.

```
C → S   LOGIN|binh|1234
S → C   OK_LOGIN|binh
C → S   LIST_ROOMS
S → C   OK_ROOMS|1
S → C   ROOM|1|Do co Ha Noi|an|WAITING|2|2
S → C   END_ROOMS
C → S   JOIN_ROOM|1
S → C   OK_JOINED|1|Do co Ha Noi|an|WAITING|3

        ... An gửi START_AUCTION|1 ...

S → C   EVT_ITEM_START|1|1|Binh gom|Gom Bat Trang|500000|3000000|500000|-|120
C → S   BID|1|505000
S → C   ERR|E_BID_TOO_LOW|Minimum bid is 510000
C → S   BID|1|510000
S → C   OK_BID|1|510000
S → C   EVT_NEW_BID|1|1|510000|binh|95

        ... Chi trả 550000 ...

S → C   EVT_NEW_BID|1|1|550000|chi|62
S → C   EVT_TIME_WARNING|1|1|30
C → S   BID|1|850000                         (lúc còn 12 giây)
S → C   OK_BID|1|850000
S → C   EVT_NEW_BID|1|1|850000|binh|30
S → C   EVT_TIME_EXTENDED|1|1|30

        ... 30 giây không ai trả thêm ...

S → C   EVT_ITEM_SOLD|1|1|850000|binh|AUCTION
S → C   EVT_ITEM_START|1|2|Dong ho cu|Dong ho co|200000|1000000|200000|-|120

        ... Dũng gửi BUY_NOW|2 ...

S → C   EVT_ITEM_SOLD|1|2|1000000|dung|BUYNOW
S → C   EVT_AUCTION_END|1
```

---

## 13. Đối chiếu thang điểm

| Chức năng | Điểm | Message liên quan |
|---|---|---|
| Stream handling | 1 | mục 1 |
| Socket I/O trên server | 2 | vòng lặp `select()` |
| Đăng ký, quản lý tài khoản | 2 | `REGISTER` |
| Đăng nhập, quản lý phiên | 2 | `LOGIN` `LOGOUT` `EVT_KICKED` |
| Kiểm soát truy cập phòng | 1 | `E_PERMISSION` `E_IN_ROOM` `E_NOT_IN_ROOM` |
| Tạo phòng | 1 | `CREATE_ROOM` |
| Liệt kê phòng | 1 | `LIST_ROOMS` |
| Tạo vật phẩm | 2 | `ADD_ITEM` |
| Xóa vật phẩm | 1 | `DEL_ITEM` |
| Vào phòng | 2 | `JOIN_ROOM` |
| Trả giá | 2 | `BID` + `EVT_NEW_BID` |
| Mua ngay | 1 | `BUY_NOW` |
| Ghi log | 1 | `server.log` |
| Cảnh báo 30s + gia hạn | 2 | `EVT_TIME_WARNING` `EVT_TIME_EXTENDED` |
| Nâng cao | 2-10 | mục 9, 10 |
| GUI | 3 | client |

---

## 14. Còn mở

| Mục | Nội dung | Khi nào chốt |
|---|---|---|
| P14 | Đề bài yêu cầu "search by time slot". Phương án A: thêm giờ bắt đầu dự kiến cho phòng (`CREATE_ROOM|name|start_at`) và tìm theo khoảng giờ. Phương án B: tìm theo khoảng thời gian món đã được đấu (`ended_at`). | Hỏi thầy ở Checkpoint 1 |
| P16 | Đã chọn `crypt()` có sẵn của hệ thống (`DATA_MODEL.md` mục 7). Cần thầy xác nhận được phép dùng | Hỏi thầy ở Checkpoint 1 |
| — | Luật chi tiết của chat và auto-bid | v1.1 |

---

## 15. Lịch sử thay đổi

### v1.0.1 — 18/10/2026

- Mục 11: khi server khởi động lại, món `PENDING` của phòng đang đấu dở cũng
  chuyển `UNSOLD`, khớp với `DATA_MODEL.md` mục 6.
- Mục 4 và mục 14: ghi rõ cách băm mật khẩu là `crypt()` kiểu SHA-512.

### v1.0 — 18/10/2026

- **P1** `BID|<item_id>|<amount>` và `BUY_NOW|<item_id>`. Thêm `E_ITEM_CLOSED`.
  `OK_BID` và `OK_BUY_NOW` trả kèm `item_id`.
- **P2** `EVT_ITEM_START` mang trạng thái hiện tại (`current_price`, `leader`),
  dùng cả khi vào phòng đang chạy.
- **P3** Chỉ mua ngay được khi giá hiện tại < giá mua ngay. BID ≥ giá mua ngay
  được xử lý như mua ngay. `buynow_price` = 0 hoặc ≥ `start_price + 10000`. Thêm
  `E_NO_BUYNOW`, bỏ dùng `E_NOT_FOUND` cho mua ngay.
- **P4** Mọi event dùng `seconds_left` thay cho `deadline` (Unix timestamp).
- **P5** Gia hạn giữ `warned = 1`, không gửi lại `EVT_TIME_WARNING`.
- **P6** Thêm luật thứ tự: request lần lượt, response trước event, response
  nhiều dòng liền khối.
- **P7** Thêm bảng giới hạn giá trị (mục 3) và mã lỗi `E_INVALID`.
- **P8** Thêm `E_UNKNOWN_CMD`. Mỗi lệnh liệt kê đủ lỗi. Thêm thứ tự kiểm tra lỗi.
- **P9** Người tạo phòng tự vào phòng. Không vào được phòng `FINISHED`. Phòng
  không bị xóa khi chủ phòng rời đi. `OK_JOINED` thêm `owner`.
- **P10** Đăng nhập mới đá phiên cũ (`EVT_KICKED|logged in elsewhere`). `LOGOUT`
  kéo theo `LEAVE_ROOM`.
- **P11** Chủ phòng không được trả giá hay mua ngay trong phòng mình.
- **P12** BID chỉ hợp lệ nếu server đọc được trước deadline.
- **P13** Dòng tìm kiếm đổi từ `RESULT` thành `FOUND`. Dòng `ITEM` thêm `desc`,
  `duration_sec` và quy định nghĩa của `price`, `leader` theo trạng thái.
  `ROOM_RESULT` thêm `status`.
- **P15** Bỏ `\r` cuối dòng, bỏ qua dòng rỗng, báo lỗi trước khi đóng kết nối vì
  dòng quá dài.
- **P16** Mật khẩu lưu dạng băm kèm salt.
- **P17** Luật khi server khởi động lại giữa phiên đấu.

### v0.1

Bản nháp đầu tiên, xem commit đầu tiên của repo.
