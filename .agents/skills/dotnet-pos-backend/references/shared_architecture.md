# Kiến trúc Thư mục Dùng chung (`Shared/`)

Tài liệu tham khảo các mẫu code chuẩn cho module `Shared/` trong hệ thống .NET Core Web API phục vụ Flutter Mobile App.

---

## 1. Response Models (`Shared/Responses/`)

### 1.1 `ApiResponse<T>`
```csharp
namespace StoreBackend.Shared.Responses;

public record ApiResponse<T>(
    bool Success,
    string Message,
    T? Data = default,
    List<string>? Errors = null
)
{
    public static ApiResponse<T> Ok(T data, string message = "Success") 
        => new(true, message, data);

    public static ApiResponse<T> Fail(string message, List<string>? errors = null) 
        => new(false, message, default, errors);
}
```

### 1.2 `PagedResult<T>`
```csharp
namespace StoreBackend.Shared.Responses;

public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int PageNumber,
    int PageSize
)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => PageNumber < TotalPages;
    public bool HasPreviousPage => PageNumber > 1;
}
```

---

## 2. Exceptions & Middleware (`Shared/Exceptions/`)

### 2.1 Custom Exceptions
```csharp
namespace StoreBackend.Shared.Exceptions;

public abstract class AppException(string message, int statusCode = 400) : Exception(message)
{
    public int StatusCode { get; } = statusCode;
}

public class NotFoundException(string message) : AppException(message, 404);
public class BadRequestException(string message) : AppException(message, 400);
public class ConcurrencyConflictException(string message) : AppException(message, 409);
```

### 2.2 Global Exception Handling Middleware
```csharp
using System.Text.Json;
using StoreBackend.Shared.Responses;

namespace StoreBackend.Shared.Exceptions;

public class GlobalExceptionHandlingMiddleware(RequestDelegate next, ILogger<GlobalExceptionHandlingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Lỗi chưa được xử lý (Unhandled Exception): {Message}", ex.Message);
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        var (statusCode, message) = exception switch
        {
            AppException appEx => (appEx.StatusCode, appEx.Message),
            _ => (StatusCodes.Status500InternalServerError, "Đã xảy ra lỗi máy chủ nội bộ.")
        };

        context.Response.StatusCode = statusCode;
        var response = ApiResponse<object>.Fail(message);
        
        var json = JsonSerializer.Serialize(response, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });

        await context.Response.WriteAsync(json);
    }
}
```

---

## 3. Helpers & Utilities (`Shared/Helpers/`)

```csharp
using System.Globalization;

namespace StoreBackend.Shared.Helpers;

public static class StoreHelpers
{
    // Luôn chuẩn hóa thời gian về UTC
    public static DateTime UtcNow() => DateTime.UtcNow;

    // Sinh mã hóa đơn: HD-yyyyMMddHHmmss-Random
    public static string GenerateInvoiceCode() 
        => $"HD-{DateTime.UtcNow:yyyyMMddHHmmss}-{Random.Shared.Next(100, 999)}";

    // Format tiền tệ Việt Nam Đồng (VNĐ)
    public static string FormatVnd(decimal amount) 
        => string.Format(new CultureInfo("vi-VN"), "{0:C0}", amount);
}
```

---

## 4. Hằng số Hệ thống (`Shared/Constants/`)

```csharp
namespace StoreBackend.Shared.Constants;

public static class AppRoles
{
    public const string Admin = "Admin";
    public const string Manager = "Manager";
    public const string Cashier = "Cashier";
    public const string InventoryStaff = "InventoryStaff";
}

public static class InvoiceStatus
{
    public const string Pending = "Pending";
    public const string Completed = "Completed";
    public const string Cancelled = "Cancelled";
    public const string Refunded = "Refunded";
}
```
