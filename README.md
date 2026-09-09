# ADRV9029 trên PetaLinux

Nguồn và tài liệu để dựng môi trường **PetaLinux 2023.2**, build image và kiểm tra **ADRV9029** trên bo **Zynq UltraScale+ MPSoC XCZU15EG** của dự án.

## Hướng dẫn

| Tài liệu | Nội dung |
|---|---|
| [Petalinux My Guide.txt](Petalinux%20My%20Guide.txt) | Môi trường, phục hồi project, cấu hình driver/device tree, build, đóng gói, boot SD, kiểm tra ADRV/JESD và chuẩn bị port |
| [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md) | Clock SI5518 riêng của bo; firmware script; TX/RX DMA; DPD TX4 và các bài kiểm tra |

Bắt đầu với hướng dẫn chính. Trước khi khởi tạo ADRV/JESD trên bo này, thực hiện phần clock trong tài liệu SI5518. Các app được mô tả riêng để người port phân biệt phần driver và phần phụ thuộc bo.

## Dữ liệu trong repository

| File | Vai trò |
|---|---|
| [project-spec.zip](project-spec.zip) | Cấu hình PetaLinux, XSA/bitstream, device tree, driver tùy chỉnh, firmware/profile và mã ứng dụng |
| [adrv-firmware.zip](adrv-firmware.zip) | Gói firmware lưu trữ riêng; cần đối chiếu phiên bản trước khi sử dụng thay bộ trong project-spec |

Driver tùy chỉnh cho ADRV9029 đang mang tên **adrv9025_custom**. Các tên ADRV9025/ADRV9026 trong firmware/API/IP được giữ theo source đang dùng.

## Trạng thái bàn giao

Tài liệu được đối chiếu từ source và tài liệu AMD/ADI ngày **09/09/2026**. Lần cập nhật này chưa chạy build PetaLinux hoặc kiểm thử trên bo.

Bộ image đã chạy tốt, file clock SI5518, waveform tham chiếu, ảnh cấu hình boot của bo và log đo thực tế còn cần chủ bo cung cấp. Các tiêu chí PASS và mẫu output trong hướng dẫn không phải chứng nhận hệ thống đã qua kiểm thử.

Hướng dẫn ghi rõ mốc nguồn công khai dùng để thử build. Commit/phần sửa đổi của môi trường lab gốc chưa được xác nhận; cần chốt chúng khi bàn giao bản đã kiểm thử.

