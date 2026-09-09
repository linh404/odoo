# Odoo Inventory Core — nghiệp vụ nội bộ kho và mapping code

Tài liệu mô tả **luồng nghiệp vụ nội bộ kho** theo cách đọc BA, đồng thời map 1:1 sang model, method, XML ID và test của Odoo 19.

## 1. Phạm vi đợt triển khai

Trục chính của tài liệu là ba hoạt động:

- Nhập kho (Receipt).
- Xuất kho (Delivery).
- Điều chuyển nội bộ (Internal Transfer).

Cấu hình, master data, action và code mapping đều được tổ chức xoay quanh ba flow này.

Nghiệp vụ Mua hàng, Bán hàng, Sản xuất và các phần tích hợp liên quan sẽ được bổ sung ở các giai đoạn tiếp theo. Trong đợt này, Receipt/Delivery chỉ đi tới mức stock operation giữa virtual location và internal location; chưa mô tả chứng từ nguồn của các phân hệ đó.

`Inventory Adjustment` được giữ riêng như một flow kiểm soát tồn độc lập để triển khai sau; không phải precondition của ba flow chính.

## 2. Chuỗi xử lý dùng chung

```text
Master data
    -> Tạo picking và stock move
    -> Confirm
    -> Reserve / Check Availability
    -> Nhập chi tiết move line
       (quantity, lot/serial, package, owner)
    -> Validate
    -> Ghi nhận stock move done
    -> Cập nhật quant source/destination
```

## 3. Bản đồ ba flow chính

```text
Receipt:
Supplier -> Input/QC -> Internal Stock

Delivery:
Internal Stock -> Pick/Pack/Output -> Customer

Internal Transfer:
Internal A -> Transit/Intermediate -> Internal B
```

## 4. Tài liệu

| File | Nội dung |
|---|---|
| [`01_master_data.md`](01_master_data.md) | Master data dùng chung: warehouse, location, operation type, product/UoM, lot/serial, package, owner, route/rule, putaway, removal, quant. |
| [`03_receipt_flow.md`](03_receipt_flow.md) | Luồng nhập kho 1/2/3 bước, master data sử dụng, Mermaid nghiệp vụ, code sequence, action, nhánh backorder/return/scrap và mapping code. |
| [`04_delivery_flow.md`](04_delivery_flow.md) | Luồng xuất kho 1/2/3 bước, reservation/removal, master data sử dụng, Mermaid nghiệp vụ, code sequence, action, nhánh backorder/return/scrap và mapping code. |
| [`05_internal_transfer_flow.md`](05_internal_transfer_flow.md) | Luồng điều chuyển nội bộ, transit/multi-step, master data sử dụng, Mermaid nghiệp vụ, code sequence, action và mapping code. |
| [`06_inventory_adjustment_flow.md`](06_inventory_adjustment_flow.md) | Flow kiểm kê/điều chỉnh tồn độc lập, đánh dấu triển khai sau. |

## 5. Quy ước mapping code

- Tên nghiệp vụ dùng tiếng Việt; technical identifier giữ nguyên tên Odoo.
- Model ghi theo technical name, ví dụ `stock.picking`, `stock.warehouse`, `stock.move`.
- Mỗi flow mô tả theo thứ tự: đường đi → master data → bước nghiệp vụ → Mermaid activity → code sequence → nhánh/action → kết quả → mapping model/method/XML/test.
- Source chuẩn nằm trong mã nguồn Odoo 19 tại `addons/stock`; tài liệu là bản đồ nghiệp vụ ↔ code để tra cứu và cập nhật cùng source.

## 6. Cấu trúc thư mục

```text
README.md
01_master_data.md
03_receipt_flow.md
04_delivery_flow.md
05_internal_transfer_flow.md
06_inventory_adjustment_flow.md  # triển khai sau
```
