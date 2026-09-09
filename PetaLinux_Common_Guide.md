# PetaLinux Common Guide
## Hướng dẫn chung: môi trường → build → nạp/boot → kiểm tra

## 1. Quy ước 

- **HOST:** máy Linux để cài công cụ, build, chuẩn bị thẻ.
- **BOARD:** Linux chạy trên phần cứng đích.
- **U-BOOT:** console bootloader, trước khi Linux khởi động.
- Chạy lệnh HOST bằng Bash, tài khoản thường; chỉ dùng sudo cho thao tác hệ thống cần quyền.
- Các giá trị dạng `<...>` là chỗ phải thay, không dán nguyên vào terminal.
- Dừng tại bước lỗi và xử lý trước khi thực hiện bước phụ thuộc tiếp theo.

| Thuộc hướng dẫn chung này | Thuộc tài liệu riêng của board |
|---|---|
| Cài công cụ, kích hoạt môi trường | Phiên bản đã kiểm thử, mã board/revision |
| Tạo project, nhập XSA, cấu hình và build | XSA/bitstream, BSP, layer và commit cụ thể |
| Đóng gói, chép SD, theo dõi boot | Boot switch, UART, nguồn, khe SD, flash layout |
| Kiểm tra Linux và cách kiểm tra driver | Tên driver, firmware, device tree, địa chỉ, clock/reset |
| Mẫu báo cáo và xử lý lỗi chung | SI5518, ADRV9029, JESD, RF và bài test ứng dụng |


## 2. Đầu vào 

Trước khi build, đội cung cấp và đội tiếp nhận thống nhất:

| Đầu vào | Yêu cầu |
|---|---|
| Nền tảng | SoC, board và revision chính xác |
| Bộ công cụ | PetaLinux và Vivado xuất phần cứng tương thích; ưu tiên cùng release |
| Phần cứng | XSA hợp lệ; bitstream đi cùng nếu thiết kế dùng PL |
| BSP | Nếu dùng: đúng board và release; board custom có thể dùng template + XSA |
| Phần mềm | Layer/recipe, kernel, U-Boot, driver, firmware, patch cần thiết |
| Revision | Commit/tag cố định và các thay đổi chưa commit |
| Boot | SD/JTAG/flash, rootfs RAM hay EXT4, console và tài khoản |
| Kiểm thử | Image tham chiếu nếu có, bài test và kết quả mong đợi |

XSA mô tả phần cứng xuất từ Vivado; không thay thế mã nguồn driver hoặc toàn bộ project RTL. XSA thường chưa đủ để Linux điều khiển mọi ngoại vi ngoài chip: vẫn cần device tree và driver phù hợp.

Nếu chỉ cần dựng hệ thống cơ bản, có thể bắt đầu với template + XSA. Để kiểm tra driver đặc thù, phải bổ sung các phụ thuộc của driver theo tài liệu board trước khi kết luận.

## 3. Chuẩn bị môi trường — HOST

### 3.1. Hệ điều hành và tài nguyên

Dùng Linux x86_64 trong danh sách được hỗ trợ của **đúng release**. Ví dụ PetaLinux 2023.2 hỗ trợ Ubuntu 20.04.6 và 22.04.2. Theo [AMD: Installation Requirements][requirements], mốc tối thiểu là RAM 8 GB, CPU 8 lõi ở 2 GHz hoặc tương đương, ổ trống 100 GB.

Cho máy làm việc, nên dự trù RAM 16–32 GB và SSD trống 200 GB trở lên để chứa nguồn/cache/nhiều lần build; đây là đề xuất vận hành, không phải yêu cầu tối thiểu AMD.

Dùng filesystem Linux cục bộ, đường dẫn không có khoảng trắng. Có Internet hoặc mirror/cache offline đã chuẩn bị. Không đặt chung TMPDIR cho nhiều project.

```bash
cat /etc/os-release
uname -m
free -h
nproc
df -h .
readlink -f /bin/sh
```

### 3.2. Gói phụ thuộc

Cài danh sách package theo Release Notes/Prerequisites của phiên bản đã chọn. Nhóm dưới đây là ví dụ khởi đầu trên Ubuntu 22.04 cho luồng 2023.2; không thay thế danh sách đầy đủ của AMD:

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential gcc-multilib gawk git wget diffstat chrpath socat \
  xterm autoconf automake libtool texinfo unzip zip cpio pax rsync \
  python3 python3-pip python3-pexpect python3-git python3-jinja2 \
  libncurses5-dev libtinfo5 zlib1g-dev libssl-dev \
  file bc screen u-boot-tools device-tree-compiler
