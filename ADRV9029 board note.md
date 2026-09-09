# Ghi chú bo ADRV9029


## 1. Mục đích

Tài liệu này tập trung vào các nội dung riêng của bo ADRV9029:

- Phiên bản công cụ và mã nguồn nên sử dụng.
- Clone source của Analog Devices.
- Thêm layer ADI vào PetaLinux/Yocto.
- Các đường dẫn Yocto quan trọng.
- Cấu hình kernel liên quan ADRV, IIO, JESD204 và DMA.
- Cấu hình rootfs.
- Device tree.
- Firmware/profile ADRV.
- Trình tự build.
- Các bước kiểm tra trên bo sau khi boot.
- Quy trình dùng làm mốc khi port driver sang project khác.

Các bước PetaLinux chung như cài tool, tạo project, import XSA, đóng gói BOOT.BIN và chuẩn bị SD card không lặp lại chi tiết ở đây.

---

## 2. Phiên bản

Môi trường tham chiếu:

```text
Vivado / Vitis : 2023.2
PetaLinux      : 2023.2
ADI release    : 2023_R2
Kiến trúc      : Zynq UltraScale+ MPSoC / zynqMP
```
Kết quả phải trỏ về bộ PetaLinux 2023.2 đang dùng.

---

## 3. Chuẩn bị thư mục làm việc

Khuyến nghị :

```text
~/work/adrv9029/
├── adi/
│   ├── linux/
│   ├── meta-adi/
│   └── hdl/
└── petalinux/
    └── adrv9029/
```

Không đặt source ADI bên trong `build/tmp` vì đây là vùng sinh tự động của Yocto.

---

## 4. Clone source Analog Devices

### 4.1. Linux kernel của ADI

LINUX KERNEL HIỆN TẠI ĐÃ CÓ CÙNG VỚI MÔI TRƯỜNG PETALINUX 2023_2. CÓ SẴN DPD VÀ CFR TUY NHIÊN CHƯA RÕ CÁCH SỬ DỤNG TRỰC TIẾP, CÓ THỂ BỎ QUA PHẦN 4.1 NÀY

```bash
cd ~/work/adrv9029/adi

git clone --branch 2023_R2 https://github.com/analogdevicesinc/linux.git
```

Không chỉ ghi tên branch; commit SHA giúp tái tạo đúng môi trường đã kiểm thử.

Source ADRV902x trong kernel ADI nằm tại:

```text
~/work/adrv9029/adi/linux/drivers/iio/adc/adrv902x/
```

### 4.2. Layer Yocto của ADI

```bash
cd ~/work/adrv9029/adi

git clone --branch 2023_R2 https://github.com/analogdevicesinc/meta-adi.git
cd meta-adi
git rev-parse HEAD
```

Layer dùng cho PetaLinux/Xilinx:

```text
~/work/adrv9029/adi/meta-adi/meta-adi-xilinx
```

### 4.3. HDL của ADI

Chỉ cần khi muốn rebuild hoặc đối chiếu thiết kế HDL/XSA tham chiếu:

```bash
cd ~/work/adrv9029/adi

git clone --branch hdl_2023_r2 https://github.com/analogdevicesinc/hdl.git
```

Nếu đã có XSA đã xác nhận chạy tốt thì bước này không bắt buộc cho việc build Linux.

---

## 5. Tạo hoặc phục hồi project PetaLinux

ĐÃ HƯỚNG DẪN Ở FILE Petalinux_Common_Guide.md

## 6. Thêm layer ADI vào Yocto

Từ thư mục project:

```bash
cd <PROJECT_DIR>
petalinux-config
```

Vào:

```text
Yocto Settings
  -> User Layers
```

Thêm đường dẫn tuyệt đối:

```text
/home/<user>/work/adrv9029/adi/meta-adi/meta-adi-xilinx
```

Sau khi lưu cấu hình, kiểm tra layer đã được Yocto nhận.

Nguyên tắc:

- Source/layer ADI đặt ngoài vùng sinh tự động.
- Các thay đổi của project nên giữ trong `project-spec/meta-user` hoặc layer riêng.
- Không sửa trực tiếp file trong `build/tmp` vì có thể bị ghi đè ở lần build tiếp theo.

---

## 7. Các đường dẫn Yocto/PetaLinux quan trọng

