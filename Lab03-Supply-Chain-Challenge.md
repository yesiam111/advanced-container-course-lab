# Lab 03 — Challenge: "Có người tráo image trên registry"

> **Thời gian ước lượng: ~110 phút** (lab nặng nhất)
> **Đầu vào:** image `smartapp:1.0` (từ Lab 02) + khóa công khai `cosign.pub` (chữ ký tuần trước)
> **Đầu ra:** image đã sửa & ký theo digest + `sbom.json` + `pipeline.sh` — dùng cho Lab 04.

---

## Bối cảnh

Hệ thống giám sát báo: image `smartapp:1.0` trên registry **không còn khớp chữ ký** mà team đã ký tuần trước. Cùng lúc, đợt rà soát dependency đang trễ hạn. Một image "sạch CVE" vẫn có thể bị tráo nguồn gốc — nhóm phải chứng minh điều đó, tìm thành phần nguy hiểm, và dựng hàng rào để không tái diễn.

## Thông tin đang có

- Một registry nội bộ; tag `smartapp:1.0` là image **hiện tại** (đáng ngờ).
- `cosign.pub` — khóa công khai dùng để ký `smartapp:1.0` **tuần trước**.

## Mục tiêu

1. **Chứng minh bị tráo.** Dùng chữ ký để cho thấy bản đã ký tuần trước là hợp lệ, còn image đang nằm ở tag `:1.0` **không qua được xác minh**. Ghi `SWAPPED_TAG` vào `answers.env`.
2. **Tìm thành phần nguy hiểm.** Tạo **SBOM** của image đáng ngờ, tìm thư viện lỗ hổng được cài lén. Ghi `VULN_LIB` vào `answers.env`.
3. **Dựng pipeline gác cổng.** Viết `pipeline.sh <image>` — quét CVE và xác minh chữ ký — **thoát khác 0** nếu có CVE mức CRITICAL hoặc chữ ký sai, thoát 0 nếu image sạch và hợp lệ.
4. **Sửa & ký lại.** Build lại image không còn thư viện lỗ hổng, ký bằng khóa của nhóm, đẩy lên registry theo **digest**. Ghi tham chiếu đã ký vào `SIGNED_IMAGE.txt` kèm `cosign.pub` của nhóm.

## `answers.env` (trong thư mục `supplychain/`)

```env
SWAPPED_TAG="..."
VULN_LIB="..."
```

## Bàn giao — thư mục `supplychain/`

```
supplychain/
  answers.env
  sbom.json
  pipeline.sh
  SIGNED_IMAGE.txt      # repo@sha256:... (image đã sửa & ký của nhóm)
  cosign.pub            # khóa công khai của nhóm
```

## Tự chấm

```bash
./graders/grade03.sh supplychain
```

> Grader dùng một file ground-truth **chỉ giảng viên có** — bản thân script an toàn để phát cho nhóm tự chấm.

## Ràng buộc

- `pipeline.sh` phải nhận **một** tham số là image ref và tự quyết PASS/FAIL bằng mã thoát (không hỏi tương tác).
- Image sửa lại phải ký được và **xác minh theo digest**, không theo tag.

---

### Input cho Lab 04

Giữ lại image đã sửa. Ở Lab 04, một báo cáo pentest cho thấy container smartapp **đọc được `/etc/shadow` của host** — nhóm sẽ tái hiện lỗ hổng rồi hardening để chặn.
