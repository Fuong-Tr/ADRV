# SI5518 và các ứng dụng trên bo ADRV9029

Cập nhật: **09/09/2026**. Áp dụng cho **project-spec.zip** trong repository [Fuong-Tr/ADRV](https://github.com/Fuong-Tr/ADRV).

Tài liệu này trình bày phần riêng của bo: clock **SI5518**, khởi tạo ADRV/JESD bằng script, phát/thu DMA và DPD TX4. Phần cài PetaLinux, phục hồi source, build, đóng gói image và kiểm tra driver nằm trong [Petalinux My Guide.txt](Petalinux%20My%20Guide.txt).

Các thông số và hành vi bên dưới được đối chiếu với mã nguồn/XSA. **Chưa có log chạy hoặc kết quả đo trên bo được cung cấp để xác nhận PASS.** Mẫu output chỉ dùng làm tiêu chí kiểm tra.

## 1. Thành phần và quan hệ phụ thuộc

| Thành phần | Vai trò | Cách truy cập phần cứng |
|---|---|---|
| si5518config | Reset, nạp firmware/config cho clock SI5518, đọc trạng thái | SPI spidev và GPIO sysfs |
| adrv-firmware | Recipe cài firmware/profile ADRV; chứa script khởi tạo | modprobe và IIO sysfs/JESD FSM |
| adrv9025-custom | Driver ADRV9029 có mở rộng DPD | SPI, API ADI, JESD204, IIO |
| tx-dma | Nạp waveform vào DDR và phát lặp; bật/tắt RF/PA | /dev/mem, thanh ghi DMA/DAC/GPIO; gọi iio_attr để tắt DDS |
| rx-dma | Thu I/Q một trong RX1–RX4, ghi capture.txt | IIO sysfs và /dev/iio:deviceN |
| dpd-app | Quét gain ORX và điều khiển DPD cho TX4 | Các thuộc tính sysfs của driver tùy chỉnh |

Thứ tự chạy trên bo tham chiếu:

1. Boot đúng bộ image và xác nhận file trên SD.
2. Cấu hình SI5518; xác nhận clock thực.
3. Khởi tạo ADRV9029/JESD; kiểm tra trạng thái link.
4. Thử TX và RX với tín hiệu tham chiếu đã biết.
5. Nếu cần DPD, chuẩn bị phản hồi TX4/PA về ORX4 rồi chạy DPD.

Clock/JESD là điều kiện của thử TX/RX. Việc tạo ứng dụng bằng petalinux-create chỉ tạo recipe và source mẫu; phải có phần triển khai thật và đưa gói vào rootfs.

## 2. Build và các điều kiện trên Linux

### 2.1. Các recipe đã có

Đường dẫn dưới project-spec/meta-user:

| Recipe | Mã/file chính |
|---|---|
| recipes-apps/si5518config/si5518config.bb | files/si5518config.c, files/si5518config.h |
| recipes-apps/adrv-firmware/adrv-firmware.bb | files/adrv-firmware.sh và firmware/profile |
| recipes-apps/tx-dma/tx-dma.bb | files/tx-dma.c |
| recipes-apps/rx-dma/rx-dma.bb | files/rx-dma.c |
| recipes-apps/dpd-app/dpd-app.bb | files/dpd-app.c |
| recipes-modules/adrv9025-custom/adrv9025-custom.bb | files/adrv902x/adrv9025.c và API ADI |

Đã có source nên không tạo lại các app cùng tên bằng template.

**HOST**, từ project đã phục hồi theo hướng dẫn chính:

~~~bash
petalinux-config -c rootfs
~~~

Bật sáu gói trên cùng kernel-modules. Để tx-dma gọi được iio_attr, thêm bộ tiện ích libiio; trong menu rootfs tìm libiio và gói công cụ **libiio-tests** nếu được recipe cung cấp. Kết quả bắt buộc kiểm tra trên bo là có lệnh iio_attr; chỉ bật thư viện libiio chưa chứng minh công cụ đã được cài.

~~~bash
petalinux-build -c si5518config
petalinux-build -c adrv-firmware
petalinux-build -c tx-dma
petalinux-build -c rx-dma
petalinux-build -c dpd-app
petalinux-build -c adrv9025-custom
petalinux-build
~~~

Sau khi sửa source, full build rồi đóng gói/nạp image theo hướng dẫn chính. Build từng recipe không tự thay đổi image đang chạy trên bo.

### 2.2. Kernel và quyền truy cập

Ngoài các driver ADRV/JESD/DMA/IIO trong hướng dẫn chính:

| Điều kiện | Ứng dụng phụ thuộc |
|---|---|
| CONFIG_SPI_SPIDEV và controller SPI đúng | si5518config |
| GPIO sysfs, thường là CONFIG_GPIO_SYSFS | si5518config dùng đường dẫn GPIO kiểu cũ |
| /dev/mem và chính sách mmap cho các vùng của thiết kế | tx-dma |
| CONFIG_DEBUG_FS khi cần thuộc tính debug IIO | Đọc thông tin DPD/debug |
| IIO buffer, ADC core, AXI DMAC | rx-dma |
| Thư mục /lib/firmware có đủ file | Driver ADRV và script khởi tạo |

**BOARD**, chạy bằng root:

~~~bash
id
for app in si5518config tx-dma rx-dma dpd-app iio_attr
do
  command -v "$app"
done
ls -l /usr/bin/adrv-firmware.sh
ls -l /dev/mem /sys/class/gpio/export
~~~

Nếu /dev/mem bị từ chối, kiểm tra cấu hình/chính sách kernel và vùng reserved-memory của bản này. Khi port, cần thiết kế lại quyền sở hữu và truy cập DMA thích hợp; ứng dụng /dev/mem hiện có gắn chặt với bo tham chiếu.

tx-dma điều khiển thanh ghi trực tiếp trong khi DT vẫn khai báo DMA cho driver Linux. Trong phiên thử, không để một chương trình IIO/DMA khác đồng thời dùng cùng đường TX. Đây là đặc điểm của app hiện tại cần tính đến khi port.

## 3. File ngoài image và thư mục lưu kết quả

### 3.1. Các file SI5518 chưa có trong project-spec.zip

si5518config.c dùng các đường dẫn cố định:

| File | Đường dẫn ứng dụng đang mở | Nguồn cung cấp |
|---|---|---|
| Firmware clock | /run/media/BOOT-mmcblk1p1/fwboot1.bin | Bộ SI5518 đã chạy trên bo |
| User configuration | /run/media/BOOT-mmcblk1p1/userboot1.bin | Frequency plan đúng bo |
| Patch clock | /run/media/BOOT-mmcblk1p1/patchboot1.bin | Bộ patch đi cùng firmware/config |

Trong source, TEST_PATCH_BOOT=1 nên lần chạy hiện tại cần cả ba file. Chúng **không được đóng gói trong recipe si5518config**. Build PetaLinux không tạo ra ba file này.

Cần lấy đúng bộ clock đã chạy tại lab và lưu checksum. Không thể xác định nội dung các file chỉ từ tên SI5518 hoặc từ tần số trong device tree.

Biến IQFileName trỏ tới TM31a.bin có khai báo trong si5518config.c nhưng main không dùng nó. Ứng dụng clock không tự đọc/phát waveform.

### 3.2. Mount trên bo

Nhãn FAT32 là BOOT không đảm bảo Linux luôn đặt tên thiết bị là mmcblk1p1. Kiểm tra thực tế:

~~~bash
cat /proc/partitions
cat /proc/mounts
ls /run/media
ls /dev/mmcblk*
~~~

Chỉ sử dụng đường dẫn mặc định sau khi thấy phân vùng SD đúng được mount tại đó:

~~~bash
export ADRV_SD="/run/media/BOOT-mmcblk1p1"
grep -F " $ADRV_SD " /proc/mounts

for f in fwboot1.bin userboot1.bin patchboot1.bin
do
  test -s "$ADRV_SD/$f" && ls -l "$ADRV_SD/$f"
done
sha256sum "$ADRV_SD/fwboot1.bin" "$ADRV_SD/userboot1.bin" \
  "$ADRV_SD/patchboot1.bin"
~~~

Nếu điểm mount khác, lựa chọn một cách nhất quán:

- Mount **đúng phân vùng đã nhận diện** vào đường dẫn ứng dụng đang dùng; hoặc
- Sửa ba đường dẫn FwFileName/UserFileName/PatchFileName trong source, build và nạp lại app/image.

Việc chỉ đổi biến ADRV_SD trong terminal không đổi các chuỗi đã biên dịch vào si5518config.

Sau khi mount đúng, tạo thư mục log cho lần chạy:

~~~bash
export ADRV_LOG="$ADRV_SD/adrv9029-logs/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$ADRV_LOG"
uname -a > "$ADRV_LOG/uname.txt"
cat /proc/cmdline > "$ADRV_LOG/cmdline.txt"
dmesg > "$ADRV_LOG/kernel-before.log"
~~~

Nếu RTC trên bo chưa đúng, ghi ngày thực tế/mã lần thử vào hồ sơ. Lưu log trên SD giúp giữ kết quả qua reboot; /tmp và rootfs INITRD không phải nơi lưu bền vững.

## 4. Cấu hình clock SI5518

### 4.1. Những giá trị gắn với bo hiện tại

| Tham số trong code | Giá trị |
|---|---|
| Spidev được mở | /dev/spidev1.0 |
| Tốc độ SPI yêu cầu | 10 MHz |
| Bits per word | 8 |
| GPIO reset theo sysfs cũ | gpio370, từ base 312 + offset 58 trong ghi chú |
| Reset phần cứng | Kéo thấp 100 ms, kéo cao rồi chờ 1 giây |
| Tệp nạp | fwboot1.bin, userboot1.bin, patchboot1.bin |

system-user.dtsi đặt spidev clock dưới **&spi0**, trong khi app mở **/dev/spidev1.0**. Số bus Linux có thể phụ thuộc alias/thứ tự đăng ký; không kết luận hai tên đó mâu thuẫn chỉ từ con số. Cần đối chiếu controller thực của thiết bị trên bo:

~~~bash
ls -l /dev/spidev*
for d in /sys/bus/spi/devices/spi*
do
  [ -d "$d" ] || continue
  printf '\n%s\n' "$d"
  cat "$d/modalias" 2>/dev/null
  readlink -f "$d/of_node"
done

for g in /sys/class/gpio/gpiochip*
do
  [ -d "$g" ] || continue
  printf '\n%s\n' "$g"
  cat "$g/label" "$g/base" "$g/ngpio" 2>/dev/null
done
~~~

Đối chiếu GPIO reset với schematic. Số global gpio370 không phải định danh ổn định trên mọi kernel/bo. Với nền tảng khác phải sửa phần ánh xạ phù hợp trước khi chạy app.

### 4.2. Chạy

Sau khi xác nhận bus, reset và ba file clock:

~~~bash
si5518config > "$ADRV_LOG/si5518.log" 2>&1
adrv_clock_rc=$?
cat "$ADRV_LOG/si5518.log"
printf 'si5518config exit=%s\n' "$adrv_clock_rc"
~~~

Code thực hiện:

1. Export/cấu hình GPIO reset; mở spidev.
2. Reset SI5518 và kiểm tra giao tiếp SIO.
3. Đọc ba file từ SD vào bộ nhớ.
4. Đọc thông tin cũ của chip.
5. Restart ở chế độ host load; gửi firmware, user configuration và patch.
6. Gửi BOOT.
7. Đọc INPUT_STATUS, PLL_STATUS và REFERENCE_STATUS; in thông tin mới.

### 4.3. Cách đánh giá

- Không có lỗi mở SPI/GPIO/file hoặc lỗi truyền command.
- Chip trả lời SIO và có thông tin firmware/config mới.
- Kiểm tra trạng thái input hợp lệ; các cờ LOSS_OF_SIGNAL, OUT_OF_FREQUENCY,
  PLL_LOSS_OF_LOCK/PLL_OUT_OF_FREQUENCY phải phù hợp trạng thái clock đã ổn định.
- Xác nhận clock thực tại các điểm tham chiếu của bo đáp ứng profile 245,76 MHz.
- Xác nhận chế độ SYSREF theo thiết kế JESD.

Exit code 0 chỉ chứng minh chương trình đi qua các bước mà nó kiểm tra.
Một số hàm chỉ in cờ trạng thái rồi trả thành công; không dùng exit code
thay cho xác nhận PLL lock/clock thực.

Source có SI5518_GenSysrefPulse(), nhưng main hiện tại **không gọi hàm đó**.
Không suy ra việc chạy si5518config tự phát một chuỗi SYSREF chủ động bằng
hàm này; chế độ SYSREF thực còn phụ thuộc frequency plan/phần cứng.

Sau khi ADRV/JESD đã chạy, không reset/nạp lại clock giữa một bài thử đang
hoạt động. Nếu cần đổi clock, dừng TX và thực hiện lại trình tự khởi tạo.

## 5. Firmware ADRV và script khởi tạo

### 5.1. Firmware khác với image Linux

| Thành phần | Nơi được sử dụng |
|---|---|
| BOOT.BIN / image.ub / boot.scr | Boot ZynqMP, FPGA và Linux |
| ADRV9025_FW.bin / ADRV9025_DPDCORE_FW.bin | Firmware xử lý bên trong ADRV9029 |
| stream_image.bin | Stream processor của ADRV |
| ActiveUseCase.profile | Cấu hình clock, sample rate, datapath, JESD |
| ActiveUtilInit.profile | Cấu hình radio/LO và calibration khởi tạo |
| RxGainTable.csv / TxAttenTable.csv | Bảng gain thu và suy hao phát |

Recipe adrv-firmware cài các file ADRV vào /lib/firmware và cài
adrv-firmware.sh vào /usr/bin. Không đổi tên ADRV9025_FW.bin thành
ADRV9029_FW.bin nếu chưa sửa mọi nơi tham chiếu.

Dùng bộ file trong project-spec.zip làm một bộ nhất quán. Không tự trộn
với adrv-firmware.zip hoặc firmware SDK khác khi chưa đối chiếu version/profile.

### 5.2. Chạy script

Thực hiện sau phần SI5518. Script trong ZIP không có shebang nên gọi rõ
bằng sh để tránh phụ thuộc cách shell xử lý file executable:

~~~bash
dmesg > "$ADRV_LOG/kernel-before-adrv.log"
sh /usr/bin/adrv-firmware.sh > "$ADRV_LOG/adrv-init.log" 2>&1
adrv_init_rc=$?
cat "$ADRV_LOG/adrv-init.log"
printf 'adrv-firmware exit=%s\n' "$adrv_init_rc"
dmesg > "$ADRV_LOG/kernel-after-adrv.log"
~~~

Hành vi hiện tại:

- modprobe DMA, transceiver, AXI XCVR, JESD TX/RX và adrv9025_custom.
- Chờ 5 giây rồi tìm IIO device có name=adrv9029.
- Nếu đã ở opt_post_running_stage và error=0, kết thúc thành công.
- Nếu chưa đạt, script xóa ring buffer dmesg cũ rồi ghi 1 vào jesd204_fsm_ctrl
  một lần; vì vậy cần lưu kernel-before-adrv.log trước khi chạy.
- Đọc state/error mỗi giây, tối đa 35 giây cho lượt chờ này.
- Báo lỗi sớm nếu FSM trở về idle với error khác 0 sau thời gian đầu.

Dấu hiệu thành công mong đợi:

~~~text
FSM DONE: already running
~~~

hoặc:

~~~text
STATE=[opt_post_running_stage], ERROR=[0]
--DONE SETUP--
~~~

Phải kiểm tra thêm log kernel và dữ liệu thực. Script không đo RF và cũng
không xác nhận chất lượng waveform.

### 5.3. Tìm PHY để dùng các lệnh tiếp theo

~~~bash
ADRV_PHY=""
for d in /sys/bus/iio/devices/iio:device*
do
  [ -f "$d/name" ] || continue
  printf '%s: ' "$d"
  cat "$d/name"
  if [ "$(cat "$d/name")" = "adrv9029" ]; then
    ADRV_PHY="$d"
  fi
done
printf 'ADRV_PHY=%s\n' "$ADRV_PHY"
~~~

Chỉ thực hiện thao tác sysfs tiếp theo khi ADRV_PHY không rỗng.

### 5.4. Khi khởi tạo thất bại

Script có các gợi ý cho error -14, -1, -5, nhưng đây là hướng khoanh vùng,
không phải chẩn đoán duy nhất từ một mã lỗi.

- -14: kiểm tra log CPU/power-up/firmware.
- -1: kiểm tra log JESD RX, CGS và timing/reset/lane.
- -5: kiểm tra clock/lane clock/PLL theo log thực tế.

Lưu log trước khi thử lại. Nếu cần làm lại từ đầu, dừng hoạt động TX,
power cycle rồi cấu hình clock và khởi tạo ADRV theo thứ tự. Không lặp
reset liên tục để cố làm lỗi biến mất.

## 6. Phát waveform bằng tx-dma

### 6.1. Cách dữ liệu đi qua FPGA

XSA đi kèm thể hiện:

- AXI TX DMA đọc DDR, xuất AXI Stream **32 bit**.
- Dữ liệu đi qua FIFO và FIR **nội suy 4**, có hai data path.
- Các lát I/Q sau FIR được ghép thành 32 bit.
- Cùng từ 32 bit đó được nhân bản vào bốn nhóm đầu vào DAC/TPL.
- DAC/TPL và JESD đưa dữ liệu tới ADRV9029.

Do đó bản phần cứng này phát cùng waveform vào bốn đường TX. Không suy ra
file chứa bốn luồng I/Q độc lập từ việc tx-dma đặt FRAME_BYTES=16.

Ứng dụng chỉ sao chép byte nguyên trạng, không đổi I/Q, scale, endianness
hay resample. File tham chiếu phải phù hợp đường 32 bit gồm hai thành phần
16 bit của thiết kế. Cần bàn giao rõ thứ tự I/Q, định dạng signed và
endianness của file đã kiểm thử.

Profile đặt tốc độ I/Q phía ADRV là 245,76 MSPS. Với nội suy 4 và luồng chạy
đúng tốc độ đó, tốc độ mẫu trước FIR suy ra là 61,44 MSPS. Đây là quan hệ
thiết kế, cần đối chiếu clock và waveform thực; không gán mặc định Fs của
file trước FIR bằng 245,76 MSPS.

### 6.2. Giới hạn và địa chỉ

| Tham số | Giá trị code |
|---|---|
| DDR phát | 0x60000000 |
| Dung lượng tối đa | 0x01000000 = 16 MiB |
| Kích thước file hợp lệ | Lớn hơn 0 và chia hết cho 16 byte |
| TX DMAC | 0x9C420000 |
| DAC/TPL | 0x84A04000 |
| GPIO RF/PA | 0x80020000 |
| Giá trị bật GPIO | 0x000FFF0F |
| Giá trị tắt GPIO | 0x00000000 |
| File lưu kích thước lần nạp | /tmp/tx-dma.size |
| DMA mode | Cyclic |

16 byte là ràng buộc kiểm tra của app. Dữ liệu vẫn đi qua AXI Stream 32 bit
trong XSA; phải xét cả phần cứng khi chuẩn bị waveform.

### 6.3. Chuẩn bị trước phát

- ADRV/JESD đã đạt kiểm tra.
- Có file waveform tham chiếu và thông tin Fs/scale/IQ.
- Đã nối các ngõ TX sử dụng đến tải hoặc thiết bị đo qua suy hao thích hợp
  với mức công suất của bo; biết GPIO app sẽ bật đường RF/PA.
- Không có ứng dụng khác dùng TX DMA/DAC.
- Kiểm tra công cụ DDS và IIO device cố định trong source.

~~~bash
command -v iio_attr
for d in /sys/bus/iio/devices/iio:device*
do
  [ -f "$d/name" ] || continue
  printf '%s: ' "$d"
  cat "$d/name"
done
cat /sys/bus/iio/devices/iio:device3/name
~~~

tx-dma.c gọi iio_attr với **iio:device3** và altvoltage0..15. Cần xác nhận
device3 đúng là DAC/DDS của thiết kế. Nếu thứ tự IIO thay đổi, sửa phần
chọn thiết bị trong source rồi build lại. App hiện không có tham số dòng
lệnh để đổi số device này.

Lệnh system() tắt DDS và một số thao tác trong app không được kiểm tra lỗi
đầy đủ. Không dùng một dòng thông báo CYCLIC MODE làm bằng chứng RF đã phát đúng.

### 6.4. Nạp, phát và dừng

Thay waveform.bin bằng file đã được bàn giao. TM31a.bin chỉ là tên xuất hiện
trong ghi chú nguồn; ZIP chưa cung cấp một waveform tham chiếu để xác nhận.

~~~bash
export ADRV_WAVE="$ADRV_SD/waveform.bin"
ls -l "$ADRV_WAVE"
sha256sum "$ADRV_WAVE"

# Dừng lượt phát trước trước khi thay dữ liệu DDR.
tx-dma stop

tx-dma load "$ADRV_WAVE" > "$ADRV_LOG/tx-load.log" 2>&1
adrv_tx_load_rc=$?
cat "$ADRV_LOG/tx-load.log"
printf 'tx load exit=%s\n' "$adrv_tx_load_rc"
~~~

Chỉ tiếp tục khi nạp đủ byte và exit=0:

~~~bash
tx-dma start > "$ADRV_LOG/tx-start.log" 2>&1
adrv_tx_start_rc=$?
cat "$ADRV_LOG/tx-start.log"
printf 'tx start exit=%s\n' "$adrv_tx_start_rc"
~~~

start không có size sẽ đọc /tmp/tx-dma.size. Có cú pháp start [size],
nhưng chỉ truyền size đúng số byte đã nạp; không dùng để đọc vùng DDR chưa
có waveform. Sau reboot cần nạp lại file.

Khi chương trình start kết thúc, DMA vẫn phát lặp. Để dừng:

~~~bash
tx-dma stop
~~~

stop dừng TX DMAC và ghi 0 vào GPIO RF/PA. Phải đo/xác nhận trạng thái RF
thực nếu đang đánh giá ngõ ra.

### 6.5. Kiểm tra kết quả TX

Ghi waveform checksum, Fs gốc, mức scale, profile, TX được đo và thiết lập
máy đo. Kiểm tra tần số trung tâm, phổ tín hiệu và công suất theo bài thử
đã thống nhất. Chụp kết quả trước/sau stop.

Không suy ra mức công suất RF từ giá trị mẫu số hoặc từ GPIO_VALUE_ON.
LO mặc định 5,6 GHz cần phù hợp phần RF ngoài chip; thay LO phải có bài
thử và cấu hình rõ ràng.

## 7. Thu I/Q bằng rx-dma

### 7.1. Cú pháp và hành vi

~~~text
rx-dma <rx1/rx2/rx3/rx4> <samples>
~~~

Có thể dùng 1/2/3/4 thay cho rx1/rx2/rx3/rx4. Mỗi lần chọn một kênh RF.

App:

1. Tìm ADC IIO có name=ad_ip_jesd204_tpl_adc.
2. Tìm PHY có name=adrv9029 và yêu cầu bật RX được chọn.
3. Tắt buffer và các scan element của lần thu trước.
4. Bật hai scan element I/Q của kênh được chọn.
5. Đặt buffer 4096 mẫu, đọc /dev/iio:deviceN.
6. Diễn giải mỗi mẫu thành I và Q signed 16-bit little-endian, 4 byte/mẫu.
7. Lưu hai cột số nguyên vào capture.txt trong thư mục hiện hành.
8. In số mẫu, RMS, peak và dBFS riêng cho I/Q.

Định dạng đọc là giả định cố định trong code, không tự phân tích scan type.
Khi port hoặc đổi ADC core, đối chiếu các file scan_elements/*_type và
*_index trước khi dùng kết quả.

### 7.2. Thử một kênh

Đưa tín hiệu đã biết vào đúng RX qua mức suy hao phù hợp; ghi mức và tần số
máy phát. Không nối thẳng đầu ra PA vào RX/ORX nếu chưa thiết kế mức vào.

Ví dụ thu 65536 cặp I/Q từ RX1:

~~~bash
mkdir -p "$ADRV_LOG/rx1"
cd "$ADRV_LOG/rx1"
rx-dma rx1 65536 > rx.log 2>&1
adrv_rx_rc=$?
cat rx.log
printf 'rx exit=%s\n' "$adrv_rx_rc"
wc -l capture.txt
head -n 5 capture.txt
~~~

Với RX khác, tạo thư mục riêng rồi đổi tên kênh trong lệnh. App mở
capture.txt ở chế độ ghi mới nên lần thu sau trong cùng thư mục sẽ ghi đè.

### 7.3. Kiểm tra

- Log phải ghi đúng kênh và Captured samples bằng số yêu cầu.
- Số dòng capture.txt phải bằng số cặp I/Q yêu cầu.
- Mẫu/phổ phải tương ứng tín hiệu thử; kiểm tra clipping, mức nền và peak.
- RMS/dBFS trong app là mức số tham chiếu full-scale 16 bit, không phải dBm
  tại cổng RF nếu chưa có hiệu chuẩn.

Có trường hợp code thoát vòng đọc do lỗi nhưng vẫn in Done và trả 0.
Vì vậy exit code hoặc dòng Done đơn lẻ chưa đủ xác nhận thu thành công.
Đánh giá cả số mẫu, log lỗi và dữ liệu.

## 8. DPD TX4 và phản hồi ORX4

### 8.1. Phạm vi

DPD bù méo phi tuyến của PA. Ở bản này, dpd-app điều khiển bộ DPD bên trong
ADRV9029 qua driver, không chạy thuật toán thích nghi trên CPU Linux.

| Tham số | Giá trị cố định của app |
|---|---|
| TX vật lý | TX4 |
| Chỉ số zero-based trong sysfs | tx3 |
| TX mask | 0x8 |
| Ngõ phản hồi được ánh xạ | ORX4 |
| ORX mask | 0x80 |
| Gain index mặc định | 190 |
| Quét gain | Cửa sổ start..start+10, bước 1, giới hạn quét 190..255 |
| File lưu gain đề xuất | /tmp/dpd_tx4_orx_gain |
| Raw status | tx3_dpd_status_raw, 112 byte được app giải mã |

Các tên ORX4/TX4 là ánh xạ trong source; đối chiếu thêm đầu nối của bo.
Chỉ số gain không phải số dB trực tiếp.

Tín hiệu TX4 cần đang phát; lấy mẫu đầu ra PA về đúng ORX qua coupler và
suy hao đã xác định. ORX cần đủ mức để thích nghi và tránh bão hòa. Mức RF
cụ thể phải theo đường phản hồi của bo và giới hạn linh kiện.

### 8.2. Kiểm tra các thuộc tính driver

ADRV_PHY lấy theo mục 5.3:

~~~bash
if [ -n "$ADRV_PHY" ]; then
  ls "$ADRV_PHY"/dpd*
  ls "$ADRV_PHY"/calibrate_ext_path_delay_en
  ls "$ADRV_PHY"/tx3_dpd_status_raw
fi
~~~

Nếu thuộc tính không xuất hiện, kiểm tra đúng module tùy chỉnh và firmware
đang dùng. Không suy ra driver chuẩn có sẵn đầy đủ các thuộc tính này.

### 8.3. Quét gain ORX

~~~bash
dpd-app orx 190 > "$ADRV_LOG/dpd-orx.log" 2>&1
adrv_orx_rc=$?
cat "$ADRV_LOG/dpd-orx.log"
printf 'dpd orx exit=%s\n' "$adrv_orx_rc"
~~~

Lệnh trên quét 190..200. Với start khác, cửa sổ kết thúc không vượt 255.
Mỗi lần thử gain, app cấu hình/reset DPD, đo trạng thái rồi tắt/reset lại.
Sau quét, tìm các dòng:

~~~text
RESULT                  : ORX_RECOMMENDED
RECOMMENDED_ORX_GAIN     : ...
RUN_COMMAND             : dpd-app run ...
~~~

Dấu ... ở khối output biểu thị giá trị thực chương trình sẽ in, không phải
tham số để sao chép vào terminal. Dùng đúng giá trị RUN_COMMAND của lần quét.

App chấm điểm gain dựa trên mức ORX và trạng thái; mục tiêu số trong code
khoảng 25% thang công suất nội bộ. Các ngưỡng này phục vụ lựa chọn của app,
không phải thông số bảo đảm của ADRV9029.

ORX_RECOMMENDED chưa chứng minh DPD đã có cập nhật tốt. Source vẫn có thể
đề xuất gain khi TX đang bão hòa; phải giải quyết lỗi TX và chạy bước sau.

### 8.4. Chạy DPD

Lệnh run không có gain sẽ đọc file gain đã lưu; nếu không có file, app dùng
190. Vì vậy chỉ dùng cách này sau khi đã quét thành công:

~~~bash
cat /tmp/dpd_tx4_orx_gain
dpd-app run > "$ADRV_LOG/dpd-run.log" 2>&1
adrv_dpd_rc=$?
cat "$ADRV_LOG/dpd-run.log"
printf 'dpd run exit=%s\n' "$adrv_dpd_rc"
~~~

Có thể chạy rõ gain theo RUN_COMMAND mà bước quét đã in.

Trình tự trong source:
- Tắt user ORX paths in_voltage4_en/in_voltage5_en.
- Tắt/reset DPD; đặt TX4 → ORX4.
- Cấu hình mô hình; đặt ORX gain; reset.
- Hiệu chuẩn external path delay.
- Cấu hình ngưỡng fault, tracking và bật tracking DPD.
- Đọc trạng thái để quyết định thành công/thất bại.

Điều kiện để app in **DPD_ENABLED**:
1. dpd_error_code = 0.
2. update_count > 0.
3. tracking enable mask có bit TX4.
4. Không có cờ action FW_SKIP_LUT_UPDATE mà code đang kiểm tra.

Nếu thất bại, app cố tắt/reset DPD. Nếu thành công, DPD tiếp tục được bật
sau khi app thoát.

Để đánh giá tác dụng RF, cần so sánh phổ/ACLR/EVM trước và sau DPD trong
cùng điều kiện đo. DPD_ENABLED là kết quả kiểm tra trạng thái của app,
không thay cho phép đo chất lượng RF.

### 8.5. Dừng DPD và kết thúc bài thử

App hiện có hai lệnh orx và run; không có subcommand stop.
Dùng thuộc tính driver để tắt/reset TX4 rồi dừng phát:

~~~bash
if [ -n "$ADRV_PHY" ]; then
  printf '0x8\n' > "$ADRV_PHY/dpd_tx_mask"
  printf '0\n' > "$ADRV_PHY/dpd_tracking_en"
  printf '1\n' > "$ADRV_PHY/dpd_reset"
fi
tx-dma stop
dmesg > "$ADRV_LOG/kernel-final.log"
sync
~~~

Kiểm tra lỗi ghi sysfs nếu có và xác nhận trạng thái kết thúc trên phần cứng.

## 9. Ma trận kiểm tra ứng dụng

| Bài thử | Điều kiện trước | Bằng chứng cần lưu | Đạt khi |
|---|---|---|---|
| SI5518 | Đúng SPI/GPIO và đủ ba file | Checksum file, log trạng thái, kiểm tra clock | Cấu hình đúng, input/PLL/clock đáp ứng thiết kế |
| ADRV/JESD | Clock sẵn sàng, đúng firmware | Log script và kernel | PHY nhận được, FSM/error đạt, không còn lỗi link |
| TX DMA | Waveform, tải/máy đo, JESD | Hash waveform, log load/start/stop, ảnh phổ | Phát/dừng đúng và tín hiệu đo phù hợp |
| RX DMA | Tín hiệu RX đã biết | rx.log, capture.txt, phổ phân tích | Đúng số mẫu và đúng tín hiệu |
| DPD TX4 | PA/ORX4 đúng, TX không bão hòa | Log quét/run, đo trước/sau | App chấp nhận update; chất lượng RF đạt bài thử |

Ghi rõ PASS, FAIL hoặc CHƯA THỬ cho từng hàng. Đừng gộp boot Linux thành
“toàn hệ thống đã chạy”. Không tự gán giới hạn ACLR/EVM/công suất khi bên
bàn giao và bên nhận chưa thống nhất bài đo.

## 10. Các điểm phải sửa khi chuyển sang bo/kernel khác

| Điểm gắn với bo | Nơi cần xem lại |
|---|---|
| /dev/spidev1.0 và gpio370 | si5518config.c; aliases/controller/GPIO trên bo đích |
| Tên mount và ba file clock | Chuỗi đường dẫn trong si5518config.c |
| Clock 245,76 MHz, SYSREF và mode JESD | Clock plan, profile, device tree, FPGA |
| Địa chỉ DAC/DMAC/DDR/GPIO RF | tx-dma.c, system-user.dtsi và XSA |
| iio:device3 dùng tắt DDS | tx-dma.c |
| Scan layout I/Q signed 16-bit LE | rx-dma.c và scan_elements của ADC mới |
| Tên IIO adrv9029 / ad_ip_jesd204_tpl_adc | Cách tìm thiết bị trong các app |
| Thuộc tính DPD, TX4/ORX4 masks | dpd-app.c và driver tùy chỉnh |
| Firmware/API/kernel ABI | Bộ nguồn ADI, module, cấu hình và image |

Giữ bộ test trên bo tham chiếu làm mốc. Cập nhật từng phần, ghi rõ source
revision và chạy lại bài kiểm tra tương ứng.

## 11. Hồ sơ còn cần lấy từ bo đang hoạt động

- Ba file SI5518 cùng checksum và bản frequency plan/chế độ SYSREF.
- Mã bo, revision, sơ đồ reset/SPI và đầu nối RF/ORX.
- Waveform tham chiếu, Fs, I/Q order, kiểu số, scale và checksum.
- Boot image đã chạy tốt, tài khoản đăng nhập và checksum.
- Log/ảnh đo của các bài thử thực tế, gồm điều kiện máy đo/suy hao.
- Giá trị gain ORX và kết quả DPD nếu DPD nằm trong phạm vi bàn giao.

Các file/giá trị này chưa được tạo thay trong lần cập nhật tài liệu.
Nguồn đối chiếu là [project-spec.zip tại mốc đã đọc](https://github.com/Fuong-Tr/ADRV/blob/2de76efbbf60f37d84ba224be8a4008353549a85/project-spec.zip), SHA-256:
743cb7e8adda47676bbd3ce0201ba544976e4b119e293019ee843a8496f84f10.

Thông tin chức năng DPD của chip: [Analog Devices ADRV9029](https://www.analog.com/en/products/adrv9029.html).