```

Nếu installer báo thiếu gói, bổ sung đúng gói/release rồi chạy lại. Không đổi sang bản Ubuntu mới hơn chỉ vì tên gói khác.

Nếu tài liệu release yêu cầu `/bin/sh` là Bash và kết quả đang là dash, trên máy Ubuntu build chuyên dụng:

```bash
sudo dpkg-reconfigure dash
```

Chọn **No** khi được hỏi dùng dash làm /bin/sh. Thao tác này thay đổi shell hệ thống của HOST.

### 3.3. Cài công cụ và kích hoạt

Tải installer từ AMD cho release đã thống nhất. Đặt biến dưới đây thành đường dẫn thật:

```bash
export PLNX_VERSION="2023.2"
export PLNX_INSTALL_DIR="$HOME/tools/petalinux/$PLNX_VERSION"
export PLNX_INSTALLER="$HOME/Downloads/<PETALINUX_INSTALLER>.run"

test -f "$PLNX_INSTALLER"
mkdir -p "$PLNX_INSTALL_DIR"
chmod u+x "$PLNX_INSTALLER"
"$PLNX_INSTALLER" --dir "$PLNX_INSTALL_DIR"
source "$PLNX_INSTALL_DIR/settings.sh"

command -v petalinux-create
command -v petalinux-config
command -v petalinux-build
```

Không chạy installer/build bằng root. Mỗi terminal mới cần source lại settings.sh. Dùng terminal riêng, tránh trộn nhiều release hoặc môi trường Yocto/SDK khác nhau. Xem [Installing the PetaLinux Tool][install].

Vivado cần khi tạo/sửa/xuất lại thiết kế FPGA. Nếu đã nhận XSA/bitstream phù hợp, người build Linux không nhất thiết phải dựng lại RTL. JTAG cần công cụ/cable driver phù hợp trên HOST.

## 4. Tạo project và nhập phần cứng — HOST

### 4.1. Cách A: project mới từ template và XSA

Ví dụ cho ZynqMP; đổi template thành `zynq` nếu dùng Zynq-7000:

```bash
export PLNX_WORK_DIR="$HOME/petalinux-work"
export PLNX_PROJECT_NAME="linux_system"
export PLNX_TEMPLATE="zynqMP"
export PLNX_HW_DIR="$HOME/hardware-export"

mkdir -p "$PLNX_WORK_DIR"
cd "$PLNX_WORK_DIR"
petalinux-create -t project --template "$PLNX_TEMPLATE" --name "$PLNX_PROJECT_NAME"
export PLNX_PROJECT_DIR="$PLNX_WORK_DIR/$PLNX_PROJECT_NAME"
cd "$PLNX_PROJECT_DIR"
petalinux-config --get-hw-description="$PLNX_HW_DIR"
```

PLNX_HW_DIR phải là thư mục chứa XSA đúng board; nên chỉ để một XSA cần nhập. Dùng thư mục xuất phần cứng riêng với thư mục nội bộ project.

Trong menu, rà lại CPU, DDR, UART, Ethernet, SD và cấu hình boot. XSA/bitstream phải cùng thiết kế. Sau khi thay XSA, nhập lại và kiểm tra các tùy chỉnh device tree còn khớp.

### 4.2. Cách B: project từ BSP

Chọn cách này thay cho cách A khi đã có BSP phù hợp:

```bash
cd "$PLNX_WORK_DIR"
petalinux-create -t project -s "<BSP_PATH>"
```

Đọc tên thư mục vừa tạo trong log, đặt lại PLNX_PROJECT_DIR và vào thư mục đó. Chỉ nhập XSA thay thế khi BSP/board yêu cầu. Không lấy BSP của board khác chỉ vì cùng họ SoC.

Tham khảo [ví dụ tạo project][create].

## 5. Cấu hình phần mềm — HOST

Từ thư mục project:

```bash
cd "$PLNX_PROJECT_DIR"
petalinux-config
petalinux-config -c kernel
petalinux-config -c rootfs
```

| Cấu hình | Cần quyết định |
|---|---|
| Hệ thống | Console, boot medium, rootfs, user layers, nguồn kernel/U-Boot |
| Kernel | Driver controller, driver thiết bị, filesystem, module support |
| Rootfs | Module, firmware, tiện ích kiểm tra, SSH nếu cần, tài khoản đăng nhập |
| Device tree | Ngoại vi, compatible, clock/reset, interrupt, bus, reserved-memory |

### 5.1. Layer, kernel và driver

Nếu board cần layer bên ngoài, lấy đúng revision và thêm đường dẫn trong **Yocto Settings → User Layers**. Kiểm tra `conf/layer.conf` có tồn tại. Ghi lại commit kernel/U-Boot và các layer; chỉ ghi tên branch là chưa đủ để tái tạo.

Driver built-in dùng `=y`; module dùng `=m` và phải được cài vào rootfs. Bật module trong kernel không tự đảm bảo file .ko có trong image. Firmware cần được recipe đưa vào đúng đường dẫn driver yêu cầu.

Cấu hình/rootfs cơ bản không tự bổ sung driver nhà cung cấp. Các tên module, symbol kernel, recipe và patch cụ thể thuộc tài liệu board.

### 5.2. Device tree

File tùy chỉnh thường nằm ở:

```text
project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

