# Mô hình dữ liệu & lưu trữ — Online Auction

Tài liệu đi kèm `PROTOCOL.md` v1.0.1. Quy định dữ liệu server giữ trong RAM, cách
lưu xuống file, cách băm mật khẩu và format log.

> **Trạng thái: ĐÃ CHỐT v1.0.1** ngày 18/10/2026, nhóm trưởng Đồng Đại Huy chốt.
> Sửa tài liệu theo đúng quy tắc ở mục 0 của `PROTOCOL.md`: sửa tài liệu trước,
> tăng phiên bản, ghi vào mục 11, báo nhóm.

| Ai dựa vào phần nào | |
|---|---|
| Huy | Toàn bộ: struct, lưu trữ (mục 4–6), mật khẩu (mục 7), log (mục 8) |
| Minh | Struct phòng và vật phẩm (mục 2–3), các bất biến (mục 9) |
| Hòa | Dữ liệu phía client (mục 10) |

---

## 0. Quyết định trong tài liệu này

| # | Quyết định | Lý do | Phương án đã bỏ |
|---|---|---|---|
| D1 | **Lưu bằng file text**, mỗi bản ghi một dòng, dùng lại format và escape của protocol | Không thêm thư viện. Dễ đọc và sửa tay khi debug. Dữ liệu nhỏ (vài trăm dòng). Dùng lại được hàm escape đã viết cho protocol | SQLite: tìm kiếm và thống kê dễ hơn, nhưng thêm thư viện và thêm một thứ phải học |
| D2 | **RAM là nguồn dữ liệu chính khi server chạy.** Khởi động thì đọc hết file vào RAM. Mỗi khi dữ liệu đổi thì ghi lại file | Server một luồng, mọi truy vấn chỉ đọc RAM nên nhanh và đơn giản | Đọc file mỗi lần có truy vấn |
| D3 | **Mảng tĩnh kích thước cố định** cho user, phòng, vật phẩm, client | Ít lỗi bộ nhớ hơn `malloc` / `realloc`. Đủ dùng cho bài tập lớn | Mảng động, danh sách liên kết |
| D4 | **Lượt trả giá ghi nối tiếp vào `bids.txt`**, không ghi lại cả file | Trả giá xảy ra nhiều nhất, ghi nối tiếp nhanh nhất. File này cũng là nhật ký để làm `HISTORY` | Ghi lại `items.txt` sau mỗi lượt trả giá |
| D5 | **Băm mật khẩu bằng `crypt()` của hệ thống, kiểu SHA-512 (`$6$`)** | Có sẵn trên Ubuntu (libxcrypt), gọi một hàm là xong, tự kèm salt. Không phải thư viện socket nên không vi phạm quy định | Tự viết SHA-256. OpenSSL (to và khó dùng hơn) |

D5 vẫn nên **hỏi thầy một câu ở Checkpoint 1** cho chắc.

---

## 1. Hằng số

```c
/* Giới hạn từ PROTOCOL.md mục 3 */
#define MAX_USERNAME   31
#define MAX_PASSWORD   64
#define MAX_NAME       100      /* tên phòng, tên vật phẩm (byte) */
#define MAX_DESC       1000
#define MAX_KEYWORD    100
#define MAX_LINE       4096     /* một dòng protocol, kể cả '\n' */

#define MIN_INCREMENT  10000LL
#define MAX_PRICE      1000000000000LL   /* 10^12 */
#define MIN_DURATION   60
#define MAX_DURATION   3600
#define WARN_SECONDS   30
#define EXTEND_SECONDS 30

/* Sức chứa (D3). Đầy thì trả ERR|E_INTERNAL|Server is full */
#define MAX_CLIENTS    1000     /* nhỏ hơn FD_SETSIZE (1024) */
#define MAX_USERS      1000
#define MAX_ROOMS      200
#define MAX_ITEMS      2000
#define MAX_SEARCH     50       /* số kết quả SEARCH_ITEMS tối đa */

#define OUT_BUF_LIMIT  (1 << 20) /* 1 MB, xem mục 2 */

typedef long long money_t;      /* VND, 64 bit */
```

---

## 2. Dữ liệu trong RAM

Field ghi `/* RAM */` **không lưu xuống file**. Mất khi server tắt.

