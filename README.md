# Online Auction — Nhóm 05, IT4062E

Ứng dụng đấu giá trực tuyến (Topic 3, Network Programming IT4062E, học kỳ 20261,
lớp 171136). Server viết bằng C trên Linux dùng `select()`, client viết bằng
Python + PySide6. Hai bên nói chuyện bằng giao thức OAP do nhóm tự thiết kế trên
TCP thô.

| Thành viên      | MSSV     | Phụ trách                                                                                                           |
| --------------- | -------- | ------------------------------------------------------------------------------------------------------------------- |
| Đồng Đại Huy    | 20235945 | Server lõi: vòng `select()`, framing, parser, tài khoản, lưu trữ, log, tìm kiếm và lịch sử. Ghép code, giữ tài liệu |
| Hồ Ngọc Minh    | 20235973 | Server nghiệp vụ: phòng và quyền trong phòng, hàng đợi, engine đấu giá và đồng hồ 30 giây                           |
| Nguyễn Việt Hòa | 20235936 | Client PySide6: luồng mạng, các màn hình, đếm ngược. Phía server: vật phẩm (`item.c`) và chat                       |

Lý do chia như vậy, thứ tự làm và cách bàn giao: [`docs/PHAN_CONG.md`](docs/PHAN_CONG.md).

---

## 1. Tài liệu phải đọc trước khi code

| Tài liệu                                   | Nội dung                                                | Ai đọc kỹ phần nào                                                |
| ------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------- |
| [`docs/PROTOCOL.md`](docs/PROTOCOL.md)     | Giao thức OAP v1.0.1: mọi message, mã lỗi, luật đấu giá | Huy: mục 1–4 · Minh: mục 5–7 · Hòa: mục 2, 6, 8 · **Cả nhóm: mục 0** |
| [`docs/DATA_MODEL.md`](docs/DATA_MODEL.md) | Struct, file lưu trữ, mật khẩu, log, bất biến           | Huy: toàn bộ · Minh: mục 2–3, 9 · Hòa: mục 10                     |
| [`docs/KIEN_THUC_NEN.md`](docs/KIEN_THUC_NEN.md) | Giải thích nghiệp vụ đấu giá, socket TCP, framing, giao thức, `select()`, hai luồng ở client | Cả nhóm, đọc trước khi đọc hai tài liệu trên |
| [`docs/KIEN_TRUC.md`](docs/KIEN_TRUC.md) | Sơ đồ kiến trúc, đường đi của một lượt BID, khối nào của ai | Cả nhóm |
| File này                                   | Cấu trúc repo, cách build, quy ước code, quy trình Git  | Cả nhóm                                                           |

**Hai tài liệu trong `docs/` là hợp đồng giữa các phần.** Code làm khác tài liệu
là code sai. Muốn đổi thì sửa tài liệu trước, theo quy tắc ở mục 0 của
`PROTOCOL.md`.

---

## 2. Yêu cầu bắt buộc của môn (Lec00)

1. Server viết bằng **C, chạy trên Linux**.
2. Client được tự chọn ngôn ngữ (nhóm chọn Python + PySide6).
3. Giao thức tự xây trên **raw socket**. Cấm thư viện hoặc nền tảng socket có sẵn.
4. Cấm dùng "proxy server" để lách luật 3.

Áp vào repo này:

- **Server:** chỉ dùng socket POSIX (`socket`, `bind`, `listen`, `accept`,
  `recv`, `send`, `select`). Thư viện ngoài socket duy nhất là `libcrypt` để băm
  mật khẩu (sẽ hỏi thầy xác nhận).
- **Client:** phần mạng chỉ dùng module `socket` chuẩn của Python và `QThread`.
  **Không** dùng `QTcpSocket`, `QWebSocket`, `websockets`, `socketio`, `requests`.

---

## 3. Cấu trúc thư mục

```
Project_OnlineAuction/
├── README.md                 file này
├── docs/
│   ├── PROTOCOL.md           giao thức OAP
│   ├── DATA_MODEL.md         dữ liệu và lưu trữ
│   ├── DE_BAI.md             đề bài và tiêu chí chấm
│   ├── PHAN_CONG.md          phân công, thứ tự làm, bàn giao
│   ├── KIEN_THUC_NEN.md      giải thích kiến thức nền
│   ├── KIEN_TRUC.md          sơ đồ kiến trúc
│   ├── img/                  hình dùng trong tài liệu
│   ├── checkpoint1/          slide và ghi chú Checkpoint 1
│   └── Group_05.docx         phiếu đăng ký nhóm
├── server/                   C, Linux
│   ├── Makefile
│   ├── include/              file .h
│   └── src/                  file .c
├── client/                   Python + PySide6
│   ├── requirements.txt
│   └── auction_client/       package của client
├── tests/                    kịch bản test bằng nc
└── data/                     dữ liệu khi chạy thử, không commit (đã có trong .gitignore)
```

