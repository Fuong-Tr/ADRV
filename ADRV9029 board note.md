# Ghi chú bo ADRV9029

## 1. Mục đích và cách sử dụng

Tài liệu hướng dẫn tích hợp **driver ADI trong kernel** vào PetaLinux cho bo ADRV9029: lấy mã nguồn, thêm Yocto layer, cấu hình kernel/rootfs, chuẩn bị device tree và firmware, kiểm tra sau khi boot.

Đọc cùng [PetaLinux Common Guide](PetaLinux_Common_Guide.md) để thực hiện các bước cài công cụ, tạo project từ XSA, build, đóng gói và nạp SD. Cấu hình SI5518 và các ứng dụng nằm trong [tài liệu riêng](SI5518_and_Apps_Guide.md); chỉ dùng phần phù hợp với bộ driver/image đang triển khai.

Các lệnh **HOST** chạy bằng Bash trên máy build; các lệnh **BOARD** chạy trên Linux của bo. Thay các giá trị `<...>` bằng thông tin thật trước khi chạy. Khi một bước báo lỗi, xử lý xong mới chuyển sang bước tiếp theo.

Nội dung đã đối chiếu với nguồn ADI và cấu hình phần cứng trong repository; chưa thực hiện build hoặc đo trên bo trong lần sửa tài liệu này.

## 2. Phiên bản và đầu vào

| Thành phần | Giá trị dùng cho hướng dẫn |
|---|---|
| PetaLinux | 2023.2 |
| Vivado xuất XSA | 2023.2 |
| Vitis | 2023.2 nếu cần sử dụng; không bắt buộc chỉ để build Linux từ XSA |
| Yocto | Langdale, theo PetaLinux 2023.2 |
| ADI meta-adi / Linux | Nhánh `2023_R2` |
| HDL ADI tham khảo | Nhánh `hdl_2023_r2` |
| Nền tảng bo đang tham chiếu | Zynq UltraScale+ MPSoC XCZU15EG, AArch64 |
| Template PetaLinux | `zynqMP` — phân biệt chữ hoa/thường |

Cần có XSA/bitstream đúng bo, device tree mô tả các kết nối, bộ firmware/profile đồng bộ và thông tin clock/reset. Không chọn BSP hoặc device tree ZCU102 chỉ vì cùng sử dụng ADRV9029.

**Cần phân biệt:** cài PetaLinux 2023.2 không tự bảo đảm project sử dụng kernel ADI. Kernel thực tế phụ thuộc recipe/layer và cấu hình nguồn của project.

Nguồn ADRV902x có các thành phần API liên quan DPD/CFR, nhưng điều đó chưa xác nhận có lệnh userspace tương ứng hoặc chức năng đã được bật trên image. Việc sử dụng phải đối chiếu phiên bản driver, firmware/profile và giao diện thực tế.

## 3. Chuẩn bị môi trường và đường dẫn — HOST

Cài PetaLinux theo Common Guide và yêu cầu AMD của bản 2023.2. Kích hoạt trong mỗi terminal mới:

```bash
export PLNX_DIR="$HOME/tools/petalinux/2023.2"
source "$PLNX_DIR/settings.sh"
command -v petalinux-config
command -v petalinux-build
```

Nếu cài tại vị trí khác, đổi PLNX_DIR. Ví dụ máy hiện có công cụ tại `/opt/tools/Xilinx/petalinux/2023.2` thì dùng đúng đường dẫn đó.

Chuẩn bị các thư mục:

```bash
export ADRV_WORK="$HOME/work/adrv9029"
export ADI_DIR="$ADRV_WORK/adi"
export PROJECT_DIR="$ADRV_WORK/petalinux/adrv9029"
mkdir -p "$ADI_DIR" "$ADRV_WORK/petalinux"
```

| Thư mục | Vai trò |
|---|---|
| `$ADI_DIR/meta-adi` | Layer tích hợp Linux ADI và các gói vào Yocto |
| `$ADI_DIR/linux` | Bản kernel tự clone để đọc/sửa, nếu cần |
| `$ADI_DIR/hdl` | HDL tham khảo, nếu cần |
| `$PROJECT_DIR` | Project PetaLinux |
| Thư mục chứa XSA/bitstream | Đầu vào phần cứng do đội FPGA bàn giao |