Giữ thay đổi trong meta-user/layer và các fragment được quản lý phiên bản. Không sửa trực tiếp file sinh dưới build/tmp vì lần build sau có thể ghi đè.

Node device tree chỉ mô tả phần cứng; node clock không tự lập trình chip clock ngoài. Kiểm tra label/address/interrupt với XSA và schematic.

### 5.3. Chọn rootfs trước khi chuẩn bị thẻ

| Phương án | Nội dung cần nạp | Lưu ý |
|---|---|---|
| INITRAMFS/INITRD chạy trong RAM | Boot files và kernel/FIT có ramdisk phù hợp, hoặc ramdisk rời theo boot script | File ghi vào RAM thường mất sau reboot |
| EXT4 trên SD | Boot files trên FAT32; rootfs trên phân vùng EXT4 | Bootargs phải trỏ đúng root device/PARTUUID |
| NFS root | Boot files/kernel và rootfs trên server | Cần mạng, export và bootargs phù hợp |

Ví dụ SD bên dưới ưu tiên **rootfs trong RAM**. Chỉ chọn phương án này nếu RAM đủ. Kiểm tra **Image Packaging Configuration** và image recipe; không mặc định image.ub luôn chứa rootfs.

## 6. Build và kiểm tra đầu ra — HOST

```bash
cd "$PLNX_PROJECT_DIR"
mkdir -p handoff-logs
set -o pipefail
petalinux-build 2>&1 | tee handoff-logs/build.log
```

Chỉ tiếp tục khi build trả mã 0 và hoàn tất thành công. `pipefail` giúp lỗi build không bị che bởi lệnh tee.

Build lại thành phần để khoanh vùng sau khi sửa:

```bash
petalinux-build -c kernel
petalinux-build -c device-tree
petalinux-build -c <RECIPE_NAME>
petalinux-build
```

Thay RECIPE_NAME bằng recipe thật. Sau build riêng driver/app, chạy full build để cập nhật image/rootfs trước khi nạp.

```bash
ls -lh images/linux
test -s images/linux/image.ub
test -s images/linux/system.dtb
dumpimage -l images/linux/image.ub
dtc -I dtb -O dts -o handoff-logs/system-built.dts images/linux/system.dtb
```

Các lệnh trên giả định output FIT thông dụng. Nếu chọn image rời, kiểm tra bộ file đúng cấu hình đó.

| File thường gặp | Vai trò |
|---|---|
| image.ub | FIT; có kernel, DTB và có thể có ramdisk theo cấu hình |
| system.dtb | Device tree đã biên dịch |
| boot.scr | Script U-Boot nạp hệ thống |
| rootfs.tar.gz / rootfs.ext4 | Rootfs rời nếu đã chọn sinh định dạng này |
| zynq_fsbl.elf / zynqmp_fsbl.elf | FSBL tương ứng kiến trúc |
| pmufw.elf, bl31.elf | PMU firmware và TF-A của luồng ZynqMP |
| u-boot.elf | U-Boot |
| BOOT.BIN | Được tạo tại bước đóng gói |

Kiểm tra FIT có ramdisk nếu luồng boot yêu cầu; DTB có node mong đợi; rootfs có module/firmware. File tồn tại chỉ xác nhận artifact được sinh, chưa xác nhận chạy tốt. Tham khảo [Building a PetaLinux System Image][build].

## 7. Đóng gói BOOT.BIN — HOST

