# Kế hoạch review tài liệu Odoo Inventory Core

> Trạng thái: `draft`
>
> Ngày lập kế hoạch: `2026-09-08`

## 1. Mục tiêu

Xác định bộ tài liệu Inventory Core có đủ độ chính xác, khả năng truy vết và governance để được dùng làm tài liệu tham chiếu hay chưa.

Kết quả review phải trả lời được:

1. Nội dung nghiệp vụ có đúng với behavior hiện tại của Odoo không?
2. Mapping model/method/XML ID/test có tồn tại và còn phù hợp không?
3. Source of Truth cho code, nghiệp vụ và cấu hình triển khai là gì?
4. Tài liệu có thể tái kiểm chứng sau khi source thay đổi không?

## 2. Phạm vi

### Tài liệu cần review

- `README.md`
- `01_master_data.md`
- `03_receipt_flow.md`
- `04_delivery_flow.md`
- `05_internal_transfer_flow.md`
- `06_inventory_adjustment_flow.md`

### Source kỹ thuật cần đối chiếu

- `addons/stock/models/**/*.py`
- `addons/stock/views/**/*.xml`
- `addons/stock/wizard/**/*.py`
- `addons/stock/wizard/**/*.xml`
- `addons/stock/tests/**/*.py`

### Ngoài phạm vi đợt này

- Purchase, Sales, Manufacturing, Quality, Barcode và Accounting integration.
- Thay đổi behavior trong source code Odoo.
- Chốt quy trình nghiệp vụ nếu chưa có business owner/approver xác nhận.

## 3. Source baseline cần ghi nhận

Review phải pin vào một revision cụ thể:

```text
Repository: https://github.com/linh404/odoo.git
Branch: 19.0
Commit: 44db31a2804e7de0b098b3582ba39dd30465eb79
Last verified: 2026-09-08
```

Nếu source thay đổi sau revision này, phải chạy lại phần technical validation trước khi đánh dấu tài liệu là `verified`.

## 4. Các bước review

### Phase 1 — Scope và cấu trúc

- [ ] Kiểm tra mục tiêu, phạm vi và các flow được mô tả.
- [ ] Kiểm tra numbering giữa tên file, workflow ID và README.
- [ ] Kiểm tra các liên kết Markdown nội bộ.
- [ ] Xác định phần nào là `draft`, `verified`, `approved` hoặc `later`.

### Phase 2 — Kiểm tra Source of Truth

- [ ] Ghi rõ technical SoT: repository, branch/tag, commit.
- [ ] Ghi rõ business/process SoT: owner và approver.
- [ ] Ghi rõ deployment SoT: database/configuration/warehouse setup.
- [ ] Định nghĩa thứ tự ưu tiên khi code, cấu hình và tài liệu nghiệp vụ khác nhau.
- [ ] Bổ sung `owner`, `approver`, `last_verified`, `source_revision` cho từng tài liệu.

### Phase 3 — Technical mapping audit

- [ ] Kiểm tra model technical name.
- [ ] Kiểm tra method/class có tồn tại trong source.
- [ ] Kiểm tra XML ID có tồn tại trong XML data.
- [ ] Kiểm tra test class/test method có tồn tại.
- [ ] Bổ sung line anchor hoặc permalink cho mapping quan trọng.
- [ ] Kiểm tra mapping có mô tả đúng call chain hay chỉ là mô tả khái quát.

### Phase 4 — Nghiệp vụ từng flow

- [ ] Receipt: one/two/three-step, putaway, tracking, package, backorder, return, scrap.
- [ ] Delivery: ship/pick-pack-ship, reservation/removal, partial delivery, backorder, return, scrap.
- [ ] Internal Transfer: same warehouse, inter-warehouse, transit, package, owner, backorder.
- [ ] Inventory Adjustment: counted quantity, difference, conflict/outdated, adjustment move.
- [ ] Kiểm tra các precondition có khớp behavior thực tế.
- [ ] Phân biệt manual operation với route/procurement/scheduler.

### Phase 5 — Validation và phân loại finding

Mỗi finding ghi theo mẫu:

```text
ID:
Severity: blocker | high | medium | low
File/line:
Claim hiện tại:
Evidence/source:
Impact:
Đề xuất sửa:
Owner:
Status:
```

Phân loại:

- `blocker`: sai hoặc thiếu khiến tài liệu không thể dùng làm chuẩn.
- `high`: sai behavior hoặc thiếu provenance quan trọng.
- `medium`: thiếu traceability, governance hoặc diễn giải gây hiểu nhầm.
- `low`: formatting, numbering hoặc wording.

### Phase 6 — Review và phê duyệt

- [ ] Tác giả tài liệu xử lý các finding.
- [ ] Technical reviewer xác nhận mapping code.
- [ ] Business owner xác nhận flow nghiệp vụ.
- [ ] Approver đánh dấu tài liệu `approved`.
- [ ] Commit tài liệu và source baseline vào Git.

## 5. Các finding đã biết cần kiểm tra lại

### HIGH — Internal Transfer bypass reservation

`05_internal_transfer_flow.md:57` đang ghi `transit/view` có thể bypass reservation.

Đối chiếu hiện tại cho thấy:

- `supplier`, `customer`, `inventory`, `production` mới là các usage bypass reservation.
- `transit` vẫn chịu reservation.
- `view` không phải source/destination thực tế của move.

### HIGH — Cross-company wording

`05_internal_transfer_flow.md:55` cần làm rõ cross-company thường đi qua transit/inter-company route và các move/picking riêng, không nên hiểu là direct internal transfer giữa hai company.

### MEDIUM — Revision/provenance

README và các bảng source reference chưa pin commit, line anchor, ngày kiểm chứng hoặc cơ chế tự kiểm tra reference.

### MEDIUM — Authority/governance

Chưa có owner, approver, effective date, review cadence và quy tắc precedence giữa code, cấu hình triển khai, business process và Markdown.

## 6. Tiêu chí hoàn thành

Review được coi là hoàn thành khi:

- [ ] Không còn finding `blocker` hoặc `high` chưa có quyết định.
- [ ] Mọi technical reference quan trọng đều trace được tới source revision cụ thể.
- [ ] Các flow có business owner/approver hoặc được đánh dấu rõ là `draft`.
- [ ] README nêu rõ technical SoT, business SoT và deployment SoT.
- [ ] Có cách kiểm tra lại source path, method, XML ID và test reference.
- [ ] Tài liệu được commit vào Git cùng metadata review.

## 7. Deliverables

1. Review report có finding theo severity.
2. Danh sách mapping đã xác minh và mapping cần sửa.
3. Bản cập nhật README với Source of Truth metadata.
4. Bản cập nhật các flow sau khi xử lý finding.
5. Biên bản approval hoặc trạng thái `draft` nếu chưa có business sign-off.