Không đặt nguồn/layer trong `build/tmp`. Các lệnh clone/tạo project bên dưới dành cho thư mục chưa có repo/project; nếu đã có, kiểm tra revision và thay đổi đang làm trước khi chuyển nhánh.

## 4. Lấy mã nguồn Analog Devices — HOST

### 4.1. Layer meta-adi — cần cho luồng tích hợp này

```bash
cd "$ADI_DIR"
git clone --branch 2023_R2 --single-branch \
  https://github.com/analogdevicesinc/meta-adi.git
export ADI_LAYER="$ADI_DIR/meta-adi/meta-adi-xilinx"
test -f "$ADI_LAYER/conf/layer.conf"
git -C "$ADI_DIR/meta-adi" rev-parse HEAD
```

Thư mục phải thêm vào Yocto là **meta-adi-xilinx**, không phải thư mục cha meta-adi.

Layer này bổ sung cấu hình để recipe `linux-xlnx` lấy kernel từ `analogdevicesinc/linux`. Vì vậy, tên recipe vẫn là linux-xlnx dù nguồn kernel là của ADI.

### 4.2. Kernel ADI — clone riêng khi cần đọc hoặc sửa

```bash
cd "$ADI_DIR"
git clone --branch 2023_R2 --single-branch \
  https://github.com/analogdevicesinc/linux.git
git -C "$ADI_DIR/linux" rev-parse HEAD
```

Driver nằm tại:

```text
<ADI_DIR>/linux/drivers/iio/adc/adrv902x/
```

**Có thể bỏ bước clone riêng** nếu chỉ build bằng recipe của meta-adi: Yocto sẽ tải kernel theo recipe.

Thư mục vừa clone không tự trở thành nguồn build của Yocto. Sửa file tại `$ADI_DIR/linux` sẽ không tác động image nếu project vẫn đang lấy kernel qua recipe Git. Khi phát triển driver, đưa thay đổi thành patch trong layer của project hoặc cấu hình rõ nguồn kernel cục bộ theo UG1144, rồi kiểm tra nguồn thực tế được build.

### 4.3. HDL ADI — chỉ khi cần

```bash
cd "$ADI_DIR"
git clone --branch hdl_2023_r2 --single-branch \
  https://github.com/analogdevicesinc/hdl.git
git -C "$ADI_DIR/hdl" rev-parse HEAD
```

Thiết kế ADI liên quan có tên `projects/adrv9026`; tên này không có nghĩa chỉ hỗ trợ chip ADRV9026. Tuy nhiên, thiết kế bo tham khảo không thay thế nguyên trạng phần cứng XCZU15EG đang dùng. Nếu đã nhận XSA/bitstream đúng bo thì không cần build lại HDL để build Linux.

### 4.4. Lưu revision

Ghi commit meta-adi, kernel và các patch thực tế của mỗi bản bàn giao. Không dùng tên branch làm bằng chứng duy nhất vì branch có thể được cập nhật.

Trong meta-adi 2023_R2 đã đối chiếu, recipe kernel/libiio có dùng AUTOREV khi build online. Chốt commit meta-adi chưa đủ để chốt toàn bộ nguồn.

Khi đã có bộ commit được đội dự án thống nhất, có thể đặt trong `project-spec/meta-user/conf/petalinuxbsp.conf`:

```bitbake
SRCREV:pn-linux-xlnx = "<KERNEL_COMMIT_40_KY_TU>"
SRCREV:pn-libiio = "<LIBIIO_COMMIT_40_KY_TU>"
```

Thay cả hai giá trị bằng commit hợp lệ thuộc nguồn/nhánh đang dùng; không để nguyên dấu `<...>`. Chỉ thêm dòng libiio nếu sử dụng gói này. Bản clone để đọc và bản kernel do Yocto lấy có thể khác commit; cần đối chiếu trước khi debug.

## 5. Tạo project và nhập XSA — HOST

Thực hiện theo [Common Guide](PetaLinux_Common_Guide.md), với:

- Template: `zynqMP`.
- Project: `$PROJECT_DIR`.
- XSA xuất từ Vivado 2023.2 cho đúng bo.

Sau khi project tồn tại:

```bash
cd "$PROJECT_DIR"
test -d project-spec/meta-user
```

Đây là luồng tạo project từ XSA và các thành phần phần mềm đã bàn giao; không yêu cầu phục hồi toàn bộ project từ ZIP.

## 6. Thêm layer ADI vào Yocto — HOST

