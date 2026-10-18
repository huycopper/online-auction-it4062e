# Phân công và cách phối hợp

Nhóm trưởng chốt ngày 19/10/2026. So với bản trước, `item.c` (thêm, xóa, xem vật phẩm)
và chat chuyển từ Minh sang Hòa. Ai sửa file nào xem bảng ở `README.md` mục 3.

---

## 1. Ai làm gì

| Người | Phụ trách | Dòng điểm lõi (`DE_BAI.md` mục 2) | Nâng cao |
|---|---|---|---|
| **Huy** | Server lõi: `main.c`, `net.c`, `proto.c`, `dispatch.c`, `auth.c`, `storage.c`, `log.c`, `search.c`. Ghép code, review PR, giữ tài liệu | Stream handling 1, socket I/O 2, tài khoản 2, đăng nhập và phiên 2, log 1 = **8** | Tìm kiếm, lịch sử (phía server) |
| **Minh** | Server nghiệp vụ: `room.c` (tạo, liệt kê, vào, rời phòng, quyền trong phòng), `auction.c` (hàng đợi, trả giá, mua ngay, đồng hồ 30 giây) | Kiểm soát truy cập 1, tạo phòng 1, liệt kê phòng 1, vào phòng 2, trả giá 2, mua ngay 1, cảnh báo 30 giây 2 = **10** | Auto-bid |
| **Hòa** | Toàn bộ client PySide6. Phía server: `item.c` và chat | Tạo vật phẩm 2, xóa vật phẩm 1, GUI 3 = **6** | Chat, màn hình tìm kiếm và lịch sử, biểu đồ |

Cộng lại 8 + 10 + 6 = 24 điểm lõi. **Điểm không phải thước đo khối lượng việc:** dòng
GUI chỉ có 3 điểm, nhưng chức năng nào cũng cần màn hình của Hòa mới demo được.

## 2. Vì sao chia như vậy

1. **Chia theo tầng để ba người làm song song.** Mỗi người làm chủ yếu một ngôn ngữ và
   một nhóm file. Chỗ nối giữa các phần đã có tài liệu: `PROTOCOL.md` giữa client và
   server, `model.h` giữa các module server.
2. **`item.c` tách khỏi `auction.c` theo trạng thái phòng** (`PROTOCOL.md` mục 6). Ở
   mỗi giai đoạn chỉ một người được ghi dữ liệu vật phẩm:

   | Trạng thái phòng | Ai ghi dữ liệu vật phẩm | Ghi gì |
   |---|---|---|
   | `WAITING` | Hòa (`item.c`) | thêm, xóa món |
   | `RUNNING` | Minh (`auction.c`) | giá, người dẫn, trạng thái món |
   | `FINISHED` | không ai | chỉ đọc: lịch sử, kết quả |

3. **Không ai giữ quá nhiều dòng điểm một mình.** Trước đây Minh giữ 13/24 điểm lõi
   cùng `auction.c`, file khó nhất dự án. Bớt `item.c` thì Minh dồn sức vào `auction.c`.
4. **Ai cũng có code C ở server để trả lời khi thầy hỏi.** `item.c` là module C dễ
   nhất: nhận lệnh rồi trả lời, không có đồng hồ, không phát event. Hòa làm trọn chức
   năng vật phẩm, từ form đến server.
5. **Huy nặng việc nhất.** Phần của Huy là nền móng trên đường găng, và người review
   mọi PR cần hiểu cả hệ thống.

---

## 3. Thứ tự làm việc

### Ai chờ ai

```
Header của Huy (22/10)
  └─► Khung server của Huy: select(), framing, LOGIN (27/10)
        └─► room.c của Minh
              ├─► item.c của Hòa ──► Minh test auction.c bằng món thật
              └─► chat của Hòa
```

Client không phải chờ server. Hòa chạy `nc -l 5500` làm server giả, cho client nối
vào, rồi tự gõ các dòng response và event lấy từ `PROTOCOL.md` mục 12.

### Theo giai đoạn

| | Đến 27/10 | Đến Checkpoint 2 | Đến cuối kỳ |
|---|---|---|---|
| **Huy** | Header `model.h`, `net.h`, `dispatch.h` (22/10). `select()`, framing, `REGISTER`, `LOGIN` | Lưu file, log, `LOGOUT`, mất kết nối | `search.c`, ghép code, báo cáo |
| **Minh** | `room.h` trước, rồi `CREATE_ROOM`, `LIST_ROOMS`, `JOIN_ROOM` | `auction.c`: `START_AUCTION`, `BID`, `BUY_NOW`, đồng hồ 30 giây | Auto-bid, chỉ làm khi engine đã chạy ổn |
| **Hòa** | Khung màn hình, luồng mạng, đăng nhập thật | `item.c` trước, rồi màn hình phòng đấu giá, chạy trọn một phiên | Chat, màn hình tìm kiếm và lịch sử, hoàn thiện GUI |

## 4. Bàn giao

| # | Ai giao cho ai | Giao gì | Khi nào | Trong lúc chờ |
|---|---|---|---|---|
| 1 | Huy → cả nhóm | Header `model.h`, `net.h`, `dispatch.h` | 22/10 | — |
| 2 | Minh → Hòa | `room.h`: tìm phòng, kiểm tra chủ phòng và trạng thái phòng, duyệt thành viên. `item.c` và chat gọi các hàm này | Tuần đầu | Hòa làm client |
| 3 | Hòa → Minh | `ADD_ITEM` chạy được | Đầu giai đoạn 2 | Minh tạo tạm vài món bằng một hàm chỉ dùng khi test |
| 4 | Server → client | Lệnh nào merge vào `main` thì Hòa nối màn hình đó với server thật | Liên tục | Hòa test với server giả bằng `nc -l` |

Khi đã có `ADD_ITEM`, Minh tạo món để test bằng một file lệnh trong `tests/` gửi qua
`nc`. Cách tự ghi món vào `items.txt` chỉ dùng được sau khi Huy xong phần đọc file
(giai đoạn 2).

## 5. Quy tắc phối hợp

1. **Tài liệu là hợp đồng.** Muốn đổi lệnh hay field thì tạo PR chỉ sửa `docs/` trước
   (`README.md` mục 8). Chỗ nào chưa rõ thì hỏi trong group chat, không tự hiểu một kiểu.
2. **Mỗi file một người phụ trách.** File chung duy nhất là bảng lệnh trong
   `dispatch.c`: mỗi PR chỉ thêm các dòng cho lệnh của mình.
3. **PR nhỏ, merge sớm:** mỗi lệnh hoặc mỗi màn hình một PR. Huy review trong vòng một
   ngày để không ai bị kẹt.
4. **Mỗi tuần ghép thử một lần, vào cuối tuần.** Chạy server bản `main` với client thật,
   đi theo kịch bản `PROTOCOL.md` mục 12, được tới đâu thì tới đó. Chỗ hỏng tạo issue
   trên GitHub, giao cho người phụ trách phần đó.

## 6. Khi nào xem lại

Ở Checkpoint 2, nhìn tiến độ thật rồi chia lại phần nâng cao: ai xong phần lõi trước thì
nhận thêm. Nếu lúc đó Huy quá tải thì chuyển `search.c` cho người đang đi trước.
