# Sơ đồ luồng hệ thống - AI Employee Schedule

## Tổng quan luồng chính (Main Flow)

```mermaid
flowchart TD
    Start([🚀 Bắt đầu - flow.py]) --> Init[Khởi tạo shared store]
    Init --> F

    subgraph Flow["AsyncFlow - Schedule Pipeline"]
        direction TB

        F[📥 FetchTelegramMessagesNode\n<i>AsyncNode</i>]
        G[📅 GroupMessagesByWeekNode\n<i>Node</i>]
        L[🏷️ LabelScheduleMessagesNode\n<i>AsyncParallelBatchNode</i>]
        E[📊 ExportExcelNode\n<i>Node</i>]

        F -->|CSV messages| G
        G -->|Weekly CSV list| L
        L -->|Labeled YAML| E
    end

    E --> Output([📁 Excel Report])
```

## Chi tiết từng Node

### 1. FetchTelegramMessagesNode

```mermaid
flowchart TD
    A([Bắt đầu Fetch]) --> B[prep_async:\nLấy year, month hiện tại]
    B --> C[Đọc data_raw/dialog_info.json]
    C --> D{Tìm topic\n'schedule'?}
    D -->|Không tìm thấy| Err1[❌ Lỗi: Không có topic schedule]
    D -->|Tìm thấy| E[Khởi tạo TelegramClient\nvới API_ID + API_HASH]
    E --> F[Kết nối Telegram]

    F --> Loop

    subgraph Loop["Lặp qua từng Schedule Topic"]
        direction TB
        G[iter_messages\nfrom group/topic]
        H{Tin nhắn\ncòn trong\ntháng hiện tại?}
        I[Lấy sender info\nfirst_name + last_name]
        J[Tạo msg_data:\nmessage_id, name, date, message]

        G --> H
        H -->|Có| I --> J --> G
        H -->|Không / Hết| K[Kết thúc topic]
    end

    K --> L{Còn topic\nkhác?}
    L -->|Có| Loop
    L -->|Không| M[Sắp xếp theo ngày\nLọc tin nhắn rỗng]
    M --> N[Chuyển đổi sang CSV]
    N --> O[post_async:\nLưu vào shared.telegram_messages_csv]
    O --> Done([✅ Hoàn tất Fetch])
```

### 2. GroupMessagesByWeekNode

```mermaid
flowchart TD
    A([Bắt đầu Group]) --> B[prep:\nĐọc shared.telegram_messages_csv]
    B --> C{CSV rỗng?}
    C -->|Có| Empty[Trả về mảng rỗng]
    C -->|Không| D[Parse CSV thành\nlist of dicts]
    D --> E[Lặp qua từng message]

    E --> F[Tính week_key\nMonday của tuần đó]
    F --> G[Nhóm message\nvào đúng tuần]
    G --> H{Còn message?}
    H -->|Có| E
    H -->|Không| I[Sắp xếp theo week_key]
    I --> J[Chuyển mỗi tuần\nthành CSV string]
    J --> K[post:\nLưu vào shared.weekly_messages]
    K --> Done([✅ Hoàn tất Group])

    Empty --> Done
```

### 3. LabelScheduleMessagesNode

```mermaid
flowchart TD
    A([Bắt đầu Label]) --> B[prep_async:\nĐọc shared.weekly_messages]
    B --> C{Có dữ liệu?}
    C -->|Không| Empty[Trả về kết quả rỗng]
    C -->|Có| D[AsyncParallelBatchNode:\nXử lý song song từng tuần]

    subgraph Parallel["⚡ Xử lý song song mỗi tuần"]
        direction TB
        W1[Tuần 1\nCSV → LLM]
        W2[Tuần 2\nCSV → LLM]
        W3[Tuần N\nCSV → LLM]
    end

    D --> Parallel

    subgraph LLM_Call["Xử lý 1 tuần (exec_async)"]
        direction TB
        P1[Tạo prompt với\nSystem Prompt + CSV]
        P2[Gọi Gemini API\ncall_llm_async]
        P3[Clean response\nXóa markdown blocks]
        P4[Parse YAML]
        P5[Normalize result\n4 categories]

        P1 --> P2 --> P3 --> P4 --> P5
    end

    Parallel --> LLM_Call

    subgraph Categories["📋 4 Loại phân loại"]
        direction LR
        C1["🏖️ nghi\nNghỉ phép"]
        C2["⏰ tre\nĐi trễ"]
        C3["🌓 nua_buoi\nNửa buổi"]
        C4["💻 remote\nLàm remote"]
    end

    LLM_Call --> Categories

    Categories --> Merge[post_async:\nGộp kết quả tất cả tuần]
    Merge --> Save[Lưu vào shared:\n- labeled_messages\n- weekly_labeled_messages\n- labeled_messages_yaml]
    Save --> Done([✅ Hoàn tất Label])
    Empty --> Done
```