### Ai sửa file nào (dự kiến)

Mỗi file có **một người chịu trách nhiệm**. Người khác muốn sửa thì báo người đó
trước, để hai người không sửa cùng một file một lúc.

| File                     | Nội dung                                               | Người phụ trách   |
| ------------------------ | ------------------------------------------------------ | ----------------- |
| `server/include/model.h` | struct và hằng số theo `DATA_MODEL.md`                 | Huy (Minh review) |
| `server/src/main.c`      | đọc tham số, khởi động, tắt server                     | Huy               |
| `server/src/net.c`       | vòng `select()`, accept, recv/send buffer, broadcast   | Huy               |
| `server/src/proto.c`     | cắt dòng, tách field, escape, ghép message             | Huy               |
| `server/src/dispatch.c`  | bảng lệnh → hàm xử lý, kiểm tra lỗi chung              | Huy               |
| `server/src/auth.c`      | `REGISTER`, `LOGIN`, `LOGOUT`, `crypt()`               | Huy               |
| `server/src/storage.c`   | đọc ghi các file trong `data/`                         | Huy               |
| `server/src/log.c`       | ghi `server.log`                                       | Huy               |
| `server/src/search.c`    | `SEARCH_ITEMS`, `HISTORY`, `ROOM_RESULT`               | Huy               |
| `server/src/room.c`      | `CREATE_ROOM`, `JOIN_ROOM`, `LEAVE_ROOM`, `LIST_ROOMS` | Minh              |
| `server/src/item.c`      | `ADD_ITEM`, `DEL_ITEM`, `LIST_ITEMS`                   | Hòa               |
| `server/src/auction.c`   | `START_AUCTION`, `BID`, `BUY_NOW`, đồng hồ 30 giây     | Minh              |
| `server/src/chat.c`      | `CHAT` (protocol v1.1)                                 | Hòa               |
| `client/auction_client/` | toàn bộ client                                         | Hòa               |
| `tests/`                 | mỗi người viết test cho phần của mình                  | Cả nhóm           |
| `docs/`                  | tài liệu                                               | Huy               |

---

## 4. Cài môi trường

### Server (Ubuntu 24.04, hoặc WSL Ubuntu 24.04 trên Windows)

```bash
sudo apt update
sudo apt install build-essential netcat-openbsd
```

`build-essential` đã có `gcc`, `make` và `libcrypt-dev`.

### Client (Windows, Linux hoặc macOS)

Cần Python 3.10 trở lên.

```bash
cd client
python -m venv .venv
```

Kích hoạt môi trường ảo: trên Windows chạy `.venv\Scripts\activate`, trên Linux
và macOS chạy `source .venv/bin/activate`. Sau đó:

```bash
pip install -r requirements.txt
```

---

## 5. Build và chạy

### Server

```bash
cd server
make                  # build/server
make SANITIZE=1       # build-asan/server, bắt lỗi bộ nhớ khi test
make clean
```

Chạy từ thư mục gốc của repo:

```bash
./server/build/server 5500 ./data
```

Tham số thứ nhất là cổng, thứ hai là thư mục dữ liệu (`DATA_MODEL.md` mục 4).

### Thử server bằng nc (chưa cần client)

```bash
nc 127.0.0.1 5500
```

Gõ từng dòng rồi Enter, ví dụ `REGISTER|binh|1234`, `LOGIN|binh|1234`,
`LIST_ROOMS`. Mở nhiều cửa sổ `nc` để giả làm nhiều người cùng đấu giá.

### Client

```bash
cd client
python -m auction_client --host 127.0.0.1 --port 5500
```

Chạy server trong WSL và client trên Windows vẫn nối được qua `127.0.0.1`, vì
WSL2 tự chuyển tiếp cổng localhost.

---

## 6. Quy ước code C (server)