```c
typedef enum { ROOM_WAITING, ROOM_RUNNING, ROOM_FINISHED } room_status_t;
typedef enum { ITEM_PENDING, ITEM_ACTIVE, ITEM_SOLD, ITEM_UNSOLD } item_status_t;
typedef enum { REASON_NONE, REASON_AUCTION, REASON_BUYNOW } sold_reason_t;

typedef struct {
    char   username[MAX_USERNAME + 1];
    char   pass_hash[128];          /* chuỗi do crypt() trả về, gồm cả salt */
    time_t created_at;
    int    online_client;           /* RAM: chỉ số trong mảng clients, -1 nếu offline */
} user_t;

typedef struct {
    int           id;               /* bắt đầu từ 1 */
    char          name[MAX_NAME + 1];
    char          owner[MAX_USERNAME + 1];
    room_status_t status;
    time_t        created_at;
    int           active_item;      /* RAM: item_id đang đấu, 0 nếu không có */
} room_t;

typedef struct {
    int           id;               /* duy nhất toàn hệ thống, bắt đầu từ 1 */
    int           room_id;
    char          name[MAX_NAME + 1];
    char          desc[MAX_DESC + 1];
    money_t       start_price;
    money_t       buynow_price;     /* 0 = không cho mua ngay */
    int           duration_sec;
    item_status_t status;
    money_t       price;            /* nghĩa theo status, xem bảng mục 6 PROTOCOL.md */
    char          leader[MAX_USERNAME + 1];  /* "" = chưa có ai, gửi đi thành "-" */
    sold_reason_t reason;
    time_t        ended_at;         /* 0 nếu chưa kết thúc */
    time_t        deadline;         /* RAM: chỉ có nghĩa khi ACTIVE */
    int           warned;           /* RAM: đã gửi EVT_TIME_WARNING chưa */
} item_t;

typedef struct {
    int     fd;                     /* -1 = ô trống */
    char    ip[INET6_ADDRSTRLEN];   /* để ghi log */
    int     port;
    char    in[MAX_LINE];           /* recv buffer, xem framing PROTOCOL.md mục 1 */
    size_t  in_len;
    char   *out;                    /* send buffer, cấp phát động */
    size_t  out_len, out_cap;
    char    username[MAX_USERNAME + 1];  /* "" = chưa đăng nhập */
    int     room_id;                /* 0 = không ở phòng nào */
} client_t;

/* Toàn bộ trạng thái server */
extern user_t   users[MAX_USERS];   extern int n_users;
extern room_t   rooms[MAX_ROOMS];   extern int n_rooms;
extern item_t   items[MAX_ITEMS];   extern int n_items;
extern client_t clients[MAX_CLIENTS];
extern int next_room_id, next_item_id;
```

Ghi chú:

- **Thành viên phòng không lưu riêng.** Thành viên của phòng R là các client có
  `room_id == R`. Đếm `member_count` hay broadcast đều duyệt mảng `clients`.
  Như vậy không bao giờ có hai danh sách lệch nhau.
- **Send buffer cấp phát động** vì một response nhiều dòng (`OK_ITEMS` của phòng
  có nhiều món) có thể dài hơn 4096 byte. Nếu `out_len` vượt `OUT_BUF_LIMIT`,
  tức là client không chịu đọc, server đóng kết nối đó và ghi log.
- **Tìm kiếm** duyệt tuần tự các mảng. Với vài nghìn phần tử thì đủ nhanh.
- **Thứ tự hàng đợi** của phòng là các item cùng `room_id` theo `id` tăng dần.
  Món kế tiếp là món `PENDING` có `id` nhỏ nhất.

---

## 3. Đối chiếu field protocol ↔ struct

| Trong protocol | Trong struct | Chuyển đổi |
|---|---|---|
| `<status>` của phòng | `room_t.status` | `WAITING` / `RUNNING` / `FINISHED` ↔ enum |
| `<status>` của vật phẩm | `item_t.status` | `PENDING` / `ACTIVE` / `SOLD` / `UNSOLD` ↔ enum |
| `<leader>`, `<winner>` | `item_t.leader` | chuỗi rỗng ↔ `-` |
| `<reason>` | `item_t.reason` | `AUCTION` / `BUYNOW` ↔ enum, `REASON_NONE` ↔ `-` |
| `<seconds_left>` | `item_t.deadline` | `max(0, ceil(deadline - now))` |
| `<current_price>`, `<price>` | `item_t.price` | giữ nguyên |
| `<member_count>` | không lưu | đếm client có `room_id` đó |
| `<item_count>` | không lưu | đếm item có `room_id` đó |
| `<ended_at>` | `item_t.ended_at` | Unix timestamp |