### 4. ExportExcelNode

```mermaid
flowchart TD
    A([Bắt đầu Export]) --> B[prep:\nĐọc shared.labeled_messages]
    B --> C{Có dữ liệu?}
    C -->|Không| Skip[Bỏ qua]
    C -->|Có| D[Tạo Workbook mới]

    D --> E[Tạo sheet 'Tổng hợp'\nBảng thống kê số lượng]

    E --> F[Tạo sheet 'Nghỉ phép'\nSTT, Message ID, Tên, Ngày, Thông tin]
    F --> G[Tạo sheet 'Đi trễ']
    G --> H[Tạo sheet 'Nghỉ nửa buổi']
    H --> I[Tạo sheet 'Làm remote']

    I --> J[post:\nTạo thư mục output/]
    J --> K[Lưu file Excel\nschedule_report_YYYYMMDD_HHMMSS.xlsx]
    K --> L[Lưu đường dẫn\nvào shared.excel_output_path]
    L --> Done([✅ Hoàn tất Export])
    Skip --> Done
```

## Kiến trúc tổng thể

```mermaid
flowchart LR
    subgraph External["🌐 Dịch vụ bên ngoài"]
        TG["📱 Telegram API"]
        GM["🤖 Gemini API"]
    end

    subgraph Config["⚙️ Cấu hình"]
        ENV[".env\nAPI_ID, API_HASH\nGEMINI_API_KEY"]
        DI["dialog_info.json\nGroup & Topic IDs"]
    end

    subgraph Core["🔧 Core Framework"]
        PF["pocketflow.py\nBaseNode, Node\nAsyncNode, AsyncFlow\nAsyncParallelBatchNode"]
    end

    subgraph Nodes["📦 Nodes"]
        N1["FetchTelegramMessagesNode"]
        N2["GroupMessagesByWeekNode"]
        N3["LabelScheduleMessagesNode"]
        N4["ExportExcelNode"]
    end

    subgraph Utils["🛠️ Utils"]
        LLM["call_llm.py\ncall_llm / call_llm_async"]
    end

    subgraph Output["📤 Output"]
        XL["📊 Excel Report\noutput/schedule_report_*.xlsx"]
    end

    ENV --> N1
    DI --> N1
    TG <--> N1
    N1 --> N2
    N2 --> N3
    N3 --> N4
    GM <--> LLM
    LLM <--> N3
    N4 --> XL
    PF -.->|kế thừa| Nodes
```

## Shared Store (Dữ liệu chia sẻ giữa các Node)

```mermaid
flowchart LR
    subgraph SharedStore["📦 Shared Store"]
        direction TB
        S1["telegram_messages_csv\n<i>CSV string</i>"]
        S2["weekly_messages\n<i>List[dict] - weekly CSVs</i>"]
        S3["labeled_messages\n<i>Dict - merged labels</i>"]
        S4["weekly_labeled_messages\n<i>List[dict] - per-week labels</i>"]
        S5["labeled_messages_yaml\n<i>YAML string</i>"]
        S6["excel_output_path\n<i>File path string</i>"]
    end

    Fetch["FetchTelegramMessagesNode"] -->|ghi| S1
    S1 -->|đọc| Group["GroupMessagesByWeekNode"]
    Group -->|ghi| S2
    S2 -->|đọc| Label["LabelScheduleMessagesNode"]
    Label -->|ghi| S3
    Label -->|ghi| S4
    Label -->|ghi| S5
    S3 -->|đọc| Export["ExportExcelNode"]
    Export -->|ghi| S6
```
