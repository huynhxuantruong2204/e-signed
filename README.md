# 📋 E-Signed System - Tổng Quan Kiến Trúc & Module Chức Năng

## 1. Sơ Đồ Kiến Trúc Tổng Quan Hệ Thống

```mermaid
graph TB
    subgraph CLIENT["🌐 CLIENT PORTAL - Tra Cứu & Ký Online"]
        direction TB
        C_USER["👤 User Vãng Lai / User Hệ Thống"]
        C_LOOKUP["🔍 Trang Tra Cứu<br/>Nhập Mã Hợp Đồng / Hóa Đơn"]
        C_RESULT["📄 Hiển Thị Thông Tin & Trạng Thái"]
        C_SIGN["✍️ Mở File & Ký Điện Tử"]
        C_DOWNLOAD["⬇️ Tải Về Tài Liệu Đã Ký"]
        C_NOTI["📩 Gửi Thông Báo<br/>Email / Zalo"]

        C_USER --> C_LOOKUP
        C_LOOKUP --> C_RESULT
        C_RESULT -->|"Chưa ký"| C_SIGN
        C_SIGN --> C_NOTI
        C_RESULT -->|"Đã ký"| C_DOWNLOAD
    end

    subgraph ADMIN["🛡️ ADMIN PORTAL - Quản Lý Ký Online"]
        direction TB
        A_USER["👨‍💼 Admin"]
        A_UPLOAD["📤 Upload Tài Liệu Cần Ký"]
        A_ASSIGN["👥 Phân Quyền Người Ký"]
        A_SEND["📧 Gửi Mail Thông Báo<br/>Có Thời Hạn Ký"]
        A_REMIND["⏰ Nhắc Ký Tự Động<br/>1 lần/ngày"]
        A_MANAGE["📊 Quản Lý & Theo Dõi"]

        A_USER --> A_UPLOAD
        A_UPLOAD --> A_ASSIGN
        A_ASSIGN --> A_SEND
        A_SEND --> A_REMIND
        A_USER --> A_MANAGE
    end

    subgraph BACKEND["⚙️ BACKEND SERVICES"]
        direction TB
        API["🔌 API Gateway"]
        AUTH["🔐 Authentication & Authorization"]
        DOC_SVC["📑 Document Service"]
        SIGN_SVC["✍️ Signing Service"]
        INV_SVC["🧾 Invoice Service<br/>Tạo XML & Gửi Thuế"]
        NOTI_SVC["🔔 Notification Service<br/>Email / Zalo"]
        BATCH_SVC["⚡ Batch Processing Service<br/>Ký hàng loạt Hóa Đơn"]
        SCHEDULER["⏲️ Scheduler Service<br/>Nhắc ký & Polling Thuế"]
        USER_SVC["👥 User Management Service"]
    end

    subgraph TAX_AUTHORITY["🏛️ CỤC THUẾ - EXTERNAL"]
        direction LR
        TAX_API["🔗 Tax Authority API<br/>Nhận XML Hóa Đơn"]
        TAX_RESULT["📋 Trả Kết Quả<br/>Mã CQT / Lỗi"]
    end

    subgraph STORAGE["💾 DATA STORAGE"]
        direction LR
        DB[("🗄️ Database<br/>PostgreSQL")]
        FILE_STORE["📁 File Storage<br/>S3 / MinIO"]
        CACHE["⚡ Cache<br/>Redis"]
        QUEUE["📨 Message Queue<br/>RabbitMQ"]
    end

    CLIENT --> API
    ADMIN --> API
    API --> AUTH
    API --> DOC_SVC
    API --> SIGN_SVC
    API --> INV_SVC
    API --> NOTI_SVC
    API --> BATCH_SVC
    API --> USER_SVC
    SCHEDULER --> NOTI_SVC
    SCHEDULER --> DOC_SVC
    SCHEDULER --> INV_SVC

    INV_SVC --> TAX_API
    TAX_API --> TAX_RESULT
    TAX_RESULT --> INV_SVC
    INV_SVC --> DB
    INV_SVC --> QUEUE

    DOC_SVC --> DB
    DOC_SVC --> FILE_STORE
    SIGN_SVC --> DB
    SIGN_SVC --> FILE_STORE
    NOTI_SVC --> QUEUE
    BATCH_SVC --> QUEUE
    BATCH_SVC --> CACHE
    USER_SVC --> DB

    style CLIENT fill:#1a1a2e,stroke:#e94560,stroke-width:2px,color:#fff
    style ADMIN fill:#1a1a2e,stroke:#0f3460,stroke-width:2px,color:#fff
    style BACKEND fill:#16213e,stroke:#533483,stroke-width:2px,color:#fff
    style STORAGE fill:#0f3460,stroke:#e94560,stroke-width:2px,color:#fff
    style TAX_AUTHORITY fill:#2d3436,stroke:#fdcb6e,stroke-width:3px,color:#fff
```

---

## 2. Luồng Xử Lý Chi Tiết - Client Portal

