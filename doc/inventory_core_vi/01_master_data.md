# Master data nghiệp vụ kho

## 1. Phạm vi

Tài liệu này gom các master data và dữ liệu nền dùng chung cho các flow Receipt, Delivery và Internal Transfer trong `addons/stock`. Inventory Adjustment được ghi nhận như flow kiểm soát tồn triển khai sau.

Các master data liên quan Purchase, Sales, Manufacturing, Procurement, Orderpoint, Quality, Barcode và Accounting sẽ được nối thêm khi triển khai các phần tương ứng.

## 2. Quan hệ tổng quát

```text
Company
  -> Warehouse
       -> View Location
            -> Internal Locations
       -> Picking Types
       -> Routes / Rules

Product + UoM + Tracking
  -> Stock Move
       -> Stock Move Line
            -> Lot/Serial + Package + Owner
       -> Stock Quant
```

Trong flow:

- Warehouse và location định nghĩa cấu trúc, nguồn và đích.
- Operation type định nghĩa loại tác nghiệp và chính sách xử lý.
- Product/UoM định nghĩa hàng và cách tính số lượng.
- Route/rule, putaway và removal quyết định cách chọn đường đi hoặc nguồn hàng.
- Move/move line ghi nhận nhu cầu và thực thi.
- Quant phản ánh số dư theo từng chiều quản lý tồn.

## 3. Company

Company là phạm vi sở hữu và kiểm soát dữ liệu kho. Warehouse, operation type, product, location, lot, package, move và quant phải nhất quán company theo các ràng buộc của stock.

Mapping sử dụng xuyên suốt:

```text
res.company
  -> company_id trên stock.warehouse, stock.location,
     stock.picking.type, stock.picking, stock.move,
     stock.move.line, stock.lot, stock.quant
```

Location có thể để trống `company_id` để dùng chung giữa company; các location nội bộ của warehouse thường thuộc company của warehouse.

## 4. Warehouse — `stock.warehouse`

Warehouse là cơ sở kho logic. Nó không lưu từng lần nhập/xuất; nó cung cấp cấu trúc và cấu hình để tạo các stock operation.

### 4.1. Dữ liệu nghiệp vụ

- Tên, mã kho, company và địa chỉ.
- View location và stock location.
- Số bước nhập: `one_step`, `two_steps`, `three_steps`.
- Số bước xuất: `ship_only`, `pick_ship`, `pick_pack_ship`.
- Các location trung gian, operation type, sequence, route và rule được sinh theo cấu hình.

### 4.2. Mapping code

```text
Model: stock.warehouse
Class: StockWarehouse
Source: addons/stock/models/stock_warehouse.py

create()
_get_locations_values()
_create_or_update_sequences_and_picking_types()
_create_or_update_route()
```

### 4.3. Đường đi mặc định

```text
Nhập 1 bước: Supplier -> Stock
Nhập 2 bước: Supplier -> Input -> Stock
Nhập 3 bước: Supplier -> Input -> Quality Control -> Stock

Xuất 1 bước: Stock -> Customer
Xuất 2 bước: Stock -> Output/Pick -> Customer
Xuất 3 bước: Stock -> Pick -> Pack -> Customer
```

Quality Control ở đây chỉ là location/bước kho; chi tiết Quality module sẽ được nối thêm ở phần triển khai sau.

## 5. Location — `stock.location`

Location là điểm nguồn hoặc đích của stock move và là chiều chính để tính tồn.

### 5.1. Loại location

| `usage` | Vai trò |
|---|---|
| `supplier` | Nguồn ảo của hàng nhập |
| `customer` | Đích ảo của hàng xuất |
| `internal` | Nơi chứa hàng thuộc kho |
| `view` | Nút nhóm trong cây location; không chứa hàng trực tiếp |
| `inventory` | Location đối ứng cho inventory adjustment/scrap |
| `transit` | Location trung chuyển giữa warehouse/company |
| `production` | Location đối ứng cho production; chi tiết production sẽ được bổ sung sau |

### 5.2. Dữ liệu nghiệp vụ

- Cây cha/con (`location_id`, `child_ids`, `parent_path`).
- Company và warehouse suy ra từ view location.
- Barcode và lịch kiểm kê định kỳ.
- Removal strategy mặc định.
- Putaway rules.
- Storage category.

### 5.3. Mapping code

```text
Model: stock.location
Class: StockLocation
Source: addons/stock/models/stock_location.py

_compute_warehouse_id()
_get_putaway_strategy()
```

Location `view` chỉ tổ chức cây; các quant chỉ được ghi nhận tại location phù hợp để chứa tồn.

## 6. Operation Type — `stock.picking.type`

Operation type là mẫu tác nghiệp để tạo picking Receipt, Delivery hoặc Internal Transfer.

### 6.1. Dữ liệu nghiệp vụ