- **C11 + POSIX**, build bằng `Makefile` ở trên. **0 warning** trước khi tạo PR.
- **Đặt tên:**
    - hàm và biến dùng `snake_case`, hằng số và macro dùng `UPPER_CASE`;
    - kiểu dữ liệu kết thúc bằng `_t` (`room_t`, `money_t`);
    - hàm dùng ngoài file có tiền tố là tên module: `room_join()`, `auction_bid()`, `proto_escape()`;
    - hàm chỉ dùng trong một file thì khai báo `static`;
    - tên hàm và biến bằng tiếng Anh, comment bằng tiếng Việt hoặc tiếng Anh, viết ngắn.
- **Mỗi module là một cặp** `src/x.c` + `include/x.h`. Header có include guard:
    ```c
    #ifndef ROOM_H
    #define ROOM_H
    ...
    #endif
    ```
- **Biến toàn cục** (mảng `users`, `rooms`, `items`, `clients`) chỉ được định
  nghĩa ở một file. Các file khác dùng qua khai báo `extern` trong `model.h`.
- **Kiểm tra mọi giá trị trả về** của system call (`socket`, `bind`, `accept`,
  `recv`, `send`, `fopen`…). Lỗi thì ghi log, không bỏ qua.
- **Không chặn trong vòng `select()`:** không gọi `sleep()`, không gọi `recv()`
  hay `accept()` trên socket mà `select()` chưa báo sẵn sàng.
- **Chuỗi:** kiểm tra độ dài theo `PROTOCOL.md` mục 3 trước khi chép. Dùng
  `snprintf` để ghép chuỗi. Không dùng `gets`, `sprintf`.
- **Tiền** luôn dùng kiểu `money_t` (`long long`), in ra bằng `%lld`.
- **Định dạng code:** thụt lề 4 dấu cách, ngoặc nhọn kiểu K&R, mỗi dòng tối đa
  100 ký tự. Các quy tắc này đã có trong `.editorconfig`.

## 7. Quy ước code Python (client)

- Python 3.10+, theo PEP 8. Hàm và biến dùng `snake_case`, class dùng `PascalCase`, có type hint.
- **Chỉ luồng GUI được sửa widget.** Luồng mạng gửi dữ liệu sang GUI bằng Qt `Signal`.
- **Không chặn luồng GUI:** không gọi `recv()` hay `time.sleep()` trong luồng GUI.
- Escape và tách message đặt chung trong một module `protocol.py`, làm giống hệt phía server.
- Đồng hồ đếm ngược dùng `time.monotonic()` (`DATA_MODEL.md` mục 10).

---

## 8. Quy trình Git

### Nhánh

- **`main` luôn build được và chạy được.** Không ai push thẳng lên `main`.
  Ngoại lệ duy nhất là nhóm trưởng sửa lỗi chính tả trong tài liệu.
- Mỗi việc làm trên một nhánh riêng, đặt tên `<tên>/<việc>`, ví dụ
  `huy/select-loop`, `minh/auction-timer`, `hoa/login-screen`.

### Commit

- Commit nhỏ, mỗi commit làm một việc.
- Message theo dạng `<phạm vi>: <mô tả ngắn>`, viết tiếng Việt cũng được. Phạm vi là
  một trong `server`, `server/<module>`, `client`, `docs`, `tests`, `chore`.
    ```
    server/auction: từ chối BID đến sau deadline
    client: màn hình phòng đấu giá
    docs: PROTOCOL v1.1 thêm CHAT
    ```
- **Không commit:** thư mục `data/`, `build/`, `.venv/`, file mật khẩu thật.

### Pull request

1. Trước khi tạo PR, cập nhật nhánh theo `main` mới nhất:
    ```bash
    git fetch origin
    git rebase origin/main
    ```
2. Build 0 warning, tự test theo mục 9.
3. Tạo PR trên GitHub. Mô tả PR làm gì và đã test thế nào.
4. **Huy review và merge.** PR của Huy thì người phụ trách phần bị ảnh hưởng review.
5. **Muốn đổi protocol hoặc data model** thì tạo một PR riêng chỉ sửa `docs/`.
   PR đó được duyệt rồi mới làm PR code.

---

## 9. Khi nào một việc được coi là xong

1. `make` không có warning. Chạy test với bản `make SANITIZE=1` không báo lỗi.
2. Làm đúng `PROTOCOL.md`: đã thử bằng `nc` cả trường hợp đúng lẫn **mọi mã lỗi**
   được liệt kê cho lệnh đó.
3. Request và sự kiện mới đều được ghi vào log (`DATA_MODEL.md` mục 8).
4. Đổi hành vi thì tài liệu trong `docs/` đã được cập nhật.
5. PR đã được review và merge vào `main`.