```bash
cd "$PROJECT_DIR"
printf '%s\n' "$ADI_LAYER"
petalinux-config
```

Trong menu **Yocto Settings → User Layers**, nhập đường dẫn tuyệt đối do printf in ra, ví dụ:

```text
/home/<user>/work/adrv9029/adi/meta-adi/meta-adi-xilinx
```

Không gõ nguyên `$ADI_LAYER` hoặc `~` vào menu. Đường dẫn cũ `/home/fw5/adi/meta-adi/meta-adi-xilinx` chỉ đúng trên máy có thư mục đó.

Lưu, thoát và kiểm tra:

```bash
grep -n 'CONFIG_USER_LAYER' project-spec/configs/config
grep -n 'meta-adi-xilinx' build/conf/bblayers.conf
test -f "$ADI_LAYER/conf/layer.conf"
```

Trong **Linux Components Selection**, dùng lựa chọn kernel `linux-xlnx` cho luồng recipe của meta-adi. Không chuyển sang nguồn kernel khác mà chưa cấu hình tương ứng.

Kiểm tra nội dung layer:

```bash
grep -nE 'KERNELURI|KBRANCH|KBUILD_DEFCONFIG|SRCREV' \
  "$ADI_LAYER/recipes-kernel/linux/linux-xlnx_%.bbappend"
```

Kỳ vọng nguồn Git là `analogdevicesinc/linux`, KBRANCH là `2023_R2` và defconfig ZynqMP là `adi_zynqmp_defconfig`. Đây là kiểm tra recipe; cần kiểm tra log/cấu hình cuối để xác nhận nó thực sự được áp dụng.

## 7. Các đường dẫn cần biết

| Nội dung | Đường dẫn trong project, trừ khi ghi khác |
|---|---|
| Cấu hình hệ thống PetaLinux | `project-spec/configs/config` |
| Lựa chọn rootfs | `project-spec/configs/rootfs_config` |
| Khai báo layer meta-user | `project-spec/meta-user/conf/layer.conf` |
| Biến cấu hình Yocto của project | `project-spec/meta-user/conf/petalinuxbsp.conf` |
| Danh sách layer đã sinh | `build/conf/bblayers.conf` |
| Bổ sung recipe kernel | `project-spec/meta-user/recipes-kernel/linux/linux-xlnx_%.bbappend` |
| Fragment kernel | `project-spec/meta-user/recipes-kernel/linux/linux-xlnx/*.cfg` |
| Device tree của bo | `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` |
| Recipe firmware của bo | `project-spec/meta-user/recipes-apps/adrv-firmware/adrv-firmware.bb` |
| File firmware/profile | `project-spec/meta-user/recipes-apps/adrv-firmware/files/` |
| Layer ADI | `$ADI_DIR/meta-adi/meta-adi-xilinx/` |
| Driver trong bản kernel clone riêng | `$ADI_DIR/linux/drivers/iio/adc/adrv902x/` |
| Device tree sinh từ XSA | Thường dưới `components/plnx_workspace/device-tree/`; vị trí cụ thể phụ thuộc luồng build |
| Kernel đang build | Thường dưới `build/tmp/work-shared/` hoặc `build/tmp/work/`; không đồng nhất với bản clone riêng |
| Image đầu ra | `images/linux/` |

Xem file sinh tự động để debug; lưu thay đổi lâu dài trong meta-user/layer được quản lý phiên bản.

## 8. Cấu hình kernel — HOST

```bash
cd "$PROJECT_DIR"
petalinux-config -c kernel
```

Dùng phím `/` để tìm symbol. Với driver ADI trong kernel 2023_R2, kiểm tra:

| Nhóm | Symbol | Thiết lập/ý nghĩa |
|---|---|---|
| Module | `CONFIG_MODULES` | `y` nếu dùng module |
| SPI | `CONFIG_SPI`, `CONFIG_SPI_CADENCE` | Bật SPI và controller của ZynqMP |
| GPIO, clock | `CONFIG_GPIOLIB`, `CONFIG_GPIO_ZYNQ`, `CONFIG_COMMON_CLK` | Bật các phụ thuộc điều khiển bo |
| IIO | `CONFIG_IIO` | Bật Industrial I/O |
| ADRV9029 | `CONFIG_ADRV9025` | `m` để nạp bằng modprobe, hoặc `y` khi cần tích hợp vào kernel |
| RX ADC/TPL | `CONFIG_CF_AXI_ADC` | Core ADC; driver ADRV9025 có select phụ thuộc này |
| TX DAC/TPL | `CONFIG_CF_AXI_DDS` | Core DDS/DAC theo datapath TX |
| IIO DMA buffer | `CONFIG_IIO_BUFFER`, `CONFIG_IIO_BUFFER_DMAENGINE` | Kiểm tra được bật qua các core sử dụng |
| DMA | `CONFIG_AXI_DMAC` | ADI AXI DMA controller |
| XCVR | `CONFIG_AXI_ADXCVR`, `CONFIG_XILINX_TRANSCEIVER` | AXI transceiver và hỗ trợ transceiver Xilinx |
| JESD framework | `CONFIG_JESD204` | Cần cho topology JESD |
| JESD RX/TX | `CONFIG_AXI_JESD204_RX`, `CONFIG_AXI_JESD204_TX` | Cần theo các IP trong XSA |
| JESD top độc lập | `CONFIG_JESD204_TOP_DEVICE` | Chỉ bắt buộc khi dùng node generic jesd204-top-device; khác thuộc tính jesd204-top-device trên node ADRV |
| SPI userspace | `CONFIG_SPI_SPIDEV` | Khi ứng dụng clock cần spidev |
| Debug | `CONFIG_DEBUG_FS` | Khi cần giao diện debugfs |

Tên **CONFIG_ADRV9025** và module **adrv9025_drv** được dùng cho họ chip gồm ADRV9029. Không tìm một symbol CONFIG_ADRV9029 riêng rồi kết luận thiếu driver.

Khi cần lập trình clock từ userspace trước khi khởi tạo ADRV, nên dùng driver ADRV dạng module và kiểm soát thời điểm nạp. Module vẫn có thể tự nạp qua udev/modalias; chọn `=m` chưa tự bảo đảm trì hoãn probe. Phải kiểm tra dịch vụ khởi động và log của image.

Nếu dùng buffer DMA liên tục, kiểm tra CMA theo cơ chế DMA của thiết kế. Không tự đặt CMA bằng kích thước reserved-memory của một ứng dụng khác.

### Lưu và kiểm tra cấu hình cuối

Giữ cấu hình cần tái tạo trong fragment .cfg và tham chiếu fragment trong bbappend. Ví dụ **nội dung thêm vào file**, không phải lệnh shell:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI:append = " file://adrv9029.cfg"
```

File tương ứng là `recipes-kernel/linux/linux-xlnx/adrv9029.cfg`. Nếu bbappend đã có các dòng này, sửa/bổ sung tại chỗ, không thay mất nội dung hiện có. Nhiều fragment có thể đặt cùng symbol khác nhau; phải kiểm tra .config cuối sau build.

```bash
petalinux-build -c kernel
find build/tmp -type f -name .config -path '*linux*'
```

Chọn đúng .config kernel của lần build hiện tại, rồi kiểm tra:

```bash
export KERNEL_CONFIG="<DUONG_DAN_CONFIG_KERNEL_VUA_TIM>"
grep -E '^CONFIG_(ADRV9025|IIO|CF_AXI_ADC|CF_AXI_DDS|AXI_DMAC|AXI_ADXCVR|XILINX_TRANSCEIVER|JESD204|AXI_JESD204_RX|AXI_JESD204_TX)=' \
  "$KERNEL_CONFIG"