---

## 4. Thư mục dữ liệu và các file

Server chạy bằng `./server <port> <data_dir>`, ví dụ `./server 5500 ./data`.
Thiếu file nào thì tạo file rỗng.

```
data/
├── meta.txt      bộ đếm id
├── users.txt     tài khoản
├── rooms.txt     phòng
├── items.txt     vật phẩm
├── bids.txt      mọi lượt trả giá và mua ngay hợp lệ (chỉ ghi nối tiếp)
└── server.log    log hoạt động (mục 8)
```

**Format chung:** mỗi bản ghi một dòng, các field nối bằng `|`, kết thúc bằng
`\n`, escape giống hệt `PROTOCOL.md` mục 1 (`\\`, `\p`, `\n`). Dòng bắt đầu bằng
`#` là chú thích, bỏ qua. Dùng chung hàm tách và escape với phần protocol.

### meta.txt

```
next_room_id|next_item_id
```
```
4|17
```

Bộ đếm id chỉ tăng, không bao giờ dùng lại. Nếu lấy `max(id) + 1` thì xóa món có
id lớn nhất xong, món tạo sau sẽ trùng id với món đã xóa. Client nào còn giữ id
cũ sẽ nhầm sang món mới.

### users.txt

```
username|pass_hash|created_at
```
```
binh|$6$Zq3c9x8LkP1aB2cD$Jd0...(86 ký tự)|1760773200
```

`pass_hash` do `crypt()` tạo ra, chỉ gồm `[a-zA-Z0-9./$]` nên không bao giờ chứa `|`.

### rooms.txt

```
room_id|name|owner|status|created_at
```
```
1|Do co Ha Noi|an|FINISHED|1760773500
```

### items.txt

```
item_id|room_id|name|desc|start_price|buynow_price|duration_sec|status|price|leader|reason|ended_at
```
```
1|1|Binh gom|Gom Bat Trang|500000|3000000|120|SOLD|850000|binh|AUCTION|1760773980
2|1|Dong ho cu|Dong ho co|200000|1000000|120|SOLD|1000000|dung|BUYNOW|1760774010
3|2|Tranh son mai||300000|0|300|PENDING|300000|-|-|0
```

### bids.txt

```
ts|item_id|username|amount|kind
```
```
1760773870|1|binh|510000|BID
1760773905|1|chi|550000|BID
1760773968|1|binh|850000|BID
1760774010|2|dung|1000000|BUYNOW
```

`kind` là `BID` hoặc `BUYNOW`. Chỉ ghi các lượt **hợp lệ**, tức là đã nhận
`OK_BID` hoặc `OK_BUY_NOW`.

---

## 5. Khi nào ghi file

| Sự kiện | Ghi gì | Cách ghi |
|---|---|---|
| `REGISTER` thành công | `users.txt` | ghi lại cả file |
| `CREATE_ROOM` | `rooms.txt`, `meta.txt` | ghi lại cả file |
| `ADD_ITEM` | `items.txt`, `meta.txt` | ghi lại cả file |
| `DEL_ITEM` | `items.txt` | ghi lại cả file |
| `START_AUCTION`, phòng chuyển `FINISHED` | `rooms.txt` | ghi lại cả file |
| Món lên sàn, được bán, hết giờ không ai mua | `items.txt` | ghi lại cả file |
| Lượt `BID` / `BUY_NOW` hợp lệ | `bids.txt` | ghi nối tiếp một dòng, rồi `fflush()` |

- **Không ghi `items.txt` sau mỗi lượt trả giá.** Giá hiện tại và người dẫn của
  món `ACTIVE` chỉ cần nằm trong RAM. Nếu server sập giữa chừng thì món đó chuyển
  `UNSOLD` (mục 6), nên không cần khôi phục giá đang đấu.
- **Ghi lại cả file một cách an toàn:** ghi ra `items.txt.tmp`, gọi `fflush()` và
  `fclose()`, rồi `rename("items.txt.tmp", "items.txt")`. Trên Linux `rename()`
  là thao tác nguyên tử, nên nếu server sập giữa chừng thì vẫn còn nguyên file
  cũ, không bị hỏng một nửa.