```mermaid
flowchart TD
    START(["🚀 User truy cập trang tra cứu"]) --> INPUT["Nhập mã Hợp Đồng / Hóa Đơn"]
    INPUT --> VALIDATE{"Mã hợp lệ?"}
    VALIDATE -->|"Không"| ERROR["❌ Hiển thị lỗi:<br/>Mã không tồn tại"]
    ERROR --> INPUT
    VALIDATE -->|"Có"| FETCH["📄 Lấy thông tin tài liệu"]
    FETCH --> DISPLAY["Hiển thị thông tin:<br/>• Tên tài liệu<br/>• Ngày tạo<br/>• Người ký<br/>• Trạng thái"]
    DISPLAY --> CHECK{"Trạng thái?"}

    CHECK -->|"✅ Đã ký"| SIGNED["Hiển thị tài liệu đã ký"]
    SIGNED --> DOWNLOAD["⬇️ Nút Tải Về PDF"]

    CHECK -->|"⏳ Chưa ký"| AUTH{"Xác thực User?"}
    AUTH -->|"Chưa"| LOGIN["🔐 Xác thực OTP<br/>qua Email / SMS"]
    LOGIN --> VERIFY{"OTP hợp lệ?"}
    VERIFY -->|"Không"| LOGIN
    VERIFY -->|"Có"| OPEN_DOC
    AUTH -->|"Đã xác thực"| CHECK_KEY{"Có RSA keypair?"}
    CHECK_KEY -->|"Chưa"| GEN_KEY["🔑 Generate RSA-2048 Keypair<br/>Lưu Public Key lên Server<br/>Private Key giữ phía Client"]
    GEN_KEY --> OPEN_DOC
    CHECK_KEY -->|"Có rồi"| OPEN_DOC["📖 Mở file xem trước<br/>Hiển thị vị trí cần ký"]

    OPEN_DOC --> SIG_TYPE{"Chọn loại chữ ký<br/>trực quan?"}
    SIG_TYPE -->|"✏️ Vẽ tay"| DRAW["🎨 Vẽ chữ ký trên Canvas<br/>Hỗ trợ chuột / touch / bút"]
    SIG_TYPE -->|"📷 Upload ảnh"| UPLOAD_IMG["🖼️ Upload ảnh chữ ký<br/>PNG / JPG / SVG<br/>Tự động xóa nền"]
    SIG_TYPE -->|"💾 Chữ ký đã lưu"| SAVED_SIG["📋 Chọn từ danh sách<br/>chữ ký đã tạo trước"]

    DRAW --> PLACE_SIG["📌 Đặt chữ ký vào vị trí<br/>Drag & Drop trên PDF<br/>Resize / Xoay"]
    UPLOAD_IMG --> PLACE_SIG
    SAVED_SIG --> PLACE_SIG

    PLACE_SIG --> PREVIEW["👁️ Xem trước tài liệu<br/>với chữ ký đã đặt"]
    PREVIEW --> CONFIRM{"Xác nhận ký?"}
    CONFIRM -->|"Sửa lại"| OPEN_DOC
    CONFIRM -->|"Xác nhận"| SEND_OTP["📩 Gửi OTP xác nhận<br/>qua Email hoặc SMS<br/>Mã có hiệu lực 5 phút"]
    SEND_OTP --> INPUT_OTP["🔢 User nhập mã OTP"]
    INPUT_OTP --> CHECK_OTP{"OTP hợp lệ?"}
    CHECK_OTP -->|"Sai / Hết hạn"| RESEND{"Gửi lại?"}
    RESEND -->|"Có"| SEND_OTP
    RESEND -->|"Hủy"| OPEN_DOC
    CHECK_OTP -->|"Đúng"| RSA_SIGN["🔐 KÝ SỐ RSA:<br/>1. Hash tài liệu (SHA-256)<br/>2. Ký hash bằng Private Key<br/>3. Đính chữ ký RSA + ảnh vào PDF"]
    RSA_SIGN --> VERIFY_SIG["✅ Server verify chữ ký<br/>Decrypt bằng Public Key<br/>So sánh hash → Xác nhận"]
    VERIFY_SIG --> SAVE["💾 Lưu tài liệu đã ký<br/>• RSA signature metadata<br/>• Ảnh chữ ký trực quan<br/>• Timestamp + IP + OTP log"]
    SAVE --> NOTIFY["📩 Gửi thông báo<br/>Email + Zalo"]
    NOTIFY --> COMPLETE(["✅ Hoàn tất"])

    style START fill:#e94560,color:#fff
    style COMPLETE fill:#00b894,color:#fff
    style ERROR fill:#d63031,color:#fff
    style DOWNLOAD fill:#0984e3,color:#fff
    style GEN_KEY fill:#6c5ce7,color:#fff
    style DRAW fill:#a29bfe,color:#fff
    style UPLOAD_IMG fill:#a29bfe,color:#fff
    style SAVED_SIG fill:#a29bfe,color:#fff
    style PLACE_SIG fill:#fdcb6e,color:#000
    style SEND_OTP fill:#e17055,color:#fff
    style INPUT_OTP fill:#e17055,color:#fff
    style RSA_SIGN fill:#d63031,color:#fff
    style VERIFY_SIG fill:#0984e3,color:#fff
```

---

## 3. Luồng Quản Lý Tài Liệu - Ký File PDF (Admin)

> [!IMPORTANT]
> **Luồng này dành cho TÀI LIỆU (Hợp đồng, Phụ lục, Biên bản…)** — Admin upload file PDF có sẵn → chỉ định người ký → theo dõi đến khi ký xong. Hoàn toàn **tách biệt** với luồng Hóa đơn điện tử.

```mermaid
flowchart TD
    A_START(["🚀 Admin đăng nhập"]) --> DASHBOARD["📊 Dashboard tổng quan"]

    DASHBOARD --> DOC_MGMT["📑 Quản Lý Tài Liệu<br/>Hợp đồng / Phụ lục / Biên bản"]
    DASHBOARD --> INV_MGMT["🧾 Quản Lý Hóa Đơn ĐT<br/>→ Xem Section 3b"]
    DASHBOARD --> MANAGE_USER["👥 Quản lý User"]
    DASHBOARD --> AUDIT_LINK["🔍 Audit Log<br/>→ Xem Section 3c"]
    DASHBOARD --> REPORTS["📈 Báo cáo & Thống kê"]

    DOC_MGMT --> DOC_LIST["📋 Danh sách tài liệu<br/>Tìm kiếm / Lọc / Phân loại"]
    DOC_MGMT --> UPLOAD_FLOW["📤 Upload tài liệu mới"]

    UPLOAD_FLOW --> UPLOAD_TYPE{"Loại upload?"}
    UPLOAD_TYPE -->|"Đơn lẻ"| SINGLE["📄 Upload 1 file PDF<br/>Hợp đồng / Phụ lục"]
    UPLOAD_TYPE -->|"Hàng loạt"| BATCH["📦 Upload Batch<br/>Nhiều file PDF cùng lúc"]

    SINGLE --> SET_META["📝 Nhập thông tin:<br/>• Tên tài liệu<br/>• Loại (HĐ/PL/BB)<br/>• Mã tra cứu<br/>• Mô tả"]
    BATCH --> BATCH_MAP["🗺️ Mapping dữ liệu<br/>• CSV template kèm file<br/>• Auto-mapping metadata"]
    BATCH_MAP --> BATCH_PROCESS["⚡ Xử lý hàng loạt<br/>Queue + Worker"]
    BATCH_PROCESS --> SET_META

    SET_META --> ASSIGN["👥 Chỉ định người ký<br/>• Chọn user hiện có<br/>• Thêm user mới via email<br/>• Thứ tự ký (tuần tự/đồng thời)"]
    ASSIGN --> SET_DEADLINE["⏰ Đặt thời hạn ký"]
    SET_DEADLINE --> SEND_NOTI["📧 Gửi Email thông báo<br/>kèm link ký trực tiếp"]
    SEND_NOTI --> SCHEDULE["⏲️ Lập lịch nhắc<br/>1 lần/ngày đến khi ký xong"]
    SCHEDULE --> MONITOR["📊 Theo dõi trạng thái<br/>Xem ai đã ký / chưa ký"]

    MONITOR --> CHECK_STATUS{"Tất cả đã ký?"}
    CHECK_STATUS -->|"Chưa + Chưa hết hạn"| REMIND["📩 Auto nhắc nhở<br/>Email + Zalo"]
    REMIND --> MONITOR
    CHECK_STATUS -->|"Chưa + Hết hạn"| EXPIRED["⚠️ Đánh dấu quá hạn<br/>Thông báo Admin escalate"]
    CHECK_STATUS -->|"Đã ký hết"| COMPLETE_DOC["✅ Tài liệu hoàn tất"]
    COMPLETE_DOC --> ARCHIVE(["📁 Lưu trữ & cho phép tải về<br/>Gửi bản copy cho các bên"])

    style A_START fill:#0f3460,color:#fff
    style DOC_MGMT fill:#0984e3,color:#fff
    style INV_MGMT fill:#e17055,color:#fff
    style AUDIT_LINK fill:#2d3436,color:#fff
    style ARCHIVE fill:#00b894,color:#fff
    style EXPIRED fill:#fdcb6e,color:#000
    style BATCH fill:#6c5ce7,color:#fff
    style BATCH_PROCESS fill:#6c5ce7,color:#fff
```

