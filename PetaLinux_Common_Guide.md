# PetaLinux Common Guide
## Hướng dẫn chung: môi trường → build → nạp/boot → kiểm tra

## 1. Quy ước 

- **HOST:** máy Linux để cài công cụ, build, chuẩn bị thẻ.
- **BOARD:** Linux chạy trên phần cứng đích.
- **U-BOOT:** console bootloader, trước khi Linux khởi động.
- Chạy lệnh HOST bằng Bash, tài khoản thường; chỉ dùng sudo cho thao tác hệ thống cần quyền.


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

Kiểm tra linux trên máy hiện tại - với máy đã sử dụng là Petalinux 2023.2


## 4. Tạo project và nhập phần cứng — HOST

### 4.1. Cách A: project mới từ template và XSA

Ví dụ cho ZynqMP; đổi template thành `zynq` nếu dùng Zynq-7000:

```bash
petalinux-create -t project --template "_template_" --name "_template_"

petalinux-config --get-hw-description="$xsa_path"
```
Template đúng với loại CPU architecture trên thiết kế, vd: Zynq, ZynqMP, versal, ...

xsa_path phải là thư mục chứa XSA đúng board;

XSA/bitstream phải cùng thiết kế. Sau khi thay XSA, nhập lại và kiểm tra các tùy chỉnh device tree còn khớp.

### 4.2. Cách B: project từ BSP

Chọn cách này thay cho cách A khi đã có BSP phù hợp:

```bash
cd "$PLNX_WORK_DIR"
petalinux-create -t project -s "<BSP_PATH>"
```


Chi tiết tại [ví dụ tạo project][create].

## 5. Cấu hình phần mềm — HOST

Từ thư mục project:

```bash
cd "$PROJECT_DIR"
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

Build / build thành phần để khoanh vùng sau khi sửa:

```bash
petalinux-build -c kernel
petalinux-build -c device-tree
petalinux-build -c <RECIPE_NAME>
petalinux-build
```

Thay RECIPE_NAME bằng recipe thật. Sau build riêng driver/app, chạy full build để cập nhật image/rootfs trước khi nạp.

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
petalinux-package --boot --fsbl zynqmp_fsbl.elf --fpga system.bit --pmufw pmufw.elf --atf bl31.elf --u-boot u-boot.elf "--option"

```


Tham khảo [AMD: cấu hình U-Boot và đóng gói][package]. Luồng Versal/PDI hoặc flash layout khác cần lệnh riêng.

## 8. Nạp qua boot

### 8.1 Nạp qua SD card và boot

Chuẩn bị SD Card -> Copy 3 file BOOT.bin, boot.scr, image.ub vào SD Card -> Đưa vào mạch -> Chuyển mạch qua SD boot Mode

### 8.2 Nạp qua JTAG & Ethernet

Chưa thực hiện và tìm hiểu

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
