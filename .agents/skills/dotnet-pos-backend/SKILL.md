---
name: dotnet-pos-backend
description: >-
  Hướng dẫn kiến trúc và quy trình phát triển Backend Web API (.NET Core) phục vụ ứng dụng di động Flutter quản lý cửa hàng tiện lợi (POS, kiểm kho, hóa đơn, mã vạch). Kích hoạt skill này khi người dùng yêu cầu thiết kế kiến trúc, xây dựng Web API, module Shared, tối ưu EF Core, xác thực JWT hoặc triển khai các tính năng bán hàng và kho.
---

# Hướng dẫn Phát triển Backend .NET Core & Web API cho Quản lý Cửa hàng Tiện lợi

Skill này đóng vai trò hướng dẫn kỹ thuật và tiêu chuẩn hóa quy trình phát triển cho **Senior Backend Developer & Kỹ sư Kiến trúc phần mềm (.NET Core & API Design)** phục vụ ứng dụng di động Flutter.

---

## 1. Bối cảnh & Ràng buộc Dự án

- **Hệ thống:** Backend Web API (.NET Core) quản lý cửa hàng tiện lợi, đồng bộ dữ liệu thời gian thực và cung cấp RESTful API cho ứng dụng di động Flutter (POS quầy, quét mã vạch kiểm kho, nhập/xuất tồn, quản lý & in hóa đơn).
- **Client chính:** Mobile App Flutter (cần payload tối ưu, serialization nhất quán).
- **Thời hạn (Timeline):** 8 tuần (chia đều 4 sprints, 2 tuần/sprint).
- **Ngân sách hạ tầng:** Tối đa 52 triệu VNĐ (tối ưu hóa VPS/cloud, tránh kiến trúc thừa thãi làm tăng chi phí vận hành).

---

## 2. Quy tắc Kiến trúc & Chuẩn Codebase

### 2.1 C# & Entity Framework Core Hiện đại
- Tận dụng cú pháp C# hiện đại: **Primary Constructors**, **File-scoped Namespaces**, **Pattern Matching**, **Expression-bodied Members**.
- Luôn sử dụng `.AsNoTracking()` cho tất cả các truy vấn chỉ đọc (read-only queries) để tối ưu memory và tốc độ.
- Tránh boilerplate, giữ code tinh gọn, rõ ràng, có chú thích inline súc tích tại các đoạn logic nghiệp vụ quan trọng.

### 2.2 Cấu trúc Thư mục Dùng chung (`Shared/`)
Mọi logic dùng chung bắt buộc phải nằm trong thư mục/namespace `Shared/`:
- `Shared/Responses/`: Mẫu response chuẩn gồm `ApiResponse<T>` và `PagedResult<T>`.
- `Shared/Helpers/`: Tiện ích xử lý DateTime UTC, sinh mã barcode/hóa đơn tự động, format tiền tệ VNĐ.
- `Shared/Exceptions/`: Custom Exceptions và `GlobalExceptionHandlingMiddleware`.
- `Shared/Constants/`: Hằng số vai trò (Roles), trạng thái đơn hàng/hóa đơn, trạng thái kho.

> Xem chi tiết mã nguồn mẫu tại [Shared Architecture Reference](./references/shared_architecture.md).

### 2.3 Tiêu chuẩn Web API cho Mobile (Flutter)
- Định dạng JSON **camelCase** toàn hệ thống.
- Sử dụng đúng **HTTP Status Codes**: `200 OK`, `201 Created`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `500 Internal Server Error`.
- Xác thực: **JWT Bearer Token** gọn nhẹ kèm cơ chế **Refresh Token**.

---

## 3. Quy trình Triển khai Endpoint (Quy tắc 4 Phần)

Mỗi khi thiết kế hoặc cài đặt một endpoint, bắt buộc phải trình bày đủ **4 phần**:

1. **HTTP Method & URL Route**: Rõ ràng, chuẩn RESTful (ví dụ: `POST /api/v1/invoices/checkout`).
2. **Request DTO**: Khai báo record/class ngắn gọn, có validation annotations cần thiết.
3. **Xử lý logic / Service logic**: Tách biệt logic nghiệp vụ, xử lý transaction/concurrency nếu có.
4. **Response Model cho Mobile**: Trả về theo định dạng `ApiResponse<T>` với dữ liệu tối ưu cho Flutter.

> Xem mẫu chuẩn tại [Endpoint Template Reference](./references/endpoint_template.md).

---

## 4. Phong cách Phản hồi & Giao tiếp

- **Ngôn ngữ:** Tiếng Việt, giữ nguyên các thuật ngữ kỹ thuật tiếng Anh trong ngoặc đơn (ví dụ: *Dependency Injection*, *Middleware*, *Concurrency*, *Idempotency*).
- **Tập trung cốt lõi:** Viết code mẫu trực tiếp vào logic chính, không tạo code thừa.
- **Ràng buộc chi phí & tiến độ:** Mọi đề xuất công nghệ hay thư viện thứ 3 phải khả thi trong khung 8 tuần và ngân sách 52 triệu VNĐ.

---

## 5. Quy trình Làm việc (Workflow)

```mermaid
flowchart LR
    A["Khởi tạo Skeleton & Module Shared"] --> B["Module Xác thực (Auth & JWT)"]
    B --> C["Quản lý Sản phẩm & Tồn kho"]
    C --> D["Bán hàng (POS) & Hóa đơn"]
    D --> E["Tối ưu & Đồng bộ Thực tế"]
```

1. **Triển khai cuốn chiếu:**
   - Xây dựng khung dự án (skeleton) và hoàn thiện thư mục `Shared/` trước.
   - Lần lượt phát triển các cụm nghiệp vụ: *Auth* $\rightarrow$ *Sản phẩm/Kho* $\rightarrow$ *Hóa đơn/POS*.
2. **Phản biện & Tối ưu (Critical Thinking):**
   - Sau khi hoàn thành mỗi module, chủ động đặt **1–2 câu hỏi kỹ thuật trọng tâm** để dự phòng rủi ro thực tế (ví dụ: giải quyết tranh chấp tồn kho khi nhiều thu ngân cùng bán, giảm kích thước payload khi mạng 3G/4G chập chờn).
3. **Kiểm soát phạm vi (Scope Control):**
   - Chủ động cảnh báo nếu có yêu cầu phát sinh nguy cơ trễ tiến độ 8 tuần hoặc vượt ngân sách 52 triệu VNĐ, đồng thời đề xuất giải pháp rút gọn khả thi.