Chọn đúng một ví dụ theo SoC. Đặt đường dẫn bitstream thực:

```bash
export PLNX_BITSTREAM="<ABSOLUTE_BITSTREAM_PATH>"
test -s "$PLNX_BITSTREAM"
```

**Zynq UltraScale+ MPSoC:**

```bash
petalinux-package --boot \
  --fsbl images/linux/zynqmp_fsbl.elf \
  --fpga "$PLNX_BITSTREAM" \
  --u-boot --force
```

**Zynq-7000:**

```bash
petalinux-package --boot \
  --fsbl images/linux/zynq_fsbl.elf \
  --fpga "$PLNX_BITSTREAM" \
  --u-boot --force
```

Nếu không nạp PL ở giai đoạn boot, bỏ `--fpga` theo thiết kế và ghi rõ khi nào PL được cấu hình. Không bỏ bitstream nếu driver cần IP PL đang hoạt động.

Đọc log đóng gói để xác nhận các input đúng lần build; ZynqMP cần chuỗi FSBL, PMUFW, TF-A và U-Boot phù hợp. `--force` cho phép ghi lại artifact đóng gói, không flash board.

```bash
test -s images/linux/BOOT.BIN
ls -lh images/linux/BOOT.BIN
```

Tham khảo [AMD: cấu hình U-Boot và đóng gói][package]. Luồng Versal/PDI hoặc flash layout khác cần lệnh riêng.

## 8. Nạp qua SD và boot

### 8.1. Gói bàn giao — HOST

Ví dụ FIT/rootfs RAM và boot.scr chuẩn:

```bash
cd "$PLNX_PROJECT_DIR"
mkdir -p handoff-images
cp images/linux/BOOT.BIN images/linux/image.ub images/linux/boot.scr handoff-images/
(
  cd handoff-images
  sha256sum BOOT.BIN image.ub boot.scr > SHA256SUMS
)
```

Nếu boot script nạp thêm DTB/Image/ramdisk rời, bổ sung đúng các file đó và checksum. Có thể đọc script bằng:

```bash
dumpimage -T script -p 0 -o handoff-logs/boot.cmd images/linux/boot.scr
```

### 8.2. Chuẩn bị thẻ — HOST

1. Xác định thẻ theo model/dung lượng; sao lưu dữ liệu trước khi phân vùng.
2. Dùng Disks/GParted chuẩn bị phân vùng boot FAT32 theo yêu cầu BootROM/board; MBR và FAT32 đầu tiên là cách thường dùng với Zynq/ZynqMP.
3. Chọn dung lượng đủ chứa toàn bộ boot files. Rootfs EXT4 cần thêm phân vùng Linux.
4. Mount FAT32 và xác nhận đúng điểm mount.

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,LABEL,MOUNTPOINTS
export PLNX_SD_BOOT="/media/$USER/BOOT"
mountpoint "$PLNX_SD_BOOT"
```

Chỉ sau khi xác nhận thành công:

```bash
cp "$PLNX_PROJECT_DIR/handoff-images/BOOT.BIN" "$PLNX_SD_BOOT/"
cp "$PLNX_PROJECT_DIR/handoff-images/image.ub" "$PLNX_SD_BOOT/"
cp "$PLNX_PROJECT_DIR/handoff-images/boot.scr" "$PLNX_SD_BOOT/"
cp "$PLNX_PROJECT_DIR/handoff-images/SHA256SUMS" "$PLNX_SD_BOOT/"
(
  cd "$PLNX_SD_BOOT"
  sha256sum -c SHA256SUMS
)
sync
```

Unmount/eject bằng công cụ hệ thống trước khi rút thẻ.

**Nếu dùng EXT4:** ngoài boot files, giải nén rootfs đã build lên phân vùng EXT4 trống, giữ quyền sở hữu:

```bash
export PLNX_SD_ROOT="/media/$USER/rootfs"
mountpoint "$PLNX_SD_ROOT"
# Chỉ chạy sau khi xác nhận đúng phân vùng EXT4 đích.
sudo tar --numeric-owner -xpf "$PLNX_PROJECT_DIR/images/linux/rootfs.tar.gz" \
  -C "$PLNX_SD_ROOT"