---

## 3b. Luồng Quản Lý Hóa Đơn Điện Tử (Admin)

> [!IMPORTANT]
> **Luồng này dành cho HÓA ĐƠN ĐIỆN TỬ** — User/Admin nhập thông tin các field theo quy định Cục Thuế → hệ thống tạo XML → đẩy lên thuế → nhận mã CQT. **Không phải** upload file PDF có sẵn.

```mermaid
flowchart TD
    INV_START(["🚀 Admin vào Quản lý Hóa đơn"]) --> INV_DASH["📊 Dashboard Hóa Đơn<br/>• Tổng HĐ đã phát hành<br/>• Chờ thuế duyệt<br/>• Bị từ chối<br/>• Đã hủy"]

    INV_DASH --> INV_LIST["📋 Danh sách hóa đơn<br/>Lọc: Nháp | Đã gửi | Cấp mã | Từ chối | Hủy"]
    INV_DASH --> INV_CREATE["➕ Tạo hóa đơn mới"]
    INV_DASH --> INV_BATCH["📦 Tạo batch hóa đơn"]
    INV_DASH --> INV_ADJUST["🔄 Hủy / Điều chỉnh / Thay thế"]
    INV_DASH --> INV_REPORT["📈 Báo cáo hóa đơn"]

    %% === LUỒNG TẠO ĐƠN LẺ ===
    INV_CREATE --> INPUT_FORM["📝 FORM NHẬP HÓA ĐƠN<br/>━━━━━━━━━━━━━━━━━━<br/>🏢 Bên bán: MST, Tên, Địa chỉ, TK<br/>🏪 Bên mua: MST, Tên, Địa chỉ, Email<br/>📦 Hàng hóa: Tên, ĐVT, SL, Đơn giá<br/>💰 Thuế suất VAT: 0% / 5% / 8% / 10%<br/>💳 Hình thức thanh toán<br/>📋 Ký hiệu mẫu, Ký hiệu HĐ"]

    INPUT_FORM --> AUTO_CALC["🧮 Tự động tính toán:<br/>• Thành tiền = SL × Đơn giá<br/>• Thuế GTGT<br/>• Tổng thanh toán<br/>• Số tiền bằng chữ"]
    AUTO_CALC --> VALIDATE_INV{"Dữ liệu<br/>hợp lệ?"}
    VALIDATE_INV -->|"Không"| SHOW_ERR["❌ Lỗi validation<br/>Highlight field sai"]
    SHOW_ERR --> INPUT_FORM
    VALIDATE_INV -->|"Có"| SAVE_DRAFT["💾 Lưu nháp<br/>Status: DRAFT"]

    SAVE_DRAFT --> PREVIEW["👁️ Xem trước HĐ<br/>Preview PDF"]
    PREVIEW --> EDIT_OR_SEND{"Hành động?"}
    EDIT_OR_SEND -->|"Sửa lại"| INPUT_FORM
    EDIT_OR_SEND -->|"Gửi Cục Thuế"| SUBMIT_TAX["📤 GỬI CỤC THUẾ<br/>→ Xem Section 4"]

    %% === LUỒNG BATCH ===
    INV_BATCH --> IMPORT_FILE["📥 Import Excel / CSV<br/>Theo template chuẩn"]
    IMPORT_FILE --> PARSE_DATA["🔍 Parse & validate<br/>từng dòng = 1 HĐ"]
    PARSE_DATA --> PREVIEW_BATCH["📋 Preview danh sách<br/>✅ Hợp lệ: X | ❌ Lỗi: Y"]
    PREVIEW_BATCH --> FIX_ERR{"Có lỗi?"}
    FIX_ERR -->|"Có"| FIX["✏️ Sửa trực tiếp<br/>hoặc re-import"]
    FIX --> IMPORT_FILE
    FIX_ERR -->|"Không"| BATCH_SUBMIT["📤 Gửi batch lên thuế<br/>Queue + Rate limiting<br/>→ Xem Section 8"]

    %% === LUỒNG HỦY / ĐIỀU CHỈNH ===
    INV_ADJUST --> ADJ_TYPE{"Loại?"}
    ADJ_TYPE -->|"Hủy"| CANCEL_INV["🚫 Tạo HĐ hủy<br/>Lý do hủy bắt buộc"]
    ADJ_TYPE -->|"Điều chỉnh"| ADJUST_INV["📝 Tạo HĐ điều chỉnh<br/>Tăng / Giảm"]
    ADJ_TYPE -->|"Thay thế"| REPLACE_INV["🔄 Tạo HĐ thay thế<br/>Link HĐ gốc"]
    CANCEL_INV --> SUBMIT_ADJ["📤 Gửi lên Cục Thuế"]
    ADJUST_INV --> SUBMIT_ADJ
    REPLACE_INV --> SUBMIT_ADJ

    %% === BÁO CÁO ===
    INV_REPORT --> RPT_TYPE{"Loại báo cáo?"}
    RPT_TYPE --> RPT_PERIOD["📅 Tổng hợp theo kỳ"]
    RPT_TYPE --> RPT_LIST["📋 Bảng kê hóa đơn"]
    RPT_TYPE --> RPT_XML["📄 Xuất XML báo cáo thuế"]
    RPT_TYPE --> RPT_EXCEL["📊 Xuất Excel"]

    style INV_START fill:#e17055,color:#fff
    style INV_DASH fill:#d63031,color:#fff
    style INPUT_FORM fill:#6c5ce7,color:#fff
    style AUTO_CALC fill:#0984e3,color:#fff
    style SUBMIT_TAX fill:#fdcb6e,color:#000
    style BATCH_SUBMIT fill:#fdcb6e,color:#000
    style SUBMIT_ADJ fill:#fdcb6e,color:#000
    style INV_BATCH fill:#6c5ce7,color:#fff
    style CANCEL_INV fill:#636e72,color:#fff
```

---

## 4. Luồng Kỹ Thuật: Tạo XML & Gửi Cục Thuế (Chi Tiết)

> [!NOTE]
> Đây là luồng kỹ thuật phía backend khi hóa đơn được xác nhận gửi từ Section 3b. Áp dụng cho cả tạo đơn lẻ và batch.

