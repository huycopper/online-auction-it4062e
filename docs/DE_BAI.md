# Đề bài và tiêu chí chấm: Topic 3, Online Auction

Chép nguyên văn từ file đề thầy phát:
`Class_Materials/Topic_202601_Network_Programming_English.xlsx`, sheet 1, dòng 4
(No 3, *Build an online auction application*).

| Cột trong file | Giá trị |
|---|---|
| Maximum points | **27–35** |
| Maximum groups | 2 (tối đa 2 nhóm được chọn đề này) |

---

## 1. Mô tả đề (cột Description)

> Each auction participant can perform the following functions:
>
> User Management: Manage user accounts and create auction rooms.
> Auction Participation Management: Join one auction room at a time.
> Item Selling: Provide item info, starting price, auction time, and "Buy Now" price for auctioned items.
> View Auction Items: See past, current, and upcoming auction items in each room.
> Search Auction Items: Search by time slot or item info, join rooms, and participate in ongoing auctions.
> Auction History: View statistics and results of past auctions participated in.
> Queue Management: Manage auction items in a queue, introduced one by one during the auction session.
> Auction Process: Validate bids (minimum 10,000 VND higher than the current price), and announce the new price if valid.
> Time Alerts: Notify participants when 30 seconds remain, and reset the timer if a new bid is made in the last 30 seconds.
> Buy Now Option: If a user chooses "Buy Now," the item is sold immediately without further auction.
> Advanced Features: Additional functionalities can be added.
> Graphical User Interface (GUI): Provide an intuitive and user-friendly interface.

## 2. Tiêu chí chấm (cột Features and points)

Cột "Tiêu chí" chép nguyên văn từ file. Cột "Nghĩa là" do nhóm tự giải thích. Message
tương ứng của từng tiêu chí xem ở `PROTOCOL.md` mục 13.

| # | Tiêu chí | Điểm | Nghĩa là |
|---|---|---|---|
| 1 | Stream handling | 1 | Tự tách message từ dòng byte TCP |
| 2 | Implementing socket I/O mechanism on the server | 2 | Server phục vụ nhiều client cùng lúc |
| 3 | Account registration and management | 2 | Đăng ký và quản lý tài khoản |
| 4 | Login and session management | 2 | Đăng nhập, đăng xuất, quản lý phiên |
| 5 | Access control for auction rooms | 1 | Kiểm soát ai được làm gì trong phòng |
| 6 | Creating auction rooms | 1 | Tạo phòng đấu giá |
| 7 | Listing auction rooms | 1 | Liệt kê phòng đấu giá |
| 8 | Creating auction items | 2 | Tạo vật phẩm đấu giá |
| 9 | Deleting auction items in an auction room | 1 | Xóa vật phẩm trong phòng |
| 10 | Joining auction rooms | 2 | Vào phòng đấu giá |
| 11 | Bidding | 2 | Trả giá |
| 12 | Direct purchase ("Buy Now") | 1 | Mua ngay |
| 13 | Logging activities | 1 | Ghi log hoạt động |
| 14 | Sending notifications when time is nearly up + resetting auction time | 2 | Cảnh báo khi còn 30 giây và đặt lại đồng hồ |
| 15 | Advanced features | 2–10 | Chức năng nâng cao, nhóm tự chọn |
| 16 | Graphical User Interface (GUI) | 3 | Giao diện đồ họa |

**Cộng lại:** 15 tiêu chí cố định được 24 điểm. Thêm phần nâng cao 2–10 điểm thì tổng
là 26–34 điểm.

---

## 3. Những điểm cần để ý

1. **Tổng điểm lệch 1 so với cột "Maximum points".** Các tiêu chí cộng lại được
   26–34, nhưng file ghi 27–35. Cách tính này đúng với đề chat (dòng 2: 23 điểm cố
   định cộng 2–5 điểm nâng cao, ra đúng 25–28 như file ghi), nên đề đấu giá có thể bị
   ghi nhầm. Có thể hỏi thầy.
2. **Đề không ghi phần nâng cao gồm những gì.** Cả hai cột chỉ có câu chung chung.
3. **Có 3 chức năng trong mô tả nhưng không có tiêu chí riêng:** View Auction Items,
   Search Auction Items, Auction History. Nhóm đang xếp search và history vào phần
   nâng cao (slide 3).
4. **Có 2 chỗ mơ hồ:** "search by time slot" và "account management". Các phương án
   hiểu đề xem ở [`checkpoint1/tham-khao-repo-cu.md`](checkpoint1/tham-khao-repo-cu.md)
   mục 3.3.

## 4. Luật bắt buộc của môn (Lec00)

1. Server viết bằng **C, chạy trên Linux**.
2. Client được tự chọn ngôn ngữ.
3. Giao thức tự xây trên **raw socket**. Cấm thư viện hoặc nền tảng socket có sẵn
   (Socket.io, WebSocket…).
4. Cấm dùng "proxy server" để lách luật 3.

Cách nhóm áp các luật này vào repo xem ở `README.md` mục 2.
