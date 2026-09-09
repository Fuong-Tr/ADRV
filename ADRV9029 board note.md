# Ghi chú bo ADRV9029

> Tài liệu riêng cho bo **ADRV9029** trong dự án PetaLinux.
>
> Mốc tham chiếu của repository: **PetaLinux 2023.2**, **Vivado/Vitis 2023.2**, **ADI 2023_R2**, Zynq UltraScale+ MPSoC **XCZU15EG**.
>
> Phần cài đặt, build, đóng gói và boot PetaLinux dùng chung xem tại [PetaLinux_Common_Guide.md](PetaLinux_Common_Guide.md). Phần clock SI5518 và các ứng dụng TX/RX/DPD xem tại [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md).

---

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

## 2. Mốc phiên bản

Môi trường tham chiếu:

```text
Vivado / Vitis : 2023.2
PetaLinux      : 2023.2
ADI release    : 2023_R2
Kiến trúc      : Zynq UltraScale+ MPSoC / zynqMP
Device         : XCZU15EG
```

Nên giữ Vivado/XSA và PetaLinux cùng release để giảm lỗi tương thích.

Trước mỗi phiên build:

```bash
source <PETALINUX_2023_2_INSTALL_DIR>/settings.sh

petalinux-util --version
which petalinux-build
which petalinux-config
```

Kết quả phải trỏ về bộ PetaLinux 2023.2 đang dùng.

---

## 3. Chuẩn bị thư mục làm việc

Ví dụ:

```bash
mkdir -p ~/work/adrv9029
cd ~/work/adrv9029
```

Khuyến nghị bố trí:

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

```bash
cd ~/work/adrv9029/adi

git clone --branch 2023_R2 https://github.com/analogdevicesinc/linux.git
cd linux
git rev-parse HEAD
git status
```

Sau khi clone nên lưu lại commit thực tế:

```bash
git rev-parse HEAD
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

Tạo project ZynqMP:

```bash
cd ~/work/adrv9029/petalinux

petalinux-create -t project --template zynqMP --name adrv9029
cd adrv9029
```

Nếu dùng dữ liệu trong repository này, giải nén `project-spec.zip` vào project để có:

```text
<PROJECT_DIR>/project-spec/meta-user/
```

Ví dụ:

```bash
unzip <PATH_TO_ADRV_REPO>/project-spec.zip -d <PROJECT_DIR>
```

Nếu tạo lại từ XSA:

```bash
petalinux-config --get-hw-description=<THU_MUC_CHUA_XSA>
```

XSA phải đúng với thiết kế phần cứng, bitstream và cấu hình JESD/DMA sẽ dùng trên bo.

---

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

Build lại device tree:

```bash
petalinux-build -c device-tree
```

Clock SI5518 được mô tả riêng trong [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md).

---

## 12. Trình tự build khuyến nghị

### 12.1. Build firmware và ứng dụng

```bash
petalinux-build -c adrv-firmware
petalinux-build -c si5518config
petalinux-build -c tx-dma
petalinux-build -c rx-dma
petalinux-build -c dpd-app
```

### 12.2. Build kernel/device tree khi có thay đổi

```bash
petalinux-build -c kernel
petalinux-build -c device-tree
```

### 12.3. Build toàn bộ image

```bash
petalinux-build
```

Sau khi build riêng từng recipe vẫn nên chạy full build để cập nhật image/rootfs cuối cùng.

Output chính thường nằm tại:

```text
<PROJECT_DIR>/images/linux/
```

Ví dụ:

```text
image.ub
system.dtb
boot.scr
BOOT.BIN
```

Việc đóng gói BOOT.BIN và chuẩn bị SD card xem trong `PetaLinux_Common_Guide.md`.

---

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

## 14. Trình tự bring-up bo ADRV9029

Khuyến nghị kiểm tra theo thứ tự:

```text
Boot Linux
   ↓
Kiểm tra device tree / SPI / GPIO
   ↓
Cấu hình SI5518
   ↓
Xác nhận reference clock và SYSREF
   ↓
Nạp firmware/profile ADRV
   ↓
Kiểm tra probe ADRV
   ↓
Kiểm tra JESD204
   ↓
Kiểm tra IIO
   ↓
Kiểm tra RX
   ↓
Kiểm tra TX
   ↓
Kiểm tra DPD nếu cần
```

Không nên debug RX/TX trước khi clock và JESD đã ổn định.

---

## 15. Khi port sang project khác

Khi một đội khác cần port ADRV9029, nên bàn giao theo nhóm sau.

### Phần dùng chung

- PetaLinux release.
- Cách cài môi trường.
- Cách build.
- Cách đóng gói.
- Cách boot/nạp image.
- Cách xem log và kiểm tra Linux.

Các nội dung này nằm trong `PetaLinux_Common_Guide.md`.

### Phần riêng ADRV9029

- XSA/bitstream đúng revision.
- ADI release và commit SHA.
- `meta-adi-xilinx` path.
- Kernel config.
- Rootfs config.
- `system-user.dtsi`.
- Firmware/profile và checksum.
- Cấu hình clock/SYSREF.
- Thứ tự bring-up ADRV/JESD.
- Bài test RX/TX tham chiếu.
- Log của một lần chạy tốt.

Mục tiêu đầu tiên khi port không phải sửa code ngay mà là tái tạo được một baseline có thể build và boot, sau đó xác nhận lần lượt SPI → clock → ADRV → JESD → IIO → DMA.

---

## 16. Checklist nhanh

### HOST

```text
[ ] PetaLinux 2023.2 đã được source đúng
[ ] XSA đúng board/revision
[ ] ADI linux 2023_R2 đã clone
[ ] meta-adi 2023_R2 đã clone
[ ] Ghi lại commit SHA
[ ] meta-adi-xilinx đã thêm vào User Layers
[ ] Kernel config đã kiểm tra
[ ] Rootfs packages đã bật
[ ] system-user.dtsi đã đối chiếu với XSA
[ ] Firmware/profile đúng bộ
[ ] petalinux-build hoàn tất
```

### BOARD

```text
[ ] Linux boot thành công
[ ] SPI device xuất hiện
[ ] GPIO/reset đúng
[ ] SI5518 được cấu hình
[ ] Reference clock ổn định
[ ] Firmware/profile ADRV tồn tại
[ ] ADRV probe thành công
[ ] JESD link hoạt động
[ ] IIO device xuất hiện
[ ] RX test đạt
[ ] TX test đạt
[ ] Log được lưu lại
```

---

## 17. Tài liệu liên quan trong repository

- [PetaLinux_Common_Guide.md](PetaLinux_Common_Guide.md) — hướng dẫn chung môi trường, build, boot và kiểm tra.
- [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md) — SI5518, firmware script, TX/RX DMA và DPD.
- `project-spec.zip` — dữ liệu PetaLinux của project.
- `adrv-firmware.zip` — gói firmware lưu trữ trong repository.
