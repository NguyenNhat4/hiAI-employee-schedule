# Phase 1: Hướng Dẫn Cài Đặt và Cấu Hình

Tài liệu này hướng dẫn chi tiết cách thiết lập môi trường cho hệ thống AI Thống kê xin nghỉ phép, đi trễ,... từ Telegram.

## 1. Yêu cầu hệ thống

- Python 3.8 trở lên
- Kết nối Internet ổn định

## 2. Cài đặt thư viện

Mở terminal tại thư mục gốc của dự án và chạy lệnh sau để cài đặt các thư viện cần thiết:

> **Lưu ý:** Tải uv tại [https://docs.astral.sh/uv/getting-started/installation/#installation-methods](https://docs.astral.sh/uv/getting-started/installation/#installation-methods)

```bash
uv pip install -r requirements.txt
```

## 3. Lấy thông tin API

### 3.1. Telegram API (Bắt buộc)

Để truy cập tin nhắn Telegram, bạn cần tạo một ứng dụng trên Telegram:

1. Truy cập [https://my.telegram.org](https://my.telegram.org)
2. Đăng nhập bằng số điện thoại của bạn
3. Chọn **API development tools**
4. Điền thông tin vào form (App title, Short name...)
5. Sau khi tạo xong, bạn sẽ thấy `App api_id` và `App api_hash`. Hãy lưu lại hai thông tin này

### 3.2. Gemini API (Cho Phase 3)

Để sử dụng AI phân tích tin nhắn:

1. Truy cập [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Tạo một API Key mới
3. Lưu lại API Key

## 4. Cấu hình môi trường

1. Sao chép file `.env.example` thành file `.env`:
   - **Windows:** `copy .env.example .env`
   - **Linux/Mac:** `cp .env.example .env`

2. Mở file `.env` bằng trình soạn thảo văn bản và điền thông tin đã lấy ở bước 3:

```ini
TELEGRAM_API_ID=123456
TELEGRAM_API_HASH=abcdef123456...
GEMINI_API_KEY=AIzaSyD...
# Để trống các trường TARGET_GROUP_ID và TARGET_TOPIC_ID tạm thời
TARGET_GROUP_ID=
TARGET_TOPIC_ID=
SESSION_NAME=telegram_session
```

## 5. Xác thực Telegram

Chạy script xác thực để đăng nhập vào Telegram:

```bash
python scripts/telegram_auth.py
```

Làm theo hướng dẫn trên màn hình để nhập mã xác thực từ Telegram.

## 6. Lấy ID nhóm và Topic

Để bot biết cần lấy dữ liệu từ nhóm nào, bạn cần lấy ID của nhóm và Topic "Schedule":

1. Chạy script lấy thông tin dialog:

```bash
python scripts/get_dialog_info.py
```

2. Kiểm tra file `data_raw/dialog_info.json` để tìm ID nhóm và topic cần thiết

3. Cập nhật file `.env` với `TARGET_GROUP_ID` và `TARGET_TOPIC_ID` phù hợp

## 7. Chạy hệ thống

Sau khi hoàn thành các bước cài đặt, chạy flow chính:

```bash
uv run python flow.py
```

Hệ thống sẽ:
- Tìm các topic schedule từ `data_raw/dialog_info.json`
- Fetch messages từ Telegram
- Label các tin nhắn schedule
- Xuất kết quả ra file Excel 
