# WF-04 — Kiểm kê và điều chỉnh tồn kho (triển khai sau)

> Trạng thái: `later-independent-stock-control-flow`.
>
> Đây là luồng kiểm soát tồn kho độc lập, không thuộc chuỗi vận hành chính Receipt/Delivery/Internal Transfer. Giữ mapping ở đây để triển khai sau; không dùng làm precondition của các flow chính.

## 1. Mục đích nghiệp vụ

Đối chiếu số lượng thực tế tại location `internal`/`transit` với số lượng hệ thống và ghi nhận chênh lệch bằng một stock move đối ứng với location điều chỉnh tồn (`property_stock_inventory`).

## 2. Đường đi của hàng

| Chênh lệch | Đường đi | Ý nghĩa |
|---|---|---|
| Thực tế lớn hơn hệ thống | `Inventory Loss -> Internal/Transit` | Tăng tồn |
| Thực tế nhỏ hơn hệ thống | `Internal/Transit -> Inventory Loss` | Giảm tồn |

Location điều chỉnh được lấy từ `product.template.property_stock_inventory` theo company; nếu thiếu, code tìm default của `product.template`.

## 3. Các bước nghiệp vụ và mapping code

### 3.1. Mở danh sách cần kiểm kê

1. Người dùng mở Physical Inventory.
2. Hệ thống hiển thị quant thuộc location `internal` hoặc `transit`.
3. Có thể lọc theo product, location, lot/serial, package, owner hoặc người được giao đếm.

Mapping:

```text
stock.quant.action_view_inventory()
  -> context inventory_mode=True, no_at_date=True
  -> domain location_id.usage in ('internal', 'transit')
  -> view stock.view_stock_quant_tree_inventory_editable
```

XML:

- `stock.stock_quant_action` — danh sách tồn kho.
- `stock.view_stock_quant_tree_inventory_editable` — danh sách nhập số đếm.

### 3.2. Ghi số lượng đếm

1. Người dùng chọn quant hoặc tạo dòng cho product/location chưa có quant.
2. Hệ thống giữ `quantity` là số hệ thống và nhận `inventory_quantity` là số đếm.
3. `inventory_diff_quantity = inventory_quantity - quantity`.
4. Có thể đặt số đếm ban đầu bằng action Set Counted Quantity, đặt về zero hoặc reset dòng.

Mapping:

```text
stock.quant.action_set_inventory_quantity()
stock.quant.action_set_inventory_quantity_zero()
stock.quant.action_clear_inventory_quantity()
stock.quant.action_reset()
stock.quant._compute_inventory_diff_quantity()
```

Các field nghiệp vụ:

```text
stock.quant.quantity                 # tồn lý thuyết
stock.quant.inventory_quantity       # tồn thực tế được đếm
stock.quant.inventory_diff_quantity  # chênh lệch
stock.quant.inventory_quantity_set   # đã nhập số đếm hay chưa
stock.quant.is_outdated               # tồn thay đổi sau thời điểm đếm
```

### 3.3. Áp dụng một hoặc nhiều dòng

1. Người dùng chọn Apply hoặc Apply All.
2. Nếu quant không bị thay đổi sau lúc đếm, hệ thống tạo move điều chỉnh và validate ngay.
3. Nếu quant đã bị thay đổi, hệ thống mở màn hình xử lý conflict.
4. Sau khi áp dụng, số đếm được xóa trạng thái chờ; ngày kiểm kê location được cập nhật.

Mapping:

```text
stock.quant.action_apply_inventory(date=None)
  -> nếu is_outdated:
       stock.inventory.conflict
     ngược lại:
       stock.quant._apply_inventory(date)

stock.inventory.adjustment.name.action_apply()
  -> stock.quant.action_apply_inventory(counting_date)

stock.inventory.conflict.action_keep_counted_quantity()
  -> cập nhật inventory_diff_quantity
  -> stock.quant.action_apply_inventory()

stock.inventory.conflict.action_keep_difference()
  -> cập nhật inventory_quantity
  -> stock.quant.action_apply_inventory()
```

XML:

- `stock.stock_inventory_adjustment_name_form_view` — nhập lý do và ngày kiểm kê.
- `stock.stock_inventory_conflict_form_view` — xử lý tồn đã thay đổi.
- `stock.inventory_warning_set_view` — cảnh báo dòng đã có số đếm.
- `inventory_warning_reset_view` — cảnh báo khi reset số đếm.

### 3.4. Tạo và hoàn tất move điều chỉnh

Với mỗi quant có chênh lệch khác zero:

```text
inventory_diff > 0:
  Inventory Loss -> quant.location_id

inventory_diff < 0:
  quant.location_id -> Inventory Loss
```

Mapping runtime:

```text
stock.quant._apply_inventory()
  -> stock.quant._get_inventory_move_values(...)
  -> stock.move.create(..., is_inventory=True)
  -> stock.move._action_done()
      -> stock.move.line._action_done()
      -> stock.move.line._synchronize_quant()
      -> stock.quant._update_available_quantity()
  -> cập nhật inventory_date
  -> stock.quant.action_clear_inventory_quantity()
```

Move điều chỉnh được tạo với `inventory_mode=False`; context `ignore_dest_packages=True` được dùng khi hoàn tất. `inventory_name` và `counting_date` được truyền từ wizard để truy vết lý do và thời điểm kiểm kê.

## 4. Nhánh và lỗi nghiệp vụ

| Tình huống | Kết quả/code |
|---|---|
| Không có chênh lệch | Không tạo move khi diff bằng zero trong nhánh inverse quantity. |
| Location không phải `internal`/`transit` | Không thuộc danh sách Physical Inventory. |
| Quant đã thay đổi sau lúc đếm | Mở `stock.inventory.conflict`, chọn giữ số đếm hoặc giữ chênh lệch. |
| Serial có tồn không phù hợp | Quant/lot được kiểm tra theo tracking và product trước khi áp dụng. |
| Sửa product/location/lot/package/owner trực tiếp trong inventory mode | Bị chặn bởi `stock.quant.write()`. |
| Đảo ngược một inventory move đã done | Gọi `stock.move.line.action_revert_inventory()`, tạo move ngược và validate. |

## 5. Kết quả sau flow

- Quant tại location thực tế có `quantity` bằng số đã đếm.
- Move và move line điều chỉnh ở trạng thái `done`, có `is_inventory=True`.
- Location được cập nhật `last_inventory_date` và ngày kiểm kê tiếp theo.
- Lịch sử điều chỉnh xem qua `stock.quant.action_inventory_history()`.

## 6. Test tham chiếu

- `addons/stock/tests/test_inventory.py`: `test_inventory_1` đến `test_inventory_7`, counted quantity, conflict/outdated, cyclic inventory.
- `addons/stock/tests/test_quant_inventory_mode.py`: `test_revert_inventory_adjustment`, `test_multi_revert_inventory_adjustment`, `test_set_inventory_quant_to_zero`, `test_revert_inventory_package_quant`.
- `addons/stock/tests/test_quant.py`: `test_inventory_adjustment_package`, `test_reservation_preserved_after_relocation`.
- `addons/stock/tests/test_warehouse.py`: `test_inventory_adjustment_and_negative_quants_1`, `test_inventory_adjustment_and_negative_quants_2`.