- `code`: `incoming`, `outgoing`, `internal`.
- Location nguồn/đích mặc định.
- Warehouse, sequence và mã reference.
- Reservation: `at_confirm`, `manual`, `by_date`.
- Cho phép tạo/chọn lot hoặc serial.
- Hiển thị detailed operations.
- Backorder: `ask`, `always`, `never`.
- Shipping policy: `direct`, `one`.
- Operation type dùng cho return.

### 6.2. Mapping code

```text
Model: stock.picking.type
Class: StockPickingType
Source: addons/stock/models/stock_picking.py

create()
code
default_location_src_id
default_location_dest_id
reservation_method
return_picking_type_id
create_backorder
move_type
```

Picking lấy mặc định từ operation type nhưng source/destination thực tế có thể bị route/rule hoặc thao tác người dùng thay đổi.

## 7. Product và UoM

Product là hàng được nhận, lưu, chuyển hoặc loại bỏ. UoM quyết định cách nhập demand/quantity và quy đổi về UoM chuẩn của product.

### 7.1. Dữ liệu nghiệp vụ

- `is_storable`: product có được quản lý tồn hay không.
- `tracking`: `none`, `lot`, `serial`.
- UoM chuẩn và các UoM được phép.
- Route, putaway rule, packaging, weight và volume.
- Quantity hiển thị theo context location/warehouse/lot/owner/package.

### 7.2. Mapping code

```text
Models: product.template, product.product, uom.uom
Stock extension: addons/stock/models/product.py

product.template.is_storable
product.template.tracking
product.product.qty_available
product.product.free_qty
product.product.virtual_available
stock.move.product_uom_qty
stock.move.product_uom
```

Tracking quyết định mức chi tiết cần có trên move line:

```text
none   -> quản lý theo số lượng
lot    -> quản lý theo lô
serial -> quản lý từng serial
```

## 8. Lot/Serial — `stock.lot`

Lot/serial là định danh truy xuất của product có tracking. Nó đi theo move line và trở thành một chiều của quant; không phải một operation độc lập.

### 8.1. Dữ liệu nghiệp vụ

- Tên lot/serial, product và company.
- Số lượng hiện có và location hiện tại.
- Liên kết tới các transfer.
- Lot properties nếu product định nghĩa.

### 8.2. Mapping code

```text
Model: stock.lot
Class: StockLot
Source: addons/stock/models/stock_lot.py

generate_lot_names()
_get_next_serial()
_check_unique_lot()
```

Lot/serial trên move line phải thuộc đúng product; stock kiểm tra điều này trước khi hoàn tất operation.

## 9. Package và Package Type

Package là container chứa quant hoặc package con. Package type mô tả loại container và được dùng khi đóng gói, putaway hoặc kiểm tra capacity.

### 9.1. Dữ liệu nghiệp vụ

- Package reference và cây package cha/con.
- Location hiện tại và destination location.
- Quant/package bên trong.
- Owner suy ra từ nội dung.
- Package type, kích thước và shipping weight.

Move line dùng:

```text
package_id       -> package nguồn
result_package_id -> package đích
```

### 9.2. Mapping code

```text
Model: stock.package
Class: StockPackage
Source: addons/stock/models/stock_package.py

action_put_in_pack()

Model: stock.package.type
Class: StockPackageType
Source: addons/stock/models/stock_package_type.py
```

Package history là dữ liệu hỗ trợ truy vết package, không thay thế move line hoặc quant:

```text
stock.package.history
  -> addons/stock/models/stock_package_history.py
```

## 10. Owner/Consignment — `res.partner`

Owner xác định hàng thuộc đối tác nào khi hàng đang nằm trong location nội bộ. Owner không phải nguồn/đích; đây là chiều sở hữu của tồn.

### 10.1. Mapping code

```text
res.partner
  -> stock.quant.owner_id
  -> stock.move.line.owner_id
  -> stock.picking.owner_id
```

Tính năng được bật qua `stock.group_tracking_owner`, khai báo trên `res.config.settings.group_stock_tracking_owner`.

## 11. Route và Rule

Route là tập hợp rule; rule nối các location và operation type thành các bước của một đường đi.

### 11.1. Dữ liệu nghiệp vụ

- Route gắn với warehouse/product/category khi được chọn.
- Rule xác định source, destination, operation type và thứ tự áp dụng.
- Rule có thể tạo move tiếp theo hoặc thay thế bước hiện tại tùy `auto`.
- `procure_method` có các giá trị MTS/MTO; chi tiết procurement sẽ được bổ sung sau.

### 11.2. Mapping code

```text
Model: stock.route
Class: StockRoute
Source: addons/stock/models/stock_location.py

Model: stock.rule
Class: StockRule
Source: addons/stock/models/stock_rule.py

run()
_run_pull()
_run_push()
```

Trong tài liệu flow kho, route/rule chỉ được dùng để giải thích các bước kho như Input → Stock hoặc Pick → Pack; không mô tả tạo Purchase/MO.

## 12. Putaway Rule — `stock.putaway.rule`