```

Nếu ADRV9025 không có trong menu, kiểm tra layer/nguồn kernel trước. Nếu symbol bị tắt sau build, kiểm tra phụ thuộc và các fragment ghi đè.

## 9. Cấu hình rootfs — HOST

```bash
cd "$PROJECT_DIR"
petalinux-config -c rootfs
```

Cần đưa vào image:

- Module kernel được dùng nếu chọn `=m`; có thể dùng gói `kernel-modules` khi bring-up.
- Gói `adrv-firmware` nếu project đã có recipe và đầy đủ các file mà recipe yêu cầu.
- `libiio` và gói tiện ích, thường là `libiio-tests` với recipe phù hợp.
- Công cụ modprobe/modinfo, cùng SSH nếu cần truy cập từ HOST.

Ví dụ thêm các gói đã tồn tại vào `petalinuxbsp.conf`:

```bitbake
IMAGE_INSTALL:append = " kernel-modules libiio libiio-tests adrv-firmware"
```

Không thêm tên gói khi chưa có recipe/package cung cấp nó. Nếu chưa bàn giao recipe firmware, hoàn thành mục 10 trước. Bật thư viện libiio không tự bảo đảm có các lệnh iio_info/iio_attr; kiểm tra trên bo:

```bash
command -v modprobe
command -v modinfo
command -v iio_info
command -v iio_attr
```

Các lựa chọn rootfs của project cũ có thể tham chiếu gói không còn nguồn; bỏ lựa chọn không dùng hoặc bổ sung recipe thật.

## 10. Firmware và profile ADRV

Các file phải phù hợp cùng phiên bản API/profile và thiết kế JESD. Tên được tham chiếu trong device tree của bo:

| File trong /lib/firmware | Nội dung |
|---|---|
| `ActiveUseCase.profile` | Profile thiết bị, kênh và JESD |
| `ActiveUtilInit.profile` | Cấu hình khởi tạo |
| `ADRV9025_FW.bin` | Firmware ARM |
| `ADRV9025_DPDCORE_FW.bin` | Firmware lõi DPD khi cấu hình yêu cầu |
| `stream_image.bin` | Stream firmware |
| `RxGainTable.csv` | Bảng gain RX |
| `TxAttenTable.csv` | Bảng attenuation TX |

Recipe `adrv-firmware.bb` và thư mục `files/` là thành phần phần mềm của bo; chúng không được tạo chỉ bằng cách clone meta-adi. Đặt bộ recipe/firmware được bàn giao vào đường dẫn mục 7 và kiểm tra toàn bộ mục trong SRC_URI đều tồn tại. Recipe có thể đóng gói thêm file/script ngoài bảy file trên.

Trên BOARD:

```bash
for f in ActiveUseCase.profile ActiveUtilInit.profile \
  ADRV9025_FW.bin ADRV9025_DPDCORE_FW.bin stream_image.bin \
  RxGainTable.csv TxAttenTable.csv
do
  if [ -s "/lib/firmware/$f" ]; then
    sha256sum "/lib/firmware/$f"
  else
    printf 'THIEU FILE: %s\n' "$f"
  fi
