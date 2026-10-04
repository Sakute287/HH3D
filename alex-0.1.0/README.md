# HH3D Auto Tool

Chrome Extension tự động hóa các tác vụ hằng ngày trên website `hoathinh3d.*`.
Extension tự nhận diện domain đang hoạt động, tự inject script vào tab, lưu cấu hình worker và resume khi tab reload.

> Dự án dùng cho mục đích cá nhân. Hãy sử dụng có trách nhiệm và tuân thủ quy định của website.

## Tính năng chính

### Tác vụ tự động

| Tác vụ | Mô tả |
| --- | --- |
| 🎁 Phúc Lợi | Tự mở rương ở Phúc Lợi Đường |
| 🛡️ Hoang Vực | Tự đánh Boss Hoang Vực |
| 👹 Tông Môn | Tự đánh Boss Tông Môn |
| 🎡 Vòng Quay | Tự quay Vòng Quay Phúc Vận |
| 💎 Thí Luyện | Tự mở rương Thí Luyện Tông Môn |
| ❓ Vấn Đáp | Tự trả lời câu hỏi từ `answers.json` |
| 🙏 Tế Lễ | Tự tế lễ khi còn lượt |
| 🏆 Thưởng Ngày | Tự nhận thưởng hoạt động ngày |
| ⛏️ Khoáng Mạch | Tự chọn mỏ, vào mỏ và nhận thưởng |
| 🧪 Luyện Đan | Tự luyện đan, điều hỏa và phân giải khi túi đầy |
| ⚔️ Mê Cung | Tự lập hoặc vào phòng, chiến đấu và nhận rương |

### Hệ thống

- 🌐 Auto-detect domain `hoathinh3d.*`, không cần sửa URL khi website đổi đuôi.
- 💉 Dynamic injection qua Manifest V3 và `chrome.scripting`.
- 🔄 Auto-resume worker sau khi refresh tab hoặc restart extension.
- ✅ Lưu trạng thái hoàn thành theo ngày bằng `chrome.storage.local`.
- ⏭️ Tự bỏ qua worker đã hoàn thành trong ngày.
- 💓 Heartbeat theo dõi kết nối content script/background.
- 🔐 Tự fetch nonce, token và action từ trang hiện tại.
- ⏱️ Request queue tuần tự, delay tối thiểu 6 giây giữa request.
- 🔁 Retry khi gặp lỗi mạng, HTTP 429 hoặc HTTP 503.
- 📋 Nhật ký realtime trong side panel.

## Cài đặt

1. Tải hoặc clone repository này.
2. Mở Chrome và truy cập:

   ```text
   chrome://extensions/
   ```

3. Bật **Developer mode**.
4. Chọn **Load unpacked**.
5. Chọn thư mục extension chứa `manifest.json`.
6. Truy cập `hoathinh3d.*` và đăng nhập tài khoản.

## Sử dụng

1. Mở website `hoathinh3d.*` đang hoạt động.
2. Đăng nhập tài khoản.
3. Bấm icon **HH3D Auto Tool** để mở side panel.
4. Kiểm tra domain ở đầu panel. Trạng thái **Sẵn sàng** nghĩa là chưa chạy. Nếu tab không phải `hoathinh3d`, panel hiện **Chưa đúng tab**.
5. Trong **Tác vụ**, chọn việc muốn chạy hoặc bấm **Tất cả**.
6. Với **Khoáng Mạch**:
   - Chọn loại mỏ: Thượng phẩm (Vàng), Trung phẩm (Bạc) hoặc Hạ phẩm (Đồng).
   - Bấm **Tải mỏ**.
   - Chọn mỏ cụ thể.
7. Với **Mê Cung**:
   - Chọn số người.
   - Chọn vai trò **Chủ phòng** hoặc **Thành viên**.
   - Ô tinh thạch để trống nếu muốn tự nhận theo VIP.
8. Với **Luyện Đan**:
   - Chọn phẩm luyện. Mặc định là phẩm cao nhất.
   - Bật **Tự phân giải khi túi đầy** nếu cần, rồi chọn phẩm và số sao.
9. Bấm **Bắt đầu**. Trạng thái chuyển thành **Đang chạy**.
10. Theo dõi mục **Nhật ký**.
11. Bấm **Dừng** khi muốn dừng toàn bộ tác vụ.

## Cấu trúc thư mục

```text
extention/
├── manifest.json          # Manifest V3
├── background.js          # Service worker, quản lý tab và inject script
├── content.js             # Logic worker chính
├── inject.js              # Bridge đọc dữ liệu từ page context
├── popup.html             # Giao diện side panel
├── popup.css              # Style side panel
├── popup.js               # Logic side panel
├── sidepanel-disabled.html # Màn hình khi tab không phải hoathinh3d
├── luyen-dan.js           # Logic liên quan Luyện Đan
├── answers.json           # Dữ liệu đáp án Vấn Đáp
├── icons/                 # Icon extension
├── *.txt                  # Ghi chú/phân tích endpoint, page source
└── README.md              # Tài liệu dự án
```

## Quyền Chrome Extension

Extension dùng các quyền sau:

| Permission | Mục đích |
| --- | --- |
| `storage` | Lưu cấu hình, trạng thái worker, trạng thái hoàn thành theo ngày |
| `tabs` | Tìm tab `hoathinh3d.*` đang mở |
| `alarms` | Hỗ trợ tác vụ nền |
| `activeTab` | Tương tác tab hiện tại |
| `scripting` | Inject content script động |
| `sidePanel` | Mở giao diện điều khiển trong side panel |
| `<all_urls>` | Tự phát hiện mọi domain `hoathinh3d.*` khi website đổi đuôi |

## Bảo mật

- Extension không gửi cookie, token hay dữ liệu tài khoản đến server bên thứ ba.
- Request chỉ chạy trên domain `hoathinh3d.*` đang mở.
- Dữ liệu trạng thái chỉ lưu cục bộ trong `chrome.storage.local`.
- Có thể kiểm tra toàn bộ mã nguồn trước khi cài đặt.

## Lưu ý vận hành

- Cần đăng nhập website trước khi chạy extension.
- Không đóng tab website nếu muốn worker tiếp tục chạy ổn định.
- Đóng side panel không dừng tác vụ. Worker vẫn chạy trong content script.
- Worker đã hoàn thành trong ngày sẽ được đánh dấu và bỏ qua khi resume.
- Khi hết lượt trong ngày, worker có thể chờ đến 0h hoặc dừng tùy logic từng tác vụ.
- Nếu website thay đổi endpoint hoặc token, cần cập nhật logic fetch nonce/action.

## Debug

Nếu extension không chạy đúng:

1. Kiểm tra side panel có nhận đúng domain không.
2. Refresh tab `hoathinh3d.*`.
3. Mở `chrome://extensions/`.
4. Chọn extension **HH3D Auto Tool**.
5. Mở **Service worker** để xem log background.
6. Mở DevTools trên tab website để xem log content script.
7. Bấm reload extension nếu context bị mất.
8. Đăng nhập lại nếu log báo phiên hết hạn.

## Changelog

### 0.1.0

- Khởi tạo app lần đầu.

## Disclaimer

Dự án không liên kết với website `hoathinh3d` hoặc bất kỳ bên thứ ba nào.
Người dùng tự chịu trách nhiệm khi sử dụng extension.