- Ghi file lỗi (đầy ổ đĩa, không có quyền ghi): ghi `[ERROR]` vào log và vẫn trả
  `OK` cho client. Dữ liệu trong RAM vẫn đúng.

---

## 6. Khi server khởi động

1. Đọc `meta.txt`, `users.txt`, `rooms.txt`, `items.txt` vào RAM. Dòng nào sai
   format thì bỏ qua và ghi `[WARN]` kèm số dòng vào log.
2. **Dọn phiên đấu dở (PROTOCOL.md mục 11):**
   - phòng đang `RUNNING` chuyển `FINISHED`;
   - mọi món `ACTIVE` hoặc `PENDING` của phòng đó chuyển `UNSOLD`, với
     `price = start_price`, `leader = ""`, `ended_at = now`.

   Phải chuyển cả món `PENDING`, nếu không phòng đã `FINISHED` vẫn còn món đang
   chờ (bất biến số 3 ở mục 9).
3. Nếu bước 2 có thay đổi thì ghi lại `rooms.txt` và `items.txt`, và ghi log.
4. Mọi user khởi đầu ở trạng thái offline, mọi `room.active_item = 0`.

---

## 7. Mật khẩu (D5)

Dùng `crypt()` của libxcrypt, có sẵn trên Ubuntu 24.04. Biên dịch kèm `-lcrypt`.
File header `crypt.h` nằm trong gói `libcrypt-dev`, thường đã được cài cùng
`build-essential`.

```c
#include <crypt.h>

/* REGISTER: tạo chuỗi băm mới */
const char *salt = crypt_gensalt("$6$", 0, NULL, 0);   /* NULL: thư viện tự lấy byte ngẫu nhiên */
const char *h = crypt(password, salt);                  /* "$6$<salt>$<hash>" */
/* h == NULL hoặc h[0] == '*' là lỗi -> ERR|E_INTERNAL */
strcpy(user->pass_hash, h);

/* LOGIN: băm lại mật khẩu nhập vào với chính chuỗi đã lưu */
const char *h2 = crypt(password, user->pass_hash);
int ok = (h2 != NULL && strcmp(h2, user->pass_hash) == 0);
```

- **`crypt()` và `crypt_gensalt()` trả về con trỏ tới vùng nhớ tĩnh.** Lần gọi
  sau ghi đè kết quả của lần gọi trước. Vì vậy phải so sánh hoặc chép
  (`strcpy`) kết quả **ngay** sau mỗi lần gọi, không giữ con trỏ lại để dùng sau.
  Đã thử trên Ubuntu 24.04: giữ hai con trỏ rồi mới so sánh thì mật khẩu đúng
  cũng báo sai.
- Chuỗi đã lưu chứa luôn salt, nên không cần lưu salt riêng. Chuỗi SHA-512 dài
  106 ký tự, vừa với `pass_hash[128]`.
- **Sai username hay sai mật khẩu đều trả cùng một lỗi** `E_CREDENTIAL`, để không
  lộ việc username đó có tồn tại hay không.
- **Không bao giờ ghi mật khẩu vào log.**
- Biến chứa mật khẩu nên xóa đi (`memset`) sau khi dùng xong.

---

## 8. Log hoạt động (`server.log`)

Thang điểm có 1 điểm cho log. Mỗi dòng một sự kiện, ghi nối tiếp và `fflush()`
ngay, để server sập vẫn còn log.

```
<YYYY-MM-DD HH:MM:SS> [<LEVEL>] <nguồn> <nội dung>
```

- `LEVEL`: `INFO`, `WARN`, `ERROR`.
- `nguồn`: `fd=5 127.0.0.1:51234 user=binh` cho việc của client, `server` cho
  việc của server, `room=1 item=1` cho việc của phiên đấu.