done
```

Chỉ grep tên ADRV trong /lib/firmware sẽ bỏ sót các profile, stream_image.bin và bảng CSV. Lưu checksum để so sánh với bộ bàn giao.

Sự tồn tại của script khởi tạo không phải tiêu chí xác nhận driver ADI đã hoạt động. Đọc nội dung script và kiểm tra nó phù hợp với module/giao diện đang dùng trước khi chạy.

## 11. Device tree và kết nối phần cứng

### 11.1. Chọn luồng device tree

Với bo XCZU15EG trong repository, cấu hình nguồn đã đối chiếu sử dụng DT sinh từ XSA cộng với `system-user.dtsi`, đồng thời chặn phần recipe device tree của meta-adi bằng dòng trong `project-spec/meta-user/conf/layer.conf`:

```bitbake
BBMASK += "meta-adi-xilinx/recipes-bsp/device-tree/"
```

Dòng này không chặn toàn bộ layer ADI; phần kernel/libiio vẫn được sử dụng. Dùng nó khi theo đúng luồng DT của bo này và đã có DTS mô tả đủ các IP ADI.

Với thiết kế tham khảo ADI sử dụng DT của meta-adi, luồng thiết lập KERNEL_DTB khác. Không áp dụng đồng thời một tên DT ZCU102 từ README ADI vào luồng DT của bo XCZU15EG.

### 11.2. Các giá trị cần đối chiếu

| Hạng mục trong cấu hình bo | Giá trị đã đọc / yêu cầu |
|---|---|
| ADRV | SPI1, CS0; SPI tối đa 10 MHz trong DTS |
| Reset | `reset-gpios = <&gpio 100 0>`; đây là chỉ số trong GPIO controller, không phải số chân trên đầu nối |
| Clock thiết bị | dev_clk, tham chiếu `misc_clk_0`; cấu hình mô tả 245,76 MHz |
| JESD TX | DEFRAMER0_LINK_TX = 0 |
| JESD RX | FRAMER0_LINK_RX = 2 |
| RX DMA / TX DMA | 0x9C400000 / 0x9C420000 |
| RX JESD / TX JESD | 0x84AA0000 / 0x84A90000 |
| RX XCVR / TX XCVR | 0x84A60000 / 0x84A80000 |
| RX ADC/TPL / TX DAC/TPL | 0x84A00000 / 0x84A04000 |

Các địa chỉ này thuộc thiết kế đang tham chiếu, phải kiểm tra lại nếu XSA thay đổi. Dùng đầy đủ DTS của bo đã được đối chiếu; bảng trên không thay thế device tree hoàn chỉnh.

Đối với driver ADI, kiểm tra compatible theo bảng match của đúng revision; tên chuẩn có vendor prefix, chẳng hạn `adi,adrv9029`. Node nguồn cũ có thể dùng chuỗi `adrv9025`; không suy ra chip hoặc driver chỉ từ tên node, và không đổi tên tùy ý mà bỏ qua binding/probe thực tế.

Ngoài các giá trị trên, kiểm tra:

- Clock vật lý, SYSREF, reset và nguồn phải sẵn sàng.
- Topology JESD, lane mapping, tham số profile và cấu hình HDL phải khớp.
- RX ADC/TPL liên kết SPI device qua spibus-connected khi driver yêu cầu.
- Các phandle dmas, clocks, interrupt và trạng thái okay đúng IP.
- Kích thước vùng reg và reserved-memory đúng thiết kế.

Fixed-clock trong DTS mô tả tần số cho Linux; nó không lập trình SI5518 hoặc tạo ra clock vật lý. Phần thao tác SI5518 xem tài liệu riêng.

## 12. Build và kiểm tra trước khi nạp — HOST

Thực hiện build/đóng gói/SD theo [Common Guide](PetaLinux_Common_Guide.md). Sau khi cấu hình ADI, có thể build từng phần để khoanh vùng:

```bash
cd "$PROJECT_DIR"
petalinux-build -c kernel
petalinux-build -c device-tree
petalinux-build -c adrv-firmware
petalinux-build
```

Chỉ chạy lệnh adrv-firmware khi recipe đã được đưa vào project. Build từng thành phần không thay cho full build để cập nhật rootfs/image.

Trước khi nạp:

```bash
mkdir -p handoff-logs
dumpimage -l images/linux/image.ub
dtc -I dtb -O dts -o handoff-logs/system-built.dts images/linux/system.dtb
grep -nE 'adrv902|jesd|9c400000|9c420000' handoff-logs/system-built.dts
find build/tmp -type f -name 'adrv9025_drv.ko*'
```

Không có file .ko là bình thường khi chọn driver built-in. Khi chọn module, cần kiểm tra module có trong **rootfs đầu ra**, không chỉ có ở thư mục biên dịch. Với luồng INITRD của bo, kiểm tra FIT có ramdisk phù hợp. Đóng gói BOOT.BIN bằng bitstream đi cùng XSA và nạp bộ file cùng lần build.

## 13. Kiểm tra sau khi boot — BOARD

### 13.1. Linux và SPI

```bash
uname -a
cat /etc/os-release
cat /proc/cmdline
cat /proc/device-tree/model

for d in /sys/bus/spi/devices/spi*
do
  [ -d "$d" ] || continue
  printf '\n%s\n' "$d"
  readlink -f "$d/driver" 2>/dev/null
  cat "$d/modalias" 2>/dev/null
done
```

Thiết bị ADRV được driver kernel quản lý không cần xuất hiện dưới /dev/spidev*. Spidev là giao diện SPI userspace của thiết bị khác khi được cấu hình; không dùng nó làm tiêu chí phát hiện ADRV.

### 13.2. Clock và nạp driver

Hoàn thành cấu hình clock/SYSREF và reset trước khi khởi tạo ADRV. Nếu driver được build dạng module và chưa được nạp:

```bash
modinfo adrv9025_drv
modprobe adrv9025_drv
lsmod
dmesg | tail -n 150
```

Dùng tài khoản đủ quyền. modprobe xử lý các phụ thuộc module đã được khai báo, nhưng không thay thế việc cấu hình clock vật lý. Không nạp lại liên tục để xử lý lỗi clock/probe.

Nếu chọn `=y`, driver không có trong lsmod. Nếu modprobe trả 0, vẫn phải xác nhận thiết bị bind/probe thành công.

### 13.3. IIO

```bash
for d in /sys/bus/iio/devices/iio:device*
do
  [ -f "$d/name" ] || continue
  printf '%s: ' "$d"
  cat "$d/name"