```mermaid
flowchart TD
    INV_CONFIRMED(["📤 Hóa đơn được xác nhận gửi<br/>từ Admin Portal"]) --> BUILD_XML["🔧 XML BUILDER ENGINE<br/>━━━━━━━━━━━━━━━━━━<br/>1. Map fields → XML schema<br/>2. Ký hiệu mẫu + Ký hiệu HĐ<br/>3. Thông tin bên bán/mua<br/>4. Chi tiết hàng hóa<br/>5. Tổng tiền + VAT<br/>6. Theo TT78/2021/TT-BTC"]

    BUILD_XML --> VALIDATE_XML{"XML valid?"}
    VALIDATE_XML -->|"Không"| XML_ERR["❌ Lỗi XML schema<br/>Log lỗi chi tiết"]
    XML_ERR --> NOTIFY_ADMIN_ERR["🔔 Thông báo Admin"]
    VALIDATE_XML -->|"Có"| SIGN_XML["🔐 Ký số XML<br/>Chứng thư số doanh nghiệp<br/>HSM / USB Token"]

    SIGN_XML --> STORE_XML["💾 Lưu XML đã ký<br/>vào File Storage"]
    STORE_XML --> SUBMIT["📤 POST XML lên<br/>API Cục Thuế<br/>+ Mã giao dịch"]
    SUBMIT --> UPDATE_DB["⏳ Cập nhật DB:<br/>status = SUBMITTED<br/>submission_id = ..."]

    UPDATE_DB --> POLL["🔄 POLLING KẾT QUẢ<br/>━━━━━━━━━━━━━━━━<br/>Interval: 15-30 giây<br/>Max retries: 10-20 lần<br/>Timeout: 5 phút"]
    POLL --> TAX_RESPONSE{"Cục Thuế<br/>phản hồi?"}
    TAX_RESPONSE -->|"Chưa có"| WAIT["⏳ Chờ interval..."]
    WAIT --> CHECK_TIMEOUT{"Quá timeout?"}
    CHECK_TIMEOUT -->|"Chưa"| POLL
    CHECK_TIMEOUT -->|"Rồi"| TIMEOUT["⏱️ Timeout<br/>Đánh dấu cần retry sau"]
    TIMEOUT --> RETRY_QUEUE["📨 Đẩy vào retry queue<br/>Exponential backoff"]

    TAX_RESPONSE -->|"✅ Thành công"| GET_CQT["Nhận mã CQT<br/>Cập nhật status = ISSUED"]
    TAX_RESPONSE -->|"❌ Lỗi"| TAX_ERROR["Nhận mã lỗi thuế<br/>Cập nhật status = REJECTED"]

    GET_CQT --> GEN_PDF["📄 Generate PDF chính thức<br/>• Mã CQT trên HĐ<br/>• QR Code tra cứu<br/>• Watermark bảo mật"]
    GEN_PDF --> STORE_PDF["💾 Lưu PDF vào Storage"]
    STORE_PDF --> AUTO_SEND["📩 Tự động gửi cho khách<br/>Email / Zalo / SMS"]
    AUTO_SEND --> LOG_SUCCESS(["✅ Ghi Audit Log<br/>Hoàn tất"])

    TAX_ERROR --> LOG_ERROR["📝 Ghi log lỗi chi tiết"]
    LOG_ERROR --> NOTIFY_ADMIN["🔔 Thông báo Admin<br/>Mã lỗi + Hướng dẫn sửa"]
    NOTIFY_ADMIN --> WAIT_FIX(["⏸️ Chờ Admin sửa & gửi lại"])

    style INV_CONFIRMED fill:#e17055,color:#fff
    style BUILD_XML fill:#6c5ce7,color:#fff
    style SIGN_XML fill:#0984e3,color:#fff
    style SUBMIT fill:#fdcb6e,color:#000
    style POLL fill:#a29bfe,color:#fff
    style GET_CQT fill:#00b894,color:#fff
    style TAX_ERROR fill:#d63031,color:#fff
    style LOG_SUCCESS fill:#00b894,color:#fff
    style TIMEOUT fill:#636e72,color:#fff
```

> [!NOTE]
> **Cấu trúc XML theo Cục Thuế (TT78/2021/TT-BTC):**
> - **Header**: Ký hiệu mẫu, ký hiệu hóa đơn, số hóa đơn
> - **Seller**: MST, tên, địa chỉ, số tài khoản
> - **Buyer**: MST, tên, địa chỉ, email
> - **Items**: Tên hàng hóa/dịch vụ, đơn vị tính, số lượng, đơn giá, thành tiền
> - **Summary**: Tổng tiền trước thuế, thuế GTGT, tổng thanh toán
> - **Digital Signature**: Chữ ký số của doanh nghiệp

---

## 4b. Luồng Kỹ Thuật: Chữ Ký Số RSA (Chi Tiết)

> [!IMPORTANT]
> Chữ ký user sử dụng **mã hóa bất đối xứng RSA** để đảm bảo tính toàn vẹn, xác thực và chống chối bỏ. Mỗi user có 1 cặp khóa RSA riêng.

```mermaid
flowchart TD
    subgraph KEYGEN["🔑 ĐĂNG KÝ CẶP KHÓA RSA - Lần đầu ký"]
        direction TB
        U_FIRST(["👤 User lần đầu ký"]) --> GEN["🔧 Generate RSA-2048 Keypair<br/>━━━━━━━━━━━━━━━━━━<br/>📗 Public Key: 2048-bit<br/>📕 Private Key: 2048-bit"]
        GEN --> STORE_PUB["📤 Gửi Public Key lên Server<br/>Lưu vào DB gắn với User ID"]
        GEN --> STORE_PRIV["💾 Private Key lưu phía Client<br/>• Browser: IndexedDB encrypted<br/>• Mobile: Secure Keychain<br/>• Bảo vệ bằng PIN/Biometric"]
    end

    subgraph SIGNING["✍️ QUY TRÌNH KÝ TÀI LIỆU"]
        direction TB
        DOC_READY(["📄 Tài liệu sẵn sàng ký"]) --> HASH["#️⃣ HASH TÀI LIỆU<br/>━━━━━━━━━━━━━━━━━━<br/>Algorithm: SHA-256<br/>Input: Nội dung PDF binary<br/>Output: 256-bit hash digest"]
        HASH --> ENCRYPT["🔐 KÝ SỐ RSA<br/>━━━━━━━━━━━━━━━━━━<br/>Input: Hash digest<br/>Key: Private Key (client-side)<br/>Algorithm: RSA-PKCS1-v1.5<br/>Output: Digital Signature (256 bytes)"]
        ENCRYPT --> EMBED["📎 ĐÍNH CHỮ KÝ VÀO PDF<br/>━━━━━━━━━━━━━━━━━━<br/>• RSA Signature bytes<br/>• Timestamp ký (ISO 8601)<br/>• Signer ID & metadata<br/>• Certificate chain"]
        EMBED --> SEND_SIG["📤 Gửi lên Server:<br/>• Signed PDF<br/>• RSA Signature<br/>• Document Hash<br/>• Timestamp"]
    end

    subgraph VERIFY["✅ XÁC THỰC CHỮ KÝ (Server-side)"]
        direction TB
        RECV(["📥 Server nhận chữ ký"]) --> GET_PUB["🔑 Lấy Public Key<br/>của User từ DB"]
        GET_PUB --> DECRYPT["🔓 GIẢI MÃ RSA<br/>━━━━━━━━━━━━━━━━━━<br/>Input: Digital Signature<br/>Key: Public Key (server)<br/>Output: Decrypted Hash"]
        DECRYPT --> REHASH["#️⃣ Hash lại tài liệu<br/>SHA-256 từ PDF gốc"]
        REHASH --> COMPARE{"So sánh hash?"}
        COMPARE -->|"✅ Khớp"| VALID["✅ Chữ ký HỢP LỆ<br/>• Tài liệu chưa bị sửa đổi<br/>• Xác nhận đúng người ký<br/>• Chống chối bỏ"]
        COMPARE -->|"❌ Không khớp"| INVALID["❌ Chữ ký KHÔNG HỢP LỆ<br/>• Tài liệu bị thay đổi<br/>• Hoặc chữ ký giả mạo"]
        VALID --> SAVE_DB["💾 Lưu vào DB:<br/>• rsa_signature<br/>• document_hash<br/>• signed_at + IP"]
    end

    KEYGEN --> SIGNING
    SIGNING --> VERIFY

    style U_FIRST fill:#6c5ce7,color:#fff
    style GEN fill:#0984e3,color:#fff
    style DOC_READY fill:#e17055,color:#fff
    style HASH fill:#a29bfe,color:#fff
    style ENCRYPT fill:#d63031,color:#fff
    style EMBED fill:#fdcb6e,color:#000
    style RECV fill:#e17055,color:#fff
    style DECRYPT fill:#00b894,color:#fff
    style VALID fill:#00b894,color:#fff
    style INVALID fill:#d63031,color:#fff
    style KEYGEN fill:#1a1a2e,stroke:#6c5ce7,stroke-width:2px,color:#fff
    style SIGNING fill:#1a1a2e,stroke:#e17055,stroke-width:2px,color:#fff
    style VERIFY fill:#1a1a2e,stroke:#00b894,stroke-width:2px,color:#fff
```