Putaway quyết định location con nơi hàng sẽ được cất khi hàng đến một khu vực. Rule có thể chuyên biệt theo product, category, package type hoặc storage category và có thứ tự ưu tiên.

### Mapping code

```text
Model: stock.putaway.rule
Class: StockPutawayRule
Source: addons/stock/models/product_strategy.py

stock.move.line._apply_putaway_strategy()
stock.location._get_putaway_strategy()
```

Kết quả putaway tác động vào destination location của move line trước khi move được hoàn tất.

## 13. Removal Strategy — `product.removal`

Removal quyết định quant nào được ưu tiên khi cần lấy hàng từ tồn. Nó không thay đổi demand của move.

Các chiến lược thường dùng:

```text
FIFO | LIFO | FEFO | Closest Location | Least Packages
```

### Mapping code

```text
Model: product.removal
Class: ProductRemoval
Source: addons/stock/models/product_strategy.py

stock.quant._get_removal_strategy()
stock.quant._get_removal_strategy_order()
stock.quant._gather()
stock.quant._get_reserve_quantity()
```

## 14. Storage Category — `stock.storage.category`

Storage category mô tả sức chứa và chính sách chứa hàng của location.

### Dữ liệu nghiệp vụ

- Trọng lượng tối đa.
- Cho phép product mới khi location rỗng, cùng product hoặc trộn product.
- Capacity theo product.
- Capacity theo package type.

### Mapping code

```text
Models: stock.storage.category,
        stock.storage.category.capacity
Source: addons/stock/models/stock_storage_category.py
```

Storage category được tham chiếu từ `stock.location` và có thể được dùng bởi putaway strategy.

## 15. Stock Move, Move Line và Quant

Đây là nhóm dữ liệu giao dịch dùng chung cho mọi flow; chúng không phải master data cấu hình nhưng là đích mà master data điều khiển.

### 15.1. Stock Move — `stock.move`

Move là nhu cầu chuyển product từ source location đến destination location.

```text
Source: addons/stock/models/stock_move.py
Class: StockMove

_action_confirm()
_action_assign()
_update_reserved_quantity()
_action_done()
```

Thông tin chính: product, demand quantity, UoM, source/destination, picking, route/rule, availability, reservation, state và upstream/downstream move.

### 15.2. Stock Move Line — `stock.move.line`

Move line ghi nhận chi tiết thực tế của move: quantity, source/destination, lot/serial, source/destination package và owner.

```text
Source: addons/stock/models/stock_move_line.py
Class: StockMoveLine

_apply_putaway_strategy()
_action_done()
_synchronize_quant()
```

### 15.3. Quant — `stock.quant`

Quant phản ánh số dư theo tổ hợp:

```text
product + location + lot + package + owner + company
```

Thông tin chính:

- `quantity`: tồn thực tế.
- `reserved_quantity`: lượng đã giữ.
- `available_quantity`: `quantity - reserved_quantity`.
- Inventory counted quantity và difference.

```text
Source: addons/stock/models/stock_quant.py
Class: StockQuant

_get_reserve_quantity()
_update_reserved_quantity()
_update_available_quantity()
_apply_inventory()
```

Move line done cập nhật quant nguồn và quant đích; reservation giữ trên quant nguồn cho tới khi move được hoàn tất hoặc unreserve.

## 16. Quan hệ dùng trong flow

```text
Warehouse
  -> view_location_id / lot_stock_id
  -> picking types
  -> routes / rules

Picking Type
  -> default source/destination
  -> reservation/backorder policy
  -> Picking

Picking
  -> Stock Moves
  -> Move Lines

Stock Move
  -> source/destination Location
  -> demand / reservation
  -> Move Lines

Move Line
  -> Product / UoM
  -> Lot/Serial / Package / Owner
  -> updates Quant

Quant
  -> Product / Location / Lot / Package / Owner
  -> source for reservation
```

## 17. Cấu hình tính năng ảnh hưởng master data

Các feature switch chỉ được ghi ở mức tác động lên dữ liệu nền:

| Tính năng | Tác động | Mapping |
|---|---|---|
| Storage Locations | Cho phép quản lý tồn theo location chi tiết | `res.config.settings.group_stock_multi_locations` |
| Multi-Step Routes | Cho phép warehouse sinh location/operation/rule trung gian | `res.config.settings.group_stock_adv_location` |
| Lots & Serial Numbers | Cho phép product tracking lot/serial | `res.config.settings.group_stock_production_lot` |
| Packages | Cho phép quản lý package | `res.config.settings.group_stock_tracking_lot` |
| Consignment | Cho phép quản lý owner trên tồn | `res.config.settings.group_stock_tracking_owner` |
| Expiration Dates | Bật module quản lý ngày hạn của lot/serial | `res.config.settings.module_product_expiry` (tích hợp `product_expiry`, bổ sung sau) |

Source: `addons/stock/models/res_config_settings.py`.