```
2026-10-27 09:00:01 [INFO] server start port=5500 data=./data users=12 rooms=3 items=17
2026-10-27 09:01:10 [INFO] fd=5 127.0.0.1:51234 connected
2026-10-27 09:01:12 [INFO] fd=5 127.0.0.1:51234 user=- LOGIN binh -> OK
2026-10-27 09:02:40 [INFO] fd=5 127.0.0.1:51234 user=binh BID item=1 amount=505000 -> E_BID_TOO_LOW
2026-10-27 09:02:44 [INFO] fd=5 127.0.0.1:51234 user=binh BID item=1 amount=510000 -> OK
2026-10-27 09:03:30 [INFO] room=1 item=1 warning seconds_left=30
2026-10-27 09:03:48 [INFO] room=1 item=1 extended seconds_left=30 by=binh
2026-10-27 09:04:18 [INFO] room=1 item=1 sold price=850000 winner=binh reason=AUCTION
2026-10-27 09:06:02 [INFO] fd=5 127.0.0.1:51234 user=binh disconnected
2026-10-27 09:10:00 [ERROR] server write items.txt failed: No space left on device
```

**Phải ghi:** server bật và tắt; client kết nối và ngắt kết nối; **mọi request**
kèm mã kết quả (`OK` hoặc mã lỗi); mọi sự kiện của phiên đấu (bắt đầu, cảnh báo,
gia hạn, bán, không bán, kết thúc); lỗi đọc ghi file; client bị đóng vì send
buffer quá lớn.

**Không ghi:** mật khẩu, nội dung chat.

---

## 9. Bất biến (luôn phải đúng)

Dùng để tự kiểm tra khi viết code và viết test. Có thể viết một hàm
`check_invariants()` chạy ở chế độ debug sau mỗi request.

1. Phòng `RUNNING` có **đúng một** món `ACTIVE` và `room.active_item` là id của món đó.
2. Phòng `WAITING` hoặc `FINISHED` không có món `ACTIVE`.
3. Phòng `FINISHED` không còn món `PENDING`.
4. Món `SOLD` có `leader` khác rỗng và `reason` khác `REASON_NONE`.
5. Món `UNSOLD` có `leader` rỗng.
6. Món `ACTIVE` có `price ≥ start_price`. Nếu `leader` rỗng thì `price == start_price`.
7. Nếu `buynow_price > 0` thì `buynow_price ≥ start_price + MIN_INCREMENT`.
8. Một username chỉ online trên **tối đa một** client.
9. `client.room_id != 0` thì `client.username` khác rỗng.
10. Chủ phòng không bao giờ là `leader` của món trong phòng mình.

---

## 10. Dữ liệu phía client (Python)

Client không lưu gì xuống đĩa. Chỉ giữ trong RAM những gì server gửi về.
Cú pháp `str | None` bên dưới cần Python 3.10 trở lên.

```python
from dataclasses import dataclass
import time

@dataclass
class RoomInfo:            # một dòng ROOM|...
    room_id: int
    name: str
    owner: str
    status: str            # WAITING / RUNNING / FINISHED
    member_count: int
    item_count: int

@dataclass
class ItemOnFloor:         # từ EVT_ITEM_START, cập nhật bằng EVT_NEW_BID
    room_id: int
    item_id: int
    name: str
    desc: str
    start_price: int
    buynow_price: int      # 0 = không có
    current_price: int
    leader: str | None     # "-" trong protocol thành None
    deadline: float        # time.monotonic() + seconds_left

    def seconds_left(self) -> int:
        return max(0, round(self.deadline - time.monotonic()))
```

- Dùng `time.monotonic()` thay cho `time.time()`. Đồng hồ monotonic không bị nhảy
  khi người dùng hoặc hệ điều hành chỉnh giờ máy.
- Mỗi lần nhận `EVT_NEW_BID` hoặc `EVT_TIME_EXTENDED` thì tính lại
  `deadline = time.monotonic() + seconds_left`.
- Giá trong Python là `int`, không giới hạn độ lớn nên không lo tràn số.
- Hiển thị giá có dấu chấm ngăn hàng nghìn kiểu Việt Nam (`850.000`), nhưng
  **gửi đi** thì là số trơn (`850000`).

---

## 11. Lịch sử thay đổi

### v1.0.1 — 18/10/2026
Mục 7: thêm lưu ý `crypt()` dùng vùng nhớ tĩnh và độ dài chuỗi băm, sau khi
chạy thử trên Ubuntu 24.04.

### v1.0 — 18/10/2026
Chốt toàn bộ bản nháp v0.9 (quyết định D1–D5), không thay đổi nội dung.
Đồng thời sửa `PROTOCOL.md` lên v1.0.1 để luật khởi động lại khớp mục 6.

### v0.9 — 18/10/2026
Bản nháp đầu tiên.