> [!NOTE]
> **Thông số kỹ thuật RSA:**
>
> | Thành phần | Giá trị |
> |---|---|
> | Key Size | RSA-2048 (hoặc RSA-4096 cho bảo mật cao) |
> | Hash Algorithm | SHA-256 |
> | Padding Scheme | PKCS#1 v1.5 hoặc PSS |
> | Signature Size | 256 bytes (RSA-2048) |
> | Private Key Storage | Client-side (IndexedDB encrypted / Secure Keychain) |
> | Public Key Storage | Server-side (DB gắn với User ID) |
> | Encoding | Base64 cho truyền tải, DER/PEM cho lưu trữ |

> [!WARNING]
> **Bảo mật Private Key:**
> - Private Key **KHÔNG BAO GIỜ** được gửi lên server
> - Mã hóa Private Key bằng AES-256 trước khi lưu IndexedDB
> - Yêu cầu PIN/Biometric mỗi lần sử dụng Private Key để ký
> - Hỗ trợ backup/restore keypair qua mã QR encrypted
> - Khi user mất key → Revoke cặp khóa cũ, generate keypair mới

---

## 3c. Luồng Quản Lý Audit Log (Admin)

> [!NOTE]
> Audit Log ghi nhận **mọi hành động** trên hệ thống để đảm bảo truy vết, tuân thủ pháp lý và phát hiện bất thường bảo mật.

```mermaid
flowchart TD
    AL_START(["🔍 Admin vào Audit Log"]) --> AL_DASH["📊 DASHBOARD LOG\n━━━━━━━━━━━━━━━━━━\n• Tổng hành động hôm nay\n• Đăng nhập: X lần\n• Ký tài liệu: Y lần\n• Tạo hóa đơn: Z lần\n• Cảnh báo bảo mật: W"]

    AL_DASH --> AL_SEARCH["🔎 Tìm kiếm & Lọc Log"]
    AL_DASH --> AL_ALERT["🚨 Cảnh báo bảo mật"]
    AL_DASH --> AL_EXPORT["📤 Xuất báo cáo"]
    AL_DASH --> AL_SETTINGS["⚙️ Cài đặt Retention"]

    %% === TÌM KIẾM & LỌC ===
    AL_SEARCH --> FILTER["🔧 BỘ LỌC\n━━━━━━━━━━━━━━━━━━\n👤 Theo User / Role\n📋 Theo hành động:\n    login | logout | view\n    sign | download | upload\n    create_invoice | submit_tax\n    assign | update | delete\n📅 Theo khoảng thời gian\n🌐 Theo IP address\n📄 Theo tài liệu / hóa đơn"]
    FILTER --> RESULT_LIST["📋 Danh sách kết quả\nPhân trang / Sắp xếp"]
    RESULT_LIST --> DETAIL["📄 CHI TIẾT LOG\n━━━━━━━━━━━━━━━━━━\n🕐 Timestamp chính xác\n👤 User: tên, email, role\n📋 Action: hành động cụ thể\n🌐 IP Address\n💻 Device & Browser\n📄 Đối tượng: doc/invoice ID\n📦 Metadata: dữ liệu trước/sau"]

    %% === CẢNH BÁO BẢO MẬT ===
    AL_ALERT --> ALERT_TYPES["🚨 LOẠI CẢNH BÁO\n━━━━━━━━━━━━━━━━━━\n🔐 Đăng nhập sai nhiều lần\n🌍 Đăng nhập từ IP lạ\n⏰ Truy cập ngoài giờ làm việc\n📄 Download hàng loạt bất thường\n🔑 Thay đổi quyền user\n❌ Xóa tài liệu/hóa đơn"]
    ALERT_TYPES --> ALERT_ACTION{"Xử lý?"}
    ALERT_ACTION -->|"Khóa user"| LOCK_USER["🔒 Tạm khóa tài khoản"]
    ALERT_ACTION -->|"Bỏ qua"| DISMISS["✅ Đánh dấu đã xem"]
    ALERT_ACTION -->|"Điều tra"| INVESTIGATE["🔍 Xem chi tiết\nhành động liên quan"]

    %% === XUẤT BÁO CÁO ===
    AL_EXPORT --> EXPORT_TYPE{"Định dạng?"}
    EXPORT_TYPE --> CSV_OUT["📊 CSV / Excel\nDữ liệu thô"]
    EXPORT_TYPE --> PDF_OUT["📄 PDF Compliance\nBáo cáo có chữ ký"]
    EXPORT_TYPE --> PERIOD_RPT["📅 Báo cáo theo kỳ\nTuần / Tháng / Quý"]

    %% === RETENTION ===
    AL_SETTINGS --> RETENTION["⚙️ CHÍNH SÁCH LƯU TRỮ\n━━━━━━━━━━━━━━━━━━\n📅 Giữ log chi tiết: 90 ngày\n📦 Archive log cũ: 1-5 năm\n🗑️ Xóa vĩnh viễn: sau 5 năm\n⏲️ Auto cleanup: CRON hàng tuần"]

    style AL_START fill:#2d3436,color:#fff
    style AL_DASH fill:#636e72,color:#fff
    style FILTER fill:#6c5ce7,color:#fff
    style DETAIL fill:#0984e3,color:#fff
    style ALERT_TYPES fill:#d63031,color:#fff
    style LOCK_USER fill:#e17055,color:#fff
    style RETENTION fill:#00b894,color:#fff
    style CSV_OUT fill:#fdcb6e,color:#000
    style PDF_OUT fill:#fdcb6e,color:#000
```

---

## 5. Scheduler - Luồng Nhắc Ký Tự Động

