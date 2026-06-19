# ADR-001: Exception CVE chưa có patch — `openssl` HIGH CVE-XXXX-XXXXX

**Ngày:** 2025-XX-XX  
**Người quyết định:** <Tên / Team>  
**Hạn xem xét lại:** 2025-XX-XX + 30 ngày  
**Trạng thái:** ACTIVE

---

## Bối cảnh
Trivy phát hiện CVE có mức độ **HIGH** trong package `openssl` (base image `debian:bookworm-slim`).  
Tại thời điểm lập ADR, upstream Debian **chưa phát hành patch**.

## Quyết định
Cho phép tạm thời bỏ qua CVE này bằng file `.trivyignore` trong repo, có thời hạn.

## Lý do
1. Không có patch → không thể upgrade package để loại bỏ CVE.
2. Attack vector: Network — tuy nhiên service không expose port bị ảnh hưởng ra ngoài.
3. Có NetworkPolicy default-deny; attack surface thực tế giảm đáng kể.

## Biện pháp giảm thiểu rủi ro
- NetworkPolicy chặn inbound từ internet vào pod.
- Pod không có quyền leo thang đặc quyền (`allowPrivilegeEscalation: false`).
- Log tất cả connection vào pod qua Cilium / eBPF.

## Điều kiện xét lại / đóng ADR
- Debian/Ubuntu phát hành patch → nâng cấp base image và xoá entry `.trivyignore`.
- Hàng tuần: kiểm tra `trivy db update` xem có patch mới chưa.

## Cách ignore trong CI

Tạo file `.trivyignore` ở root repo:
```
# ADR-001: CVE chưa có patch, hạn xem lại: <DATE>
CVE-XXXX-XXXXX
```

Hoặc dùng tham số inline trong workflow:
```yaml
- uses: aquasecurity/trivy-action@master
  with:
    trivyignores: .trivyignore
    exit-code: '1'
    severity: HIGH,CRITICAL
```

---

**Phê duyệt:** ______________________  
**Tech Lead:** ______________________