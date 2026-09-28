# Mẫu Triển khai Endpoint (Quy tắc 4 Phần)

Mỗi khi tạo mới một endpoint API cho ứng dụng Flutter, tuân thủ cấu trúc 4 phần chuẩn sau:

---

## Ví dụ: API Thanh toán Hóa đơn tại quầy POS (`POST /api/v1/invoices/checkout`)

### 1. HTTP Method & URL Route
- **Method:** `POST`
- **Route:** `/api/v1/invoices/checkout`
- **Quyền truy cập:** `Authorize(Roles = AppRoles.Cashier)`

---

### 2. Request DTO
```csharp
namespace StoreBackend.Features.Invoices.DTOs;

public record CheckoutItemDto(
    Guid ProductId,
    int Quantity,
    decimal UnitPrice
);

public record CheckoutRequest(
    List<CheckoutItemDto> Items,
    string PaymentMethod, // Cash, Transfer, Card
    decimal ReceivedAmount
);
```

---

### 3. Service Logic
```csharp
namespace StoreBackend.Features.Invoices.Services;

public class InvoiceService(AppDbContext dbContext) : IInvoiceService
{
    public async Task<InvoiceDetailDto> CheckoutAsync(Guid cashierId, CheckoutRequest request, CancellationToken ct = default)
    {
        // 1. Kiểm tra & khóa tồn kho (Pessimistic / Optimistic concurrency)
        var productIds = request.Items.Select(x => x.ProductId).ToList();
        var products = await dbContext.Products
            .Where(p => productIds.Contains(p.Id))
            .ToListAsync(ct);

        // 2. Tính toán tổng tiền & giảm tồn kho
        decimal totalAmount = 0;
        var invoice = new Invoice
        {
            Id = Guid.NewGuid(),
            InvoiceCode = StoreHelpers.GenerateInvoiceCode(),
            CashierId = cashierId,
            CreatedAt = StoreHelpers.UtcNow(),
            Status = InvoiceStatus.Completed,
            PaymentMethod = request.PaymentMethod
        };

        foreach (var item in request.Items)
        {
            var product = products.FirstOrDefault(p => p.Id == item.ProductId)
                ?? throw new NotFoundException($"Sản phẩm với ID {item.ProductId} không tồn tại.");

            if (product.StockQuantity < item.Quantity)
                throw new BadRequestException($"Sản phẩm '{product.Name}' không đủ tồn kho (Còn: {product.StockQuantity}).");

            product.StockQuantity -= item.Quantity;
            totalAmount += item.Quantity * item.UnitPrice;

            invoice.Items.Add(new InvoiceItem
            {
                ProductId = product.Id,
                Quantity = item.Quantity,
                UnitPrice = item.UnitPrice
            });
        }

        invoice.TotalAmount = totalAmount;
        dbContext.Invoices.Add(invoice);

        await dbContext.SaveChangesAsync(ct);

        // 3. Trả về DTO
        return new InvoiceDetailDto(
            invoice.Id,
            invoice.InvoiceCode,
            invoice.TotalAmount,
            StoreHelpers.FormatVnd(invoice.TotalAmount),
            invoice.CreatedAt
        );
    }
}
```

---

### 4. Response Model cho Mobile (Flutter)
```csharp
namespace StoreBackend.Features.Invoices.DTOs;

public record InvoiceDetailDto(
    Guid InvoiceId,
    string InvoiceCode,
    decimal TotalAmount,
    string FormattedTotalAmount,
    DateTime CreatedAtUtc
);
```

**JSON Payload thực tế trả về:**
```json
{
  "success": true,
  "message": "Thanh toán hóa đơn thành công.",
  "data": {
    "invoiceId": "d3b07384-d113-4a15-b778-9e67cfcf86b1",
    "invoiceCode": "HD-20260928140000-842",
    "totalAmount": 150000.0,
    "formattedTotalAmount": "150.000 ₫",
    "createdAtUtc": "2026-09-28T07:00:00Z"
  },
  "errors": null
}
```