done
iio_info -u local:
```

Dùng tên thiết bị và các thuộc tính để xác định ADRV; không cố định iio:device0/1/2. Tên thực tế có thể mang tên họ ADRV902x theo driver/thiết bị được nhận diện.

### 13.4. JESD

Sau khi tìm đúng IIO device ADRV, đặt đường dẫn:

```bash
export ADRV_DEV="/sys/bus/iio/devices/<IIO_DEVICE_ADRV>"
cat "$ADRV_DEV/jesd204_fsm_state"
cat "$ADRV_DEV/jesd204_fsm_error"
dmesg | grep -Ei 'adrv|jesd|dmac|iio|timeout|error'
```

Với luồng JESD FSM tương ứng, giá trị đích cần kiểm tra là:

```text
jesd204_fsm_state = opt_post_running_stage
jesd204_fsm_error = 0
```

Đây là giá trị mong đợi, không phải log đo thực tế. Nếu thuộc tính không tồn tại, kiểm tra đúng device, revision driver và CONFIG_JESD204 trước khi kết luận. Không ghi lệnh reset FSM khi chưa hiểu giao diện của phiên bản đang chạy.

## 14. Tiêu chí kiểm tra và lỗi thường gặp

| Mức | Điều kiện cần xác nhận |
|---|---|
| Build | Kernel ADI đúng revision; config, DT và rootfs phù hợp; full build/package thành công |
| Boot | Linux chạy đúng image trên đúng bo |
| Driver | SPI bind/probe thành công, firmware/profile được nạp, IIO xuất hiện |
| JESD | FSM/link đạt trạng thái chạy, không còn lỗi khởi tạo chưa xử lý |
| Dữ liệu | Bài thử TX/RX thực tế đạt yêu cầu; IIO xuất hiện chưa đủ chứng minh DMA hoạt động |

| Hiện tượng | Kiểm tra |
|---|---|
| Không thấy ADRV9025 trong menuconfig | Layer ADI, recipe/kernel thực tế và các phụ thuộc |
| Có clone Linux nhưng sửa không có tác dụng | Project có thực sự dùng nguồn đó hay vẫn tải nguồn qua recipe |
| Không tìm thấy layer | Đường dẫn tuyệt đối tới meta-adi-xilinx và conf/layer.conf |
| Không có gói firmware/tiện ích | Recipe, tên package và lựa chọn rootfs |
| Module not found | Driver là built-in hay module; module có trong image không |
| Invalid module format | Kernel ABI/vermagic, kiến trúc và lần build |
| DT báo thiếu label | XSA, DTS và luồng sinh DT có thống nhất không |
| Không có IIO ADRV | SPI, compatible, firmware, clock/reset và log probe |
| JESD timeout | Clock thực, SYSREF, PLL, lane mapping và profile |
| Linux chạy nhưng dữ liệu sai | Bài thử DMA/TX/RX, packing mẫu và cấu hình datapath |

Lưu commit nguồn, checksum image/firmware, cấu hình kernel, DT đã build, log UART và dmesg cho mỗi lần thử. Khi port driver, thay từng phần và so sánh lại cùng bài kiểm tra.

## 15. Tài liệu tham khảo

- [AMD UG1144 2023.2 — thêm Yocto layer](https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Adding-Layers).
- [ADI meta-adi 2023_R2 — hướng dẫn PetaLinux](https://github.com/analogdevicesinc/meta-adi/blob/2023_R2/meta-adi-xilinx/README.md).
- [Recipe kernel ADI đã đối chiếu](https://github.com/analogdevicesinc/meta-adi/blob/b406e8ff6b735deb5dbfcb8a6d46662b8a371d14/meta-adi-xilinx/recipes-kernel/linux/linux-xlnx_%25.bbappend).
- [Kconfig ADRV9025/9026/9029 đã đối chiếu](https://github.com/analogdevicesinc/linux/blob/86d61468a7856e952c7ca237f798d86d6abd2e27/drivers/iio/adc/Kconfig).
- [Tài liệu Linux driver ADRV9026/ADRV9029 của ADI](https://analogdevicesinc.github.io/linux/drivers/iio-transceiver/adrv9025/).

Các commit trong liên kết là mốc đối chiếu nội dung, không phải xác nhận một bộ image đã được kiểm thử trên bo. Tài liệu ADI trực tuyến có thể mô tả release mới hơn; dùng mã nguồn đúng revision để xác định tên symbol và giao diện.