```mermaid
flowchart LR
    CRON["⏲️ CRON Job<br/>Chạy mỗi ngày<br/>08:00 AM"] --> QUERY["🔍 Query tài liệu<br/>chưa ký + chưa hết hạn"]
    QUERY --> LOOP["🔄 Duyệt từng tài liệu"]
    LOOP --> CHECK{"Đã gửi nhắc<br/>hôm nay?"}
    CHECK -->|"Rồi"| SKIP["⏭️ Bỏ qua"]
    CHECK -->|"Chưa"| SEND["📧 Gửi Email nhắc nhở<br/>📱 Gửi Zalo notification"]
    SEND --> LOG["📝 Ghi log nhắc nhở"]
    LOG --> LOOP

    style CRON fill:#e17055,color:#fff
    style SEND fill:#0984e3,color:#fff
```

---

## 6. Tổng Quan Module Chức Năng

### 🌐 Module Client Portal

| # | Module | Chức năng chính | Ghi chú |
|---|--------|----------------|---------|
| 1 | **Tra cứu tài liệu** | Nhập mã hợp đồng/hóa đơn → hiển thị thông tin & trạng thái | Public, không cần đăng nhập |
| 2 | **Xem trước tài liệu** | Render PDF trên trình duyệt, hiển thị vị trí cần ký | PDF Viewer (pdf.js) |
| 3 | **Tạo chữ ký trực quan** | Vẽ tay trên Canvas (chuột/touch/bút) / Upload ảnh chữ ký (PNG/JPG) / Chọn chữ ký đã lưu | Canvas + Image processing |
| 4 | **Đặt chữ ký vào tài liệu** | Drag & Drop chữ ký vào vị trí trên PDF, resize, xoay | Interactive PDF overlay |
| 5 | **Ký số RSA** | Hash SHA-256 → Ký bằng Private Key RSA-2048 → Đính chữ ký số + ảnh vào PDF | RSA asymmetric signing |
| 6 | **Quản lý RSA Keypair** | Generate keypair lần đầu, lưu Private Key phía client (encrypted), Public Key trên server | RSA-2048 / RSA-4096 |
| 7 | **Xác thực người ký** | OTP qua Email/SMS trước khi ký | Bảo mật chống giả mạo |
| 8 | **Tải tài liệu** | Download PDF đã ký hoàn chỉnh, verify chữ ký RSA | Watermark + Timestamp |

---

### 🛡️ Module Admin Portal

| # | Module | Chức năng chính | Ghi chú |
|---|--------|----------------|---------|
| 1 | **Dashboard** | Tổng quan: số tài liệu chờ ký, đã ký, quá hạn, thống kê | Real-time widgets |
| 2 | **Quản lý tài liệu** | CRUD tài liệu, phân loại (Hợp đồng/Hóa đơn), tìm kiếm, lọc | Full-text search |
| 3 | **Upload & Batch Upload** | Upload đơn lẻ hoặc hàng loạt (CSV + ZIP), mapping dữ liệu tự động | Tối ưu cho hóa đơn số lượng lớn |
| 4 | **Phân quyền ký** | Chỉ định người ký cho từng tài liệu, luồng ký tuần tự/đồng thời | Workflow engine |
| 5 | **Quản lý User** | CRUD user, phân role (Admin/Manager/Signer), import user hàng loạt | RBAC |
| 6 | **Thông báo & Nhắc nhở** | Gửi mail thông báo, lập lịch nhắc hàng ngày, quản lý template email | Scheduler + Queue |
| 7 | **Báo cáo & Thống kê** | Thống kê theo thời gian, xuất báo cáo Excel/PDF | Charts & Export |
| 8 | **Quản lý thời hạn** | Đặt deadline, cảnh báo sắp hết hạn, đánh dấu quá hạn | Auto-escalation |
| 9 | **Audit Log** | Ghi nhận mọi hành động: ai ký, lúc nào, IP, thiết bị | Compliance & Traceability |
| 10 | **Quản lý Audit Log** | Dashboard log, tìm kiếm/lọc, cảnh báo bảo mật, xuất báo cáo, retention policy | Admin only |

---

### ✍️ Module Ký Văn Bản & Hợp Đồng Điện Tử

| # | Module | Chức năng chính | Ghi chú |
|---|--------|----------------|---------|
| 1 | **Tạo chữ ký** | Vẽ tay trên Canvas (chuột/touch/stylus) hoặc upload ảnh chữ ký (PNG/JPG/SVG), tự động xóa nền trắng | Canvas API + Image processing |
| 2 | **Quản lý chữ ký** | Lưu nhiều chữ ký mẫu, đặt mặc định, xóa, đổi tên. Mỗi user có thể có nhiều mẫu chữ ký | Gallery chữ ký cá nhân |
| 3 | **Vị trí ký trên tài liệu** | Drag & drop đặt chữ ký vào PDF, resize/xoay, preview trước khi ký, hỗ trợ nhiều vị trí ký | Interactive PDF overlay |
| 4 | **Workflow ký** | Ký tuần tự (lần lượt) hoặc đồng thời (song song), quy định thứ tự ký bắt buộc | Workflow engine |
| 5 | **Xác thực trước ký** | OTP Email/SMS, xác nhận PIN/Biometric trước mỗi lần ký | Multi-factor |
| 6 | **Ký số RSA** | Hash tài liệu SHA-256 → Ký bằng Private Key RSA → Embed chữ ký trực quan + chữ ký số vào PDF | RSA-2048 / Web Crypto API |
| 7 | **Xác minh chữ ký** | Verify chữ ký RSA bằng Public Key, kiểm tra tính toàn vẹn tài liệu, hiển thị trạng thái hợp lệ | Server-side verify |
| 8 | **Lưu trữ & chứng thực** | Lưu PDF đã ký + metadata (IP, device, timestamp, certificate chain), hỗ trợ tải về | Compliance & Traceability |

> [!TIP]
> **Chữ ký kết hợp 2 lớp:**
> - **Lớp trực quan** — Ảnh chữ ký tay / upload (hiển thị trên PDF cho dễ nhận biết)
> - **Lớp bảo mật** — Chữ ký số RSA mã hóa (đảm bảo tính pháp lý, toàn vẹn, chống chối bỏ)
> - Cả 2 lớp được embed đồng thời vào PDF khi ký

---

### 🧾 Module Quản Lý Hóa Đơn Điện Tử (E-Invoice)

