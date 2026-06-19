# Runbook: Rotate Cosign Signing Key

## Mục đích
Hướng dẫn từng bước thay thế cặp key Cosign mà không gây downtime cho workload đang chạy.

## Điều kiện kích hoạt
- Key bị lộ (private key leak)
- Key hết hạn theo policy (hàng năm)
- Nhân viên phụ trách key nghỉ việc

---

## Các bước thực hiện

### 1. Tạo cặp key mới
```bash
cosign generate-key-pair
# → cosign.key  (private — KHÔNG commit)
# → cosign.pub  (public  — commit vào repo)
```

### 2. Thêm private key mới vào GitHub Secret
- GitHub → Settings → Secrets → Actions
- Tạo secret tạm: `COSIGN_PRIVATE_KEY_NEW`
- Giữ `COSIGN_PRIVATE_KEY` cũ để workflow không đứt ngay

### 3. Cập nhật ClusterImagePolicy thêm authority mới (song song)
```yaml
authorities:
  - name: cosign-key-old   # giữ key cũ trong thời gian migration
    key:
      data: |
        <OLD_PUBLIC_KEY>
  - name: cosign-key-new
    key:
      data: |
        <NEW_PUBLIC_KEY>
```
Commit → ArgoCD sync → policy nhận cả 2 key.

### 4. Cập nhật workflow dùng key mới
Đổi `COSIGN_PRIVATE_KEY` → `COSIGN_PRIVATE_KEY_NEW` trong workflow.
Chạy một build để image được ký bằng key mới.

### 5. Xác nhận image mới pass admission
```bash
cosign verify --key signing/cosign.pub \
  <IMAGE_NAME>:<NEW_TAG>
# Phải in ra: Verification for <image> -- The following checks were performed...
```

### 6. Xoá key cũ khỏi ClusterImagePolicy
Sau khi TẤT CẢ workload đã được redeploy với image ký bằng key mới:
- Xoá `cosign-key-old` khỏi policy.
- Xoá GitHub Secret `COSIGN_PRIVATE_KEY_NEW`, đổi tên thành `COSIGN_PRIVATE_KEY`.
- Cập nhật `signing/cosign.pub` thành public key mới.

### 7. Ghi nhận
- Ghi vào Change Log: ngày rotate, lý do, người thực hiện.
- Xoá file `cosign.key` cũ khỏi máy local.

---

## Kiểm tra sau rotate
| Kiểm tra | Kỳ vọng |
|---|---|
| `cosign verify --key signing/cosign.pub <new-image>` | Verification OK |
| Deploy image ký bằng key cũ | admission reject |
| Deploy image ký bằng key mới | pass |