sync
```

Phải chọn sinh rootfs.tar.gz từ trước. Xác định root device/PARTUUID và bootargs đúng board; không mặc định mọi board đều dùng /dev/mmcblk0p2. Không dùng boot script rootfs RAM nguyên trạng cho rootfs EXT4.

### 8.3. Boot board

1. Tắt nguồn, đặt boot mode SD theo tài liệu board, lắp đúng khe.
2. Nối nguồn và UART console đúng thông số phần cứng.
3. Mở serial terminal và bật lưu log trước khi cấp nguồn.
4. Bật board, theo dõi U-Boot → Linux → đăng nhập.

Ví dụ UART **chỉ khi board cấu hình 115200, 8N1, không flow control**:

```bash
screen /dev/ttyUSB0 115200
```

Thay cổng bằng thiết bị thật. Tài khoản/mật khẩu lấy từ cấu hình image hoặc bên bàn giao; không mặc định root/root.

### 8.4. JTAG hoặc QSPI/eMMC

JTAG thích hợp tải image để thử/debug, thường vào RAM; không đồng nghĩa ghi image bền vững. Cần đúng cable, hardware server, boot mode và lệnh theo release. Kiểm tra `petalinux-boot --help` và UG1144 của bản dùng.

QSPI/eMMC cần xác định chip đích, partition/offset, kích thước, boot script và cách phục hồi trước khi ghi. Các giá trị này bắt buộc nằm trong tài liệu board; không có một lệnh flash chung an toàn cho mọi mạch. Luồng thao tác đầy đủ của hướng dẫn này là SD.

## 9. Kiểm tra kết quả — BOARD

### 9.1. Xác nhận Linux

```bash
uname -a
cat /etc/os-release
cat /proc/cmdline
cat /proc/device-tree/model
cat /proc/mounts
df -h
free -m
dmesg | tail -n 100
```

Đối chiếu kernel, model, rootfs và log boot với lần build vừa nạp. Rootfs RAM và EXT4 phải phù hợp cấu hình đã chọn.

Nếu cần mạng:

```bash
ip addr
ip route
ping -c 4 <HOST_IP>
```

Thay HOST_IP. Có địa chỉ IP chưa tự chứng minh SSH hoạt động; SSH phải được bật trong rootfs và có tài khoản hợp lệ.

### 9.2. Kiểm tra driver theo giao diện chung

Với module, chỉ nạp sau khi nguồn/clock/reset/phụ thuộc riêng đã sẵn sàng:

```bash
modinfo <MODULE_NAME>
modprobe <MODULE_NAME>
lsmod
dmesg | tail -n 100
```

Driver built-in không xuất hiện trong lsmod. Không thấy module ở lsmod chưa đủ kết luận thiếu driver.

Kiểm tra thiết bị đã bind đúng driver:

```bash
readlink -f /sys/bus/<BUS>/devices/<DEVICE_ID>/driver
```

Điền BUS và DEVICE_ID theo bus thực tế. Kiểm tra thêm giao diện driver cung cấp: /dev, sysfs, IIO hoặc network. `modprobe` trả 0 chỉ cho biết nạp module thành công; vẫn phải xác nhận probe/bind và chức năng thiết bị.

Tên module, firmware, trạng thái JESD, bài test TX/RX, clock SI5518 và lệnh app thuộc tài liệu riêng. Boot Linux thành công không chứng minh ngoại vi đã hoạt động.

### 9.3. Lưu log

```bash
mkdir -p /tmp/bringup-logs
uname -a > /tmp/bringup-logs/uname.txt
cat /proc/cmdline > /tmp/bringup-logs/cmdline.txt
cat /proc/mounts > /tmp/bringup-logs/mounts.txt
dmesg > /tmp/bringup-logs/dmesg.txt
```

Chép log về HOST hoặc storage bền vững trước khi reboot. /tmp và rootfs RAM có thể mất dữ liệu. Giữ cả log UART từ lúc cấp nguồn để thấy lỗi trước kernel.

## 10. Tiêu chí nghiệm thu chung

| Mức | Điều kiện đạt | Bằng chứng |
|---|---|---|
| Môi trường | Kích hoạt đúng release, đủ dependencies | OS, cấu hình máy, đường dẫn công cụ |
| Build | Full build và package thành công | Log và danh sách artifact |
| Image | Bộ boot/rootfs đồng nhất, checksum khớp | SHA256SUMS và cấu hình rootfs |
| Boot | Vào Linux trên đúng board/image | UART log, uname, model, cmdline |
| Driver | Bind đúng thiết bị, không có lỗi khởi tạo chưa xử lý | dmesg, sysfs, module/firmware |
| Chức năng | Bài test board/app đạt thông số thống nhất | Log/đo đạc theo tài liệu riêng |

Không đánh dấu mức sau dựa vào mức trước. Nếu phạm vi chỉ là Linux nền, ghi kiểm tra driver/chức năng là **CHƯA THỬ** hoặc **NGOÀI PHẠM VI**.

Mẫu ghi cho mỗi lần thử:

```text
Ngày / người thực hiện:
Board / revision:
Host OS / CPU / RAM:
PetaLinux / Vivado:
Project / layer / kernel / U-Boot commits:
Patch hoặc thay đổi chưa commit:
XSA và bitstream SHA256:
BOOT.BIN / image.ub / boot.scr / rootfs SHA256:
Boot medium / rootfs type:
Môi trường: PASS / FAIL / CHƯA THỬ
Build và image: PASS / FAIL / CHƯA THỬ
Boot Linux: PASS / FAIL / CHƯA THỬ
Driver: PASS / FAIL / CHƯA THỬ / NGOÀI PHẠM VI
Chức năng: PASS / FAIL / CHƯA THỬ / NGOÀI PHẠM VI
Đường dẫn log / lỗi / cách tái hiện:
```

## 11. Lỗi thường gặp

| Hiện tượng | Kiểm tra trước |
|---|---|
| Không tìm thấy petalinux-* | Source đúng settings.sh trong terminal hiện tại |
| Installer/build thiếu thư viện | OS được hỗ trợ, danh sách package và lỗi cụ thể |
| Fetch source thất bại | DNS, proxy, chứng chỉ, mirror và quyền truy cập nguồn |
| Không thấy layer / Nothing PROVIDES | Đường dẫn layer, recipe, tên package và revision |
| Device tree label không tồn tại | XSA mới có khớp system-user.dtsi và luồng sinh DT không |
| Hết RAM/ổ đĩa | RAM, swap, dung lượng/inode; giảm parallelism trong cấu hình Yocto khi cần |
| Không có UART | Nguồn, boot mode, đúng cổng/baud, image bootloader |
| U-Boot không thấy image | Đúng SD/partition, tên file và đường dẫn trong boot.scr |
| Kernel không mount root | root=, filesystem driver, ramdisk hoặc phân vùng EXT4 |
| Module not found | Module có được cài vào rootfs và đúng kernel đang chạy không |
| Invalid module format | Kernel ABI/vermagic, kiến trúc và cấu hình module |
| Unknown symbol | Dependency/module hoặc API kernel không tương thích |
| Driver probe lỗi | DT, clock/reset, nguồn, firmware và bus thực |
| Sửa app nhưng chạy bản cũ | Full build, checksum image vừa chép và nguồn boot thực tế |

Đọc lỗi đầu tiên có ý nghĩa và log task được build báo. Không xóa toàn bộ cache hoặc project ngay khi gặp lỗi; lưu bằng chứng rồi sửa đúng nguyên nhân.

## 12. Bộ bàn giao tối thiểu

- Hướng dẫn chung này và tài liệu board đã điền đủ thông số.
- XSA/bitstream hoặc BSP hợp lệ; nguồn/layer/recipe/patch có revision.
- Project configs, rootfs config, device tree và kernel fragments.
- Bộ image cùng lần build, checksum và thông tin rootfs/boot.
- Log build/package/UART và báo cáo PASS/FAIL thực tế.
- Bài test driver/app cùng đầu vào và kết quả mong đợi.

Đội tiếp nhận nên boot bộ image tham chiếu đã kiểm thử trước nếu có, rồi boot bộ tự build và so sánh cùng bài test. Nếu chưa có image/log tham chiếu, ghi rõ chưa có thay vì suy ra từ mã nguồn.

## 13. Tài liệu AMD tham khảo

Chọn đúng phiên bản trong AMD Docs khi thay release:

- [Yêu cầu cài đặt PetaLinux 2023.2][requirements]
- [Cài đặt công cụ][install]
- [Ví dụ tạo project][create]
- [Build system image][build]
- [Cấu hình U-Boot và đóng gói boot][package]
- [UG1144 — PetaLinux Tools Reference Guide](https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide)

[requirements]: https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Installation-Requirements
[install]: https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Installing-the-PetaLinux-Tool
[create]: https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/petalinux-create-t-project-Examples
[build]: https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Building-a-PetaLinux-System-Image
[package]: https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Configuring-U-Boot