| # | Module | Chức năng chính | Ghi chú |
|---|--------|----------------|---------|
| 1 | **Tạo hóa đơn** | Form nhập fields cơ bản theo quy định Cục Thuế: MST, tên bên bán/mua, hàng hóa, thuế suất, tổng tiền | Validate theo TT78 |
| 2 | **Tạo XML thuế** | Tự động generate XML theo cấu trúc chuẩn của Cục Thuế từ dữ liệu nhập | XML builder engine |
| 3 | **Ký số XML** | Ký chứng thư số doanh nghiệp lên XML trước khi gửi | HSM / USB Token |
| 4 | **Gửi & nhận kết quả thuế** | Push XML lên API Cục Thuế → Polling 1-2 phút → Nhận mã CQT hoặc mã lỗi | Async + Retry |
| 5 | **Quản lý hóa đơn** | Danh sách, tìm kiếm, lọc theo trạng thái (Nháp / Đã gửi / Cấp mã / Từ chối / Đã hủy) | CRUD + Filters |
| 6 | **Batch tạo hóa đơn** | Import Excel/CSV → Tạo hàng loạt hóa đơn → Gửi thuế batch | Queue + Worker Pool |
| 7 | **Hủy / Điều chỉnh / Thay thế** | Tạo hóa đơn hủy, điều chỉnh tăng/giảm, thay thế theo quy định | Theo NĐ123 |
| 8 | **Gửi hóa đơn cho khách** | Gửi PDF hóa đơn chính thức qua Email / Zalo / SMS cho người mua | Auto sau khi có mã CQT |
| 9 | **Báo cáo hóa đơn** | Tổng hợp theo kỳ, bảng kê hóa đơn, xuất XML báo cáo thuế | Export Excel/XML |

---

### ⚙️ Module Backend Services

| # | Service | Chức năng | Tech gợi ý |
|---|---------|-----------|-------------|
| 1 | **API Gateway** | Routing, rate limiting, load balancing | Nginx / Kong |
| 2 | **Auth Service** | JWT, OAuth2, OTP, RBAC | Keycloak / Custom |
| 3 | **Document Service** | Upload, lưu trữ, render PDF, quản lý metadata | pdf-lib, sharp |
| 4 | **Signing Service** | RSA keypair, hash SHA-256, ký/verify RSA, xử lý ảnh chữ ký (vẽ tay/upload), embed vào PDF | node-forge, pdf-lib, Web Crypto API, sharp |
| 5 | **Invoice Service** | Tạo XML thuế, ký số XML, gửi/nhận kết quả Cục Thuế, quản lý hóa đơn | xml2js, node-forge, axios |
| 6 | **Notification Service** | Gửi Email (SMTP), Zalo OA API, push notification | Nodemailer, Zalo OA SDK |
| 7 | **Batch Processing** | Xử lý hàng loạt hóa đơn + ký, queue-based async processing | Bull/BullMQ + Redis |
| 8 | **Scheduler Service** | CRON nhắc ký, kiểm tra deadline, polling kết quả thuế | node-cron / Agenda |
| 9 | **User Service** | CRUD user, quản lý profile, phân quyền | Custom |
| 10 | **Audit Service** | Ghi log hành động, truy vết | Elasticsearch (optional) |

---

### 💾 Module Data & Storage

| # | Component | Mục đích | Tech gợi ý |
|---|-----------|----------|-------------|
| 1 | **Database** | Lưu metadata tài liệu, user, lịch sử ký | PostgreSQL |
| 2 | **File Storage** | Lưu file PDF gốc & đã ký | MinIO / AWS S3 |
| 3 | **Cache** | Cache tra cứu, session, rate limiting | Redis |
| 4 | **Message Queue** | Async processing cho batch & notification | RabbitMQ / Redis Stream |

---

## 7. Batch Processing - Tối Ưu Cho Hóa Đơn Số Lượng Lớn

```mermaid
flowchart TD
    UPLOAD["📤 Admin upload batch<br/>CSV + ZIP chứa hóa đơn"] --> PARSE["🔍 Parse CSV<br/>Extract metadata"]
    PARSE --> VALIDATE_B["✅ Validate dữ liệu<br/>• Kiểm tra format<br/>• Kiểm tra trùng mã"]
    VALIDATE_B --> SPLIT["✂️ Chia thành chunks<br/>100 files/chunk"]
    SPLIT --> QUEUE_B["📨 Đẩy vào Queue<br/>Mỗi chunk = 1 job"]
    QUEUE_B --> WORKERS["⚡ Worker Pool<br/>Xử lý song song"]
    WORKERS --> PROCESS["Mỗi Worker xử lý:<br/>1. Lưu file vào Storage<br/>2. Tạo record trong DB<br/>3. Gán người ký<br/>4. Gửi notification"]
    PROCESS --> PROGRESS["📊 Cập nhật progress<br/>Real-time via WebSocket"]
    PROGRESS --> COMPLETE_B(["✅ Hoàn tất batch<br/>Thông báo Admin"])

    style UPLOAD fill:#6c5ce7,color:#fff
    style WORKERS fill:#e17055,color:#fff
    style COMPLETE_B fill:#00b894,color:#fff
```

> [!TIP]
> **Chiến lược tối ưu Batch Processing:**
> - Chia nhỏ batch thành chunks (100 files/chunk) để xử lý song song
> - Sử dụng Worker Pool với số lượng worker có thể scale
> - Bulk insert vào DB thay vì insert từng record
> - Hiển thị progress real-time qua WebSocket
> - Retry mechanism cho các job thất bại

---

## 8. Batch Hóa Đơn Điện Tử - Gửi Thuế Hàng Loạt

```mermaid
flowchart TD
    IMPORT["📥 Import Excel/CSV<br/>Danh sách hóa đơn"] --> PARSE_INV["🔍 Parse & Validate<br/>Từng dòng = 1 hóa đơn"]
    PARSE_INV --> PREVIEW["👁️ Preview danh sách<br/>Hiển thị lỗi nếu có"]
    PREVIEW --> CONFIRM_B{"Xác nhận tạo batch?"}
    CONFIRM_B -->|"Sửa lại"| IMPORT
    CONFIRM_B -->|"Xác nhận"| GEN_BATCH["🔧 Generate XML batch<br/>Tạo XML cho từng hóa đơn"]

    GEN_BATCH --> SIGN_BATCH["🔐 Batch ký số<br/>Ký tất cả XML"]
    SIGN_BATCH --> QUEUE_TAX["📨 Đẩy vào Queue<br/>Rate limiting theo API thuế"]
    QUEUE_TAX --> SEND_TAX["📤 Gửi lên Cục Thuế<br/>Tuần tự hoặc batch API"]
    SEND_TAX --> POLL_BATCH["🔄 Polling kết quả batch<br/>Theo dõi từng hóa đơn"]

    POLL_BATCH --> RESULT{"Kết quả?"}
    RESULT -->|"✅ Thành công"| OK["Cập nhật mã CQT<br/>Tạo PDF chính thức"]
    RESULT -->|"❌ Lỗi"| ERR["Đánh dấu lỗi<br/>Log chi tiết"]
    OK --> PROGRESS_B["📊 Dashboard progress<br/>X/Y hoàn tất"]
    ERR --> PROGRESS_B
    PROGRESS_B --> ALL_DONE{"Tất cả xong?"}
    ALL_DONE -->|"Chưa"| POLL_BATCH
    ALL_DONE -->|"Xong"| SUMMARY(["📋 Báo cáo tổng hợp<br/>Thành công / Lỗi / Retry"])

    style IMPORT fill:#6c5ce7,color:#fff
    style GEN_BATCH fill:#0984e3,color:#fff
    style SIGN_BATCH fill:#00b894,color:#fff
    style SEND_TAX fill:#fdcb6e,color:#000
    style SUMMARY fill:#00b894,color:#fff
```

