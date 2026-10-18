# Tham khảo repo của các khóa trước

Rà soát ngày 18/10/2026, trước Checkpoint 1. Đây là các repo công khai trên GitHub
của những nhóm HUST đã làm cùng đề Online Auction ở các kỳ trước.

> **Quy tắc:** chỉ xem để hiểu đề và lấy ý tưởng. **Không chép code, không mở code
> của họ khi đang viết code của mình.** Thầy có kiểm tra trùng code và hỏi vấn đáp.

---

## 1. Các repo đã xem

| Repo | Kỳ | Server | Client | Lưu trữ |
|---|---|---|---|---|
| [mtrung12/auction_app](https://github.com/mtrung12/auction_app) | 20251 | C, mỗi client một thread | Qt C++, dùng `QTcpSocket` | PostgreSQL |
| [nghiaphunng18/online-auction-system-network-programming-20251](https://github.com/nghiaphunng18/online-auction-system-network-programming-20251) | 20251 | **C++**, `select()` một luồng | Qt, dùng `QTcpSocket` | file text |
| [JosephD0310/AuctionApp_NetworkProgramming](https://github.com/JosephD0310/AuctionApp_NetworkProgramming) | 20241 | C, `select()` và một thread đếm giờ | GTK, socket thô | file text |
| [minhpham2409/Auction](https://github.com/minhpham2409/Auction) | cuối 2025 | C, mỗi client một thread | console | file |
| [kiet-dt/auction](https://github.com/kiet-dt/auction) | cuối 2025 | C, `select()` | console | file |
| [dtdng/OnlineAuctionAppSocket](https://github.com/dtdng/OnlineAuctionAppSocket) | 20231 | C, mỗi client một process (`fork()`) | console | SQLite |

## 2. Đối chiếu với thang điểm và luật Lec00

✓ có · ~ có nhưng lệch yêu cầu · ✗ không có · – chưa kiểm tra

| | mtrung12 | nghiaphunng18 | JosephD0310 | minhpham2409 | kiet-dt | dtdng |
|---|---|---|---|---|---|---|
| Server viết bằng C | ✓ | ✗ C++ | ✓ | ✓ | ✓ | ✓ |
| Client không dùng thư viện socket có sẵn | ✗ `QTcpSocket` | ✗ `QTcpSocket` | ✓ | ✓ | ✓ | ✓ |
| Stream handling | ✓ header nhị phân, có độ dài | ✓ cắt dòng | ✗ gọi `recv` một lần | – | ✓ cắt dòng | ✗ |
| Tài khoản, đăng nhập | ✓ | ✓ đá phiên cũ | ✓ | ✓ | ✓ | ✓ |
| Phòng, vật phẩm, mua ngay | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| Trả giá cao hơn ít nhất 10.000 | ✓ | ✓ | ✗ | ~ bước giá tự chọn | ✓ | – |
| Cảnh báo 30 giây và đặt lại đồng hồ | ✓ | ✓ | ~ không cảnh báo | ✓ | ✗ | ✗ |
| Ghi log | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| Tìm theo khung giờ | ✓ | ✗ chỉ theo từ khóa | ✗ | ✗ | ✗ | ~ xem lịch phiên |
| Lịch sử | ✓ | ✓ | ~ chỉ đồ đã mua | ✓ | ✗ | ✗ |
| Nâng cao | chat, ví tiền ảo, phòng có PIN | chat, auto-bid, bản nháp | – | – | – | – |
| GUI | Qt | Qt | GTK | ✗ | ✗ | ✗ |

**Kết luận:** không repo nào vừa đủ chức năng vừa đúng luật.

- **mtrung12 gần đề nhất.** Repo có file `yêu cầu project.txt` chứa đúng đề Topic 3
  bằng tiếng Việt. Nhưng client của họ dùng `QTcpSocket`.
- **nghiaphunng18 giống thiết kế của nhóm mình nhất:** giao thức dạng dòng text,
  `select()` một luồng, event gia hạn thời gian, đá phiên cũ khi đăng nhập nơi khác.
  Nhưng server của họ viết bằng C++.
- **JosephD0310 là repo duy nhất có GUI mà vẫn đúng luật,** nhưng phần luật đấu giá
  còn yếu.

Mình không biết các nhóm này được bao nhiêu điểm.

---

## 3. Insight cho nhóm

### 3.1. Thầy đã chấm đề này nhiều lần, nên phải làm rõ nhóm mình khác ở đâu

Điểm khác biệt của nhóm:

- **Đúng luật ở cả hai phía:** server viết bằng C, client Python chỉ dùng module
  `socket` chuẩn. Hai repo đầy đủ nhất đều vi phạm đúng điểm này.
- **Đồng hồ nằm trong `select()`.** Timeout của `select()` bằng thời gian tới deadline
  gần nhất. Các nhóm trước dùng thread riêng chạy `sleep(1)` để đếm giờ, dễ sinh
  tranh chấp dữ liệu.
- **Giao thức có tài liệu đầy đủ** (`PROTOCOL.md`), code chia module và mỗi module có
  người phụ trách. Nhiều nhóm trước dồn cả server vào một file 900–1.800 dòng.

### 3.2. Lỗi phổ biến của các khóa trước là câu thầy dễ hỏi

| Lỗi thấy ở repo cũ | Câu thầy có thể hỏi | Nhóm trả lời thế nào |
|---|---|---|
| Gọi `recv` một lần, không cắt message | "recv nhận nửa message thì sao?" | Mỗi client có buffer riêng, cắt tại `\n` (`PROTOCOL.md` mục 1) |
| Mỗi client một thread, không khóa dữ liệu | "Hai người trả giá cùng lúc thì sao?" | Server một luồng, xử lý lần lượt, không cần khóa |
| Thread đếm giờ bằng `sleep(1)` | "Chỉ một luồng thì đồng hồ chạy thế nào?" | Timeout của `select()` bằng thời gian tới deadline gần nhất |
| Lưu tiền bằng `double` | "Tiền lưu kiểu gì?" | `long long` (`money_t`) đếm theo đồng, không bị sai số làm tròn |

### 3.3. Đề có 3 chỗ mơ hồ, mỗi nhóm hiểu một kiểu, nên cần hỏi thầy

1. **Search by time slot** có 4 cách hiểu:
   - A. Phòng có giờ bắt đầu dự kiến, tìm phòng theo khoảng giờ (phương án A, P14).
   - B. Tìm theo thời điểm món được bán, tức `ended_at` (phương án B, P14).
   - C. Tìm theo thời điểm món được tạo (mtrung12).
   - D. Xem lịch phiên của từng phòng (dtdng).
2. **Account management:** chỉ đăng ký và đăng nhập là đủ chưa? mtrung12 làm thêm nạp
   và rút tiền ảo, sửa thông tin cá nhân.
3. **Tìm kiếm và lịch sử:** đề yêu cầu nhưng thang điểm không có dòng riêng. Hai chức
   năng này được tính vào phần nâng cao, hay là phần bắt buộc?

Đề không ghi phần nâng cao gồm những gì. Cả cột mô tả lẫn cột điểm chỉ có câu chung:
*"Additional functionalities can be added"* (2–10 điểm).

### 3.4. Không hứa quá nhiều phần nâng cao

Các nhóm trước làm chat, auto-bid, ví tiền ảo, phòng có PIN, bản nháp vật phẩm.
Thường mỗi nhóm chỉ làm tốt 2–3 chức năng. Slide 3 hiện ghi 4 chức năng (search,
history, chat, auto-bid), như vậy là vừa. Làm xong phần lõi rồi mới làm phần nâng cao.

### 3.5. Nói về các repo cũ thế nào

Không nhắc đến chúng trong bài thuyết trình. Nếu thầy hỏi có tham khảo không thì trả
lời thật: nhóm có xem để hiểu đề, nhưng tự thiết kế giao thức và tự viết code.

---

## 4. Việc nên làm trước Checkpoint 1

- [ ] Slide 11: thêm lại mục "Open questions" với 3 câu ở mục 3.3, mỗi câu kèm các
      phương án để thầy chọn. Sửa kịch bản cho khớp.
- [ ] Kịch bản hỏi đáp: thêm câu "How is your design different?" (Huy trả lời) và
      "How do you store money?" (Minh trả lời).
- [ ] Khi nói slide 4 và 5, nhấn mạnh 3 điểm khác biệt ở mục 3.1.