| Nội dung | Đường dẫn |
|---|---|
| Layer của project | `<PROJECT_DIR>/project-spec/meta-user/` |
| Cấu hình layer | `<PROJECT_DIR>/project-spec/meta-user/conf/petalinuxbsp.conf` |
| Device tree | `<PROJECT_DIR>/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` |
| Firmware ADRV | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/adrv-firmware/` |
| Ứng dụng SI5518 | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/si5518config/` |
| Ứng dụng TX | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/tx-dma/` |
| Ứng dụng RX | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/rx-dma/` |
| Ứng dụng DPD | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/dpd-app/` |
| Layer ADI cho Xilinx | `<ADI_DIR>/meta-adi/meta-adi-xilinx/` |
| Source driver ADRV902x của ADI | `<ADI_DIR>/linux/drivers/iio/adc/adrv902x/` |
| Device tree PL sinh tự động | `<PROJECT_DIR>/components/plnx_workspace/device-tree/device-tree/pl.dtsi` |
| Output build | `<PROJECT_DIR>/images/linux/` |

Khi debug Yocto có thể xem nội dung trong `build/tmp`, nhưng không dùng các file trong đó làm source chính thức.

---

## 8. Cấu hình kernel

Mở kernel configuration:

```bash
petalinux-config -c kernel
```

Tên symbol có thể khác đôi chút theo kernel/release. Dùng phím `/` trong menuconfig để tìm theo từ khóa.

Các nhóm chức năng cần kiểm tra:

| Chức năng | Cấu hình cần có |
|---|---|
| SPI | Controller SPI của ZynqMP được bật |
| SPI userspace | `CONFIG_SPI_SPIDEV` nếu ứng dụng SI5518 dùng spidev |
| IIO | Industrial I/O core |
| IIO buffer | Buffer support cho luồng RX/TX |
| JESD204 | JESD204 framework và các thành phần liên quan |
| AXI DMAC | ADI AXI DMAC |
| AXI JESD RX/TX | Các core JESD RX/TX tương ứng thiết kế |
| AXI transceiver | Transceiver support tương ứng HDL |
| ADC/DAC | Các AXI ADC/DAC/IIO core cần cho datapath |
| CMA/DMA | Bộ nhớ liên tục đủ cho buffer RX/TX |
| GPIO | Reset/control GPIO của bo |
| GPIO sysfs | Cần nếu chương trình SI5518 hiện tại vẫn dùng giao diện sysfs cũ |
| DebugFS | Hữu ích cho debug IIO/JESD/DPD |
| Module support | Bật nếu các driver được build dạng module |

Sau khi chỉnh kernel:

```bash
petalinux-build -c kernel
```

---

## 9. Cấu hình rootfs

Mở:

```bash
petalinux-config -c rootfs
```

Bật các gói cần cho bo, tối thiểu theo nhu cầu thử nghiệm:

```text
adrv-firmware
si5518config
tx-dma
rx-dma
dpd-app
kernel-modules
```

Nên bật thêm bộ công cụ libiio:

```text
libiio
libiio-tests
```

Sau khi boot phải kiểm tra lệnh thực tế đã có:

```bash
command -v iio_info
command -v iio_attr
```

Bật thư viện libiio không đồng nghĩa mọi tiện ích dòng lệnh đều đã được đưa vào image.

---

## 10. Firmware và profile ADRV

Recipe firmware nằm tại:

```text
project-spec/meta-user/recipes-apps/adrv-firmware/adrv-firmware.bb
```

Các file liên quan nằm dưới:

```text
project-spec/meta-user/recipes-apps/adrv-firmware/files/
```

Sau khi build, firmware/profile cần được cài vào đúng vị trí mà driver hoặc script yêu cầu, thường là:

```text
/lib/firmware/
```

Kiểm tra trên bo:

```bash
ls -lah /lib/firmware | grep -Ei 'adrv|9025|9029'
ls -l /usr/bin/adrv-firmware.sh
```

Nếu cần đối chiếu phiên bản:

```bash
sha256sum /lib/firmware/*ADRV* 2>/dev/null
```

Không thay firmware/profile chỉ vì tên file giống nhau. Khi bàn giao nên lưu cả checksum của bộ đã kiểm thử.

---

## 11. Device tree

File chính cần kiểm tra:

```text
project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

Không sửa trực tiếp các file sinh dưới `build/tmp`.

Với ADRV9029 cần đối chiếu ít nhất:

- Node SPI của ADRV.
- `compatible`.
- Chip select.
- SPI frequency.
- Reference clock.
- SYSREF.
- Reset/control GPIO.
- JESD204 links.
- AXI transceiver.
- AXI JESD RX/TX.
- AXI ADC/DAC.
- RX/TX DMA.
- Interrupt.
- CMA/reserved memory nếu thiết kế yêu cầu.
- `status = "okay"` cho các block cần sử dụng.


## 12. Build, Đóng gói và Boots

ĐÃ CÓ TRONG Petalinux_Commnond_Guide.md

## 13. Kiểm tra sau khi boot

### 13.1. Xác nhận hệ thống Linux

```bash
uname -a
cat /etc/os-release
cat /proc/cmdline
cat /proc/device-tree/model
dmesg | tail -n 150
```

### 13.2. Kiểm tra SPI

```bash
ls -l /dev/spidev*
ls -l /sys/bus/spi/devices/
```

Xem driver bind với từng SPI device:

```bash
for d in /sys/bus/spi/devices/spi*
do
  [ -d "$d" ] || continue
  echo "=== $d ==="
  readlink -f "$d/driver" 2>/dev/null
  cat "$d/modalias" 2>/dev/null
done
```

### 13.3. Kiểm tra IIO

```bash
ls -l /sys/bus/iio/devices/
iio_info
```

Liệt kê tên IIO device:

```bash
for d in /sys/bus/iio/devices/iio:device*
do
  [ -d "$d" ] || continue
  printf '%s: ' "$d"
  cat "$d/name" 2>/dev/null
done
```

### 13.4. Kiểm tra log ADRV/JESD

```bash
dmesg | grep -Ei 'adrv|9025|9029|jesd|axi|dmac|iio'
```

Cần xác nhận không chỉ driver xuất hiện mà còn:

- Probe thành công.
- SPI giao tiếp được.
- Firmware/profile được nạp.
- JESD link lên đúng trạng thái.
- IIO device được tạo.
- DMA hoạt động với bài thử RX/TX.

### 13.5. Kiểm tra module

Nếu driver được build dạng module:

```bash
lsmod
modinfo <TEN_MODULE>
```

Nếu driver được build vào kernel (`=y`) thì không xuất hiện trong `lsmod`.

---