> [!WARNING]
> **Lưu ý khi gửi thuế hàng loạt:**
> - API Cục Thuế có **rate limit**, cần tuân thủ giới hạn số request/phút
> - Mỗi hóa đơn cần **polling riêng** để nhận kết quả (1-2 phút/hóa đơn)
> - Cần cơ chế **retry với exponential backoff** cho các trường hợp timeout
> - Nên gửi theo batch nhỏ (50-100 hóa đơn) và chờ kết quả trước khi gửi batch tiếp

---

## 9. Database Schema Tổng Quan (ERD)

```mermaid
erDiagram
    USERS {
        uuid id PK
        string email
        string phone
        string full_name
        string role "admin | manager | signer | guest"
        string zalo_id
        text rsa_public_key "RSA Public Key (PEM)"
        string key_algorithm "RSA-2048 | RSA-4096"
        timestamp key_generated_at "Ngày tạo keypair"
        timestamp created_at
    }

    DOCUMENTS {
        uuid id PK
        string code UK "Mã tra cứu"
        string title
        string type "contract | invoice"
        string status "draft | pending | partially_signed | completed | expired"
        uuid uploaded_by FK
        string file_path
        string signed_file_path
        timestamp deadline
        timestamp created_at
    }

    INVOICES {
        uuid id PK
        string invoice_symbol "Ký hiệu hóa đơn"
        string invoice_number UK "Số hóa đơn"
        string template_code "Mã mẫu hóa đơn"
        string seller_tax_code "MST bên bán"
        string seller_name "Tên bên bán"
        string seller_address "Địa chỉ bên bán"
        string buyer_tax_code "MST bên mua"
        string buyer_name "Tên bên mua"
        string buyer_address "Địa chỉ bên mua"
        string payment_method "Hình thức thanh toán"
        decimal total_before_tax "Tổng tiền trước thuế"
        decimal vat_amount "Tiền thuế GTGT"
        decimal total_amount "Tổng thanh toán"
        string status "draft | submitted | issued | rejected | cancelled | adjusted | replaced"
        string tax_authority_code "Mã CQT"
        string xml_file_path "Path XML đã tạo"
        string pdf_file_path "Path PDF chính thức"
        uuid created_by FK
        uuid document_id FK "Link tới document nếu cần ký"
        timestamp issued_date
        timestamp created_at
    }

    INVOICE_ITEMS {
        uuid id PK
        uuid invoice_id FK
        int line_number "STT"
        string item_name "Tên hàng hóa dịch vụ"
        string unit "Đơn vị tính"
        decimal quantity "Số lượng"
        decimal unit_price "Đơn giá"
        decimal vat_rate "Thuế suất"
        decimal amount "Thành tiền"
    }

    INVOICE_TAX_SUBMISSIONS {
        uuid id PK
        uuid invoice_id FK
        string xml_content "Nội dung XML gửi đi"
        string submission_id "Mã giao dịch với thuế"
        string status "pending | processing | success | error"
        string tax_response_code "Mã phản hồi từ thuế"
        string tax_error_message "Mô tả lỗi nếu có"
        string tax_authority_code "Mã CQT được cấp"
        int retry_count "Số lần retry"
        timestamp submitted_at
        timestamp responded_at
    }

    SIGNING_ASSIGNMENTS {
        uuid id PK
        uuid document_id FK
        uuid signer_id FK
        int sign_order "Thứ tự ký"
        string status "pending | signed | rejected | expired"
        timestamp signed_at
        string signature_type "draw | upload | saved"
        string signature_image_path "Path ảnh chữ ký trực quan"
        text rsa_signature "Chữ ký RSA mã hóa (Base64)"
        string document_hash "SHA-256 hash của tài liệu"
        string hash_algorithm "SHA-256"
        boolean signature_verified "Server đã verify?"
        jsonb signature_position "x y width height page"
        string ip_address
        string device_info
    }

    NOTIFICATIONS {
        uuid id PK
        uuid document_id FK
        uuid user_id FK
        string channel "email | zalo | sms"
        string type "invite | reminder | completed | expired | invoice_issued"
        string status "queued | sent | failed"
        timestamp sent_at
    }

    BATCH_JOBS {
        uuid id PK
        uuid created_by FK
        string batch_type "signing | invoice"
        int total_files
        int processed_files
        int failed_files
        string status "processing | completed | failed"
        timestamp created_at
    }

    AUDIT_LOGS {
        uuid id PK
        uuid user_id FK
        uuid document_id FK
        uuid invoice_id FK
        string action "view | sign | download | upload | assign | create_invoice | submit_tax"
        string ip_address
        jsonb metadata
        timestamp created_at
    }

    USERS ||--o{ DOCUMENTS : "uploads"
    USERS ||--o{ SIGNING_ASSIGNMENTS : "signs"
    USERS ||--o{ INVOICES : "creates"
    DOCUMENTS ||--o{ SIGNING_ASSIGNMENTS : "has"
    DOCUMENTS ||--o{ NOTIFICATIONS : "triggers"
    DOCUMENTS ||--o{ INVOICES : "linked_invoice"
    INVOICES ||--o{ INVOICE_ITEMS : "contains"
    INVOICES ||--o{ INVOICE_TAX_SUBMISSIONS : "submitted_to"
    INVOICES ||--o{ AUDIT_LOGS : "tracked_in_inv"
    USERS ||--o{ NOTIFICATIONS : "receives"
    USERS ||--o{ BATCH_JOBS : "creates"
    USERS ||--o{ AUDIT_LOGS : "performs"
    DOCUMENTS ||--o{ AUDIT_LOGS : "tracked_in"
```

---

## 10. Tóm Tắt Tổng Quan

| Hệ thống | Số Module | Mục đích chính |
|-----------|-----------|----------------|
| **Client Portal** | 8 modules | Tra cứu, xem, tạo chữ ký, ký RSA, tải tài liệu |
| **Ký Văn Bản & HĐ ĐT** | 8 modules | Tạo/quản lý chữ ký, workflow ký, verify, lưu trữ |
| **Admin Portal** | 10 modules | Quản lý toàn bộ quy trình ký + audit log |
| **E-Invoice (Hóa đơn ĐT)** | 9 modules | Tạo, quản lý, gửi thuế hóa đơn điện tử |
| **Backend Services** | 10 services | Xử lý logic nghiệp vụ |
| **Data & Storage** | 4 components | Lưu trữ & caching |
| **External Integration** | 1 system | Tích hợp API Cục Thuế |
| **Tổng cộng** | **50 thành phần** | Hệ thống E-Signed hoàn chỉnh |

> [!IMPORTANT]
> **Ưu tiên phát triển gợi ý:**
> 1. **Phase 1** — Auth + Signing Service + Client Portal (tạo chữ ký + ký RSA đơn lẻ)
> 2. **Phase 2** — Admin Portal + User Management + Workflow ký + Notification
> 3. **Phase 3** — Invoice Service + Tạo XML + Tích hợp API Cục Thuế
> 4. **Phase 4** — Batch Processing (ký + hóa đơn) + Scheduler
> 5. **Phase 5** — Audit Log + Reports + Tối ưu hiệu suất
