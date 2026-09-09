# ADRV9029 Board Note

> Board-specific notes for building, configuring and checking the **ADRV9029** platform.
>
> Baseline used by this repository: **PetaLinux 2023.2**, Zynq UltraScale+ MPSoC **XCZU15EG**, ADI software baseline **2023_R2**.
>
> General PetaLinux installation/build/boot steps are described in [PetaLinux_Common_Guide.md](PetaLinux_Common_Guide.md). Clock SI5518 and board applications are described in [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md).

---

## 1. Scope

This file contains only the parts that are specific to the ADRV9029 board/project:

- Tool/release baseline used for this board.
- Restoring `project-spec` into a PetaLinux project.
- Cloning Analog Devices sources.
- ADI Yocto layer path.
- ADRV9029 custom driver path and build method.
- Kernel configuration checklist.
- Device-tree path.
- Firmware/profile path.
- Rootfs packages.
- Board-side driver/JESD/IIO checks.
- Recommended workflow when porting the driver to another project.

Do not use this file as a replacement for the common PetaLinux guide.

---

## 2. Repository data

The repository contains:

```text
ADRV/
├── README.md
├── PetaLinux_Common_Guide.md
├── ADRV9029 board note.md
├── SI5518_and_Apps_Guide.md
├── project-spec.zip
└── adrv-firmware.zip
```

`project-spec.zip` contains the board/project-specific PetaLinux configuration and custom software. The important paths restored under the PetaLinux project are:

```text
project-spec/meta-user/
├── conf/
│   └── petalinuxbsp.conf
├── recipes-bsp/
│   └── device-tree/
│       └── files/
│           └── system-user.dtsi
├── recipes-modules/
│   └── adrv9025-custom/
│       ├── adrv9025-custom.bb
│       └── files/
│           └── adrv902x/
└── recipes-apps/
    ├── adrv-firmware/
    ├── si5518config/
    ├── tx-dma/
    ├── rx-dma/
    └── dpd-app/
```

The ADRV9029 Linux driver in this project is intentionally named **`adrv9025-custom`**. The ADI API and many firmware/device names still use ADRV9025/ADRV9026 naming; do not rename them only for cosmetic consistency.

---

## 3. Install the required tools

Use the same release for Vivado/XSA and PetaLinux whenever possible.

For this project the reference release is:

```text
Vivado / Vitis : 2023.2
PetaLinux      : 2023.2
ADI release    : 2023_R2
Architecture   : Zynq UltraScale+ MPSoC / zynqMP
Device         : XCZU15EG
```

Install PetaLinux 2023.2 following AMD UG1144, then activate it before every build terminal:

```bash
source <PETALINUX_2023_2_INSTALL_DIR>/settings.sh

petalinux-util --version
which petalinux-build
which petalinux-config
```

Expected result: the tools resolve from the **2023.2** installation.

Do not mix a 2023.2 project with a random newer PetaLinux environment unless the purpose is explicitly to port/migrate the project.

---

## 4. Prepare a working directory

Example:

```bash
mkdir -p ~/work/adrv9029
cd ~/work/adrv9029
```

Recommended layout:

```text
~/work/adrv9029/
├── adi/
│   ├── linux/
│   ├── meta-adi/
│   └── hdl/                  # optional
└── petalinux/
    └── adrv9029/
```

Keep ADI repositories outside `build/tmp` and outside generated Yocto work directories.

---

## 5. Clone Analog Devices sources

### 5.1. ADI Linux kernel

For the 2023.2 baseline use the ADI **2023_R2** branch/release:

```bash
cd ~/work/adrv9029/adi

git clone --branch 2023_R2 https://github.com/analogdevicesinc/linux.git
cd linux
git rev-parse HEAD
git status
```

Record the commit SHA used for every validated build.

The upstream ADRV9025/ADRV902x driver is located under:

```text
linux/drivers/iio/adc/adrv902x/
```

Use this tree mainly as the upstream/reference source when reviewing or porting the board's custom driver.

### 5.2. ADI Yocto layer

```bash
cd ~/work/adrv9029/adi

git clone --branch 2023_R2 https://github.com/analogdevicesinc/meta-adi.git
cd meta-adi
git rev-parse HEAD
```

The PetaLinux/ADI layer that is added to the project is:

```text
~/work/adrv9029/adi/meta-adi/meta-adi-xilinx
```

ADI documents `meta-adi-xilinx` as the Yocto layer used to integrate the ADI Linux kernel, device trees and userspace tools into Xilinx/PetaLinux projects.

### 5.3. ADI HDL — optional

Only needed when rebuilding/comparing the ADI reference HDL/XSA:

```bash
cd ~/work/adrv9029/adi

git clone --branch hdl_2023_r2 https://github.com/analogdevicesinc/hdl.git
```

For normal driver porting with an already validated XSA, cloning the HDL repository is optional.

---

## 6. Create/restore the PetaLinux project

Create a ZynqMP project:

```bash
cd ~/work/adrv9029/petalinux

petalinux-create -t project --template zynqMP --name adrv9029
cd adrv9029
```

Restore the repository's `project-spec.zip` into the project root so that the resulting path is:

```text
<PROJECT_DIR>/project-spec/meta-user/...
```

Example:

```bash
unzip <PATH_TO_ADRV_REPO>/project-spec.zip -d <PROJECT_DIR>
```

Before overwriting an existing project, back up its current `project-spec` and compare differences.

If the project is being recreated from XSA, import the correct hardware description:

```bash
petalinux-config --get-hw-description=<DIRECTORY_CONTAINING_XSA>
```

The XSA must correspond to the ADRV9029 hardware design and the bitstream that will be booted.

---

## 7. Add the ADI Yocto layer

Run:

```bash
cd <PROJECT_DIR>
petalinux-config
```

Go to:

```text
Yocto Settings
  -> User Layers
```

Add:

```text
/home/<user>/work/adrv9029/adi/meta-adi/meta-adi-xilinx
```

Use an absolute path where possible.

After configuration, confirm that the layer is visible in the generated Yocto configuration. Do **not** edit recipes directly inside `build/tmp`.

Important:

- `meta-adi` is the upstream/vendor layer.
- `project-spec/meta-user` is where board/project customization should live.
- Do not modify the cloned `meta-adi` directly for board-specific changes; use `meta-user` recipes/bbappends instead.

---

## 8. ADRV9029 custom driver

### 8.1. Driver path in this project

The board-specific driver recipe is:

```text
project-spec/meta-user/recipes-modules/adrv9025-custom/adrv9025-custom.bb
```

Driver/API source is under:

```text
project-spec/meta-user/recipes-modules/adrv9025-custom/files/adrv902x/
```

This is the driver that should be treated as the **current project implementation** when reproducing the existing ADRV9029 system.

Do not blindly copy the latest `adrv9025.c` from ADI `main` into this folder. Port changes selectively and keep the working baseline reproducible.

### 8.2. Compare custom driver with ADI upstream

Useful when porting:

```bash
diff -ruN \
  ~/work/adrv9029/adi/linux/drivers/iio/adc/adrv902x \
  <PROJECT_DIR>/project-spec/meta-user/recipes-modules/adrv9025-custom/files/adrv902x \
  > /tmp/adrv902x-custom-vs-adi.diff
```

Review especially:

- Kernel API changes.
- SPI APIs.
- IIO APIs.
- JESD204 interfaces.
- DMA/buffer APIs.
- Clock framework calls.
- GPIO APIs.
- Device-tree property handling.
- DPD extensions.

Do not port by fixing compiler errors only. A driver that compiles can still fail probe, JESD link initialization or IIO streaming.

---

## 9. Kernel configuration checklist

Open the kernel configuration:

```bash
petalinux-config -c kernel
```

The exact symbol names can vary with the kernel/ADI release, so use menu search (`/`) and verify each dependency in the actual tree.

The ADRV9029 platform normally requires the following classes of support:

| Function | Kernel configuration to verify |
|---|---|
| SPI controller | Xilinx/ZynqMP SPI controller enabled |
| SPI userspace for SI5518 | `CONFIG_SPI_SPIDEV` when using the current SI5518 app |
| IIO core | Industrial I/O support enabled |
| IIO buffers | Buffer support required by RX capture |
| JESD204 framework | JESD204 framework/top-device support |
| AXI DMAC | ADI AXI DMAC support |
| AXI JESD RX/TX | ADI/Xilinx JESD204 RX and TX cores |
| AXI transceiver | ADI AXI transceiver support |
| ADC/DAC cores | ADI AXI ADC/DAC/IIO cores used by the XSA |
| CMA/DMA | CMA/DMA support sized for capture/transmit buffers |
| GPIO | GPIO support required by board reset/control signals |
| GPIO sysfs | Required by the current SI5518 userspace code if it still uses legacy sysfs GPIO |
| DebugFS | Useful/required for some debug/DPD attributes |
| Module loading | Required when `adrv9025-custom` is built as a module |

### Stock driver vs custom driver

This project already provides `adrv9025-custom` as an out-of-tree/module recipe.

Before enabling the stock in-kernel `CONFIG_ADRV9025`, determine whether both drivers would match the same SPI `compatible` string. Do **not** allow two ADRV9025/9029 drivers to compete for the same device.

Recommended reproduction flow:

1. Keep the known project `adrv9025-custom` recipe.
2. Build/test that baseline first.
3. Only then test an upstream/in-kernel driver in a separate controlled configuration.

---

## 10. Root filesystem configuration

Open:

```bash
petalinux-config -c rootfs
```

Enable the project packages required for the board:

```text
adrv9025-custom
adrv-firmware
si5518config
tx-dma
rx-dma
dpd-app
kernel-modules
```

Also include libiio userspace tools. In the PetaLinux menu this commonly includes:

```text
libiio
libiio-tests
```

After boot, verify that commands such as `iio_info`/`iio_attr` are actually present; enabling the library alone does not guarantee that all tools are installed.

---

## 11. Board firmware and profiles

The ADRV firmware recipe is located at:

```text
project-spec/meta-user/recipes-apps/adrv-firmware/adrv-firmware.bb
```

Associated files are under:

```text
project-spec/meta-user/recipes-apps/adrv-firmware/files/
```

The image must install the exact ADRV firmware/profile files required by the custom driver, normally under:

```text
/lib/firmware/
```

The project also provides the board initialization script:

```text
/usr/bin/adrv-firmware.sh
```

Do not assume that `adrv-firmware.zip` at repository root is automatically identical to the firmware set inside `project-spec`. Compare filenames/checksums before replacing a validated set.

Typical checks on the board:

```bash
ls -lah /lib/firmware | grep -Ei 'adrv|9025|9029'
ls -l /usr/bin/adrv-firmware.sh
sha256sum /lib/firmware/*ADRV* 2>/dev/null
```

---

## 12. Device tree

Board-specific device-tree changes should be maintained in:

```text
project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

The corresponding recipe/bbappend is under:

```text
project-spec/meta-user/recipes-bsp/device-tree/
```

Do not edit the generated files under `build/tmp` because they can be overwritten on the next build.

For ADRV9029 verify at least:

- ADRV SPI node and `compatible` string.
- SPI chip select and maximum frequency.
- Reference/device clocks.
- SYSREF/reset/control GPIOs.
- JESD204 links.
- AXI transceiver nodes.
- AXI JESD RX/TX nodes.
- AXI ADC/DAC cores.
- RX/TX DMA nodes.
- Interrupts.
- Reserved memory/CMA if required.
- `status = "okay"` on all required blocks.

The clock SI5518 is board-specific and is described separately in [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md).

---

## 13. Important Yocto/PetaLinux paths

| Purpose | Path |
|---|---|
| User project layer | `<PROJECT_DIR>/project-spec/meta-user/` |
| User layer config | `<PROJECT_DIR>/project-spec/meta-user/conf/petalinuxbsp.conf` |
| Device-tree customization | `<PROJECT_DIR>/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` |
| ADRV custom driver recipe | `<PROJECT_DIR>/project-spec/meta-user/recipes-modules/adrv9025-custom/adrv9025-custom.bb` |
| ADRV custom driver source | `<PROJECT_DIR>/project-spec/meta-user/recipes-modules/adrv9025-custom/files/adrv902x/` |
| Firmware recipe | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/adrv-firmware/` |
| SI5518 app | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/si5518config/` |
| TX app | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/tx-dma/` |
| RX app | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/rx-dma/` |
| DPD app | `<PROJECT_DIR>/project-spec/meta-user/recipes-apps/dpd-app/` |
| External ADI Yocto layer | `<ADI_DIR>/meta-adi/meta-adi-xilinx/` |
| ADI upstream ADRV driver | `<ADI_DIR>/linux/drivers/iio/adc/adrv902x/` |
| Generated PL device tree | `<PROJECT_DIR>/components/plnx_workspace/device-tree/device-tree/pl.dtsi` |
| Build output images | `<PROJECT_DIR>/images/linux/` |

For debugging Yocto recipes, generated/work files will appear under `build/tmp`, but those files are build artifacts and must not be used as the permanent source of a fix.

---

## 14. Build sequence

### 14.1. Build the custom ADRV module

```bash
cd <PROJECT_DIR>

petalinux-build -c adrv9025-custom
```

If recipe/source changes are not being picked up:

```bash
petalinux-build -c adrv9025-custom -x cleansstate
petalinux-build -c adrv9025-custom
```

### 14.2. Build firmware/apps

```bash
petalinux-build -c adrv-firmware
petalinux-build -c si5518config
petalinux-build -c tx-dma
petalinux-build -c rx-dma
petalinux-build -c dpd-app
```

### 14.3. Build device tree/kernel when changed

```bash
petalinux-build -c device-tree
petalinux-build -c kernel
```

### 14.4. Full image

```bash
petalinux-build
```

Always perform the final full build before creating the image that will be tested on the board.

Packaging/SD boot steps are in [PetaLinux_Common_Guide.md](PetaLinux_Common_Guide.md).

---

## 15. Boot order for this board

The recommended board-side sequence is:

```text
Boot PetaLinux
   |
   v
Check SD/rootfs/firmware
   |
   v
Configure SI5518
   |
   v
Verify reference clocks
   |
   v
Load/init ADRV9029
   |
   v
Check JESD204 links
   |
   v
Check IIO devices
   |
   v
Run RX/TX test
   |
   v
Run DPD test if required
```

Do not use TX/RX failure as the first diagnostic signal if SI5518 or JESD initialization has not already been confirmed.

---

## 16. Board-side driver checks

### 16.1. Basic Linux information

```bash
uname -a
cat /etc/os-release
cat /proc/cmdline
```

### 16.2. ADRV module

```bash
find /lib/modules/$(uname -r) -type f | grep -Ei 'adrv9025|adrv9029'
modinfo adrv9025_custom 2>/dev/null || true
lsmod | grep -Ei 'adrv9025|adrv9029'
```

The exact module filename should be confirmed from the recipe/build output; recipe name and kernel module name do not always have to be identical.

### 16.3. SPI device/driver binding

```bash
for d in /sys/bus/spi/devices/spi*; do
    [ -e "$d" ] || continue
    echo "=== $d ==="
    cat "$d/modalias" 2>/dev/null
    readlink -f "$d/driver" 2>/dev/null
    readlink -f "$d/of_node" 2>/dev/null
done
```

### 16.4. Kernel log

```bash
dmesg | grep -Ei 'adrv|9025|9029|jesd|iio|dmac|axi|spi'
```

Look for:

- Firmware load failure.
- Probe failure.
- SPI communication error.
- Clock not found/not locked.
- JESD state-machine error.
- DMA/IIO registration failure.

### 16.5. IIO devices

```bash
ls -l /sys/bus/iio/devices/
iio_info 2>/dev/null | tee /tmp/iio_info.txt
```

Identify which `iio:deviceN` corresponds to:

- ADRV9029 transceiver.
- AXI RX core.
- AXI TX/DDS core.

Do not hard-code an `iio:deviceN` index unless it has been confirmed on the exact image; numbering can change.

### 16.6. JESD

If the ADI `jesd-status` tool is installed:

```bash
jesd_status 2>/dev/null || true
```

Also inspect kernel logs and available JESD sysfs/debugfs nodes. A driver probe alone does not prove that JESD links are healthy.

---

## 17. Recommended driver-porting workflow

When another team wants to port the ADRV9029 driver, use the following order.

### Step 1 — Reproduce the current board

First build and run the current project without changing the ADRV driver.

Required evidence:

- Linux boots.
- SI5518 is configured and clocks are valid.
- `adrv9025-custom` probes.
- Firmware is loaded.
- JESD links are valid.
- IIO devices appear.
- At least one known RX/TX test works.

This creates a working reference before porting.

### Step 2 — Freeze the versions

Record:

```bash
petalinux-util --version
git -C <ADI_DIR>/linux rev-parse HEAD
git -C <ADI_DIR>/meta-adi rev-parse HEAD
```

Also record:

- XSA/bitstream checksum.
- `project-spec` commit/version.
- ADRV firmware/profile checksums.
- SI5518 firmware/config checksums.

### Step 3 — Compare custom vs upstream

Generate and review a diff between:

```text
ADI linux/drivers/iio/adc/adrv902x/
```

and:

```text
meta-user/recipes-modules/adrv9025-custom/files/adrv902x/
```

Separate changes into:

1. Required kernel compatibility changes.
2. Board-specific changes.
3. ADRV9029/DPD feature changes.
4. Temporary/debug changes that should not be ported.

### Step 4 — Port incrementally

Port one dependency class at a time:

```text
compile
 -> module load
 -> SPI probe
 -> firmware load
 -> JESD initialization
 -> IIO registration
 -> RX capture
 -> TX playback
 -> DPD
```

Do not jump directly from "module compiles" to RF functional testing.

---

## 18. Common mistakes

### Driver builds but is not in rootfs

Check that the package is enabled in `petalinux-config -c rootfs`, then run a full build.

### Driver exists but does not probe

Check device tree `compatible`, SPI bus/chip-select, reset, clocks and firmware path.

### Two ADRV drivers are enabled

Do not enable both the stock in-kernel ADRV driver and the custom driver if they bind to the same device.

### Editing `build/tmp`

Changes under `build/tmp` are temporary. Move permanent changes into `project-spec/meta-user`.

### Cloning ADI `main` for a 2023.2 project

`main` changes over time and can introduce kernel/API dependencies unrelated to this baseline. Start from **2023_R2**, then port newer commits intentionally.

### IIO device number is hard-coded

`iio:deviceN` numbering is not guaranteed. Detect devices by name/sysfs instead.

### Testing ADRV before clock/JESD prerequisites

On this board SI5518 configuration is part of the bring-up dependency chain. Follow [SI5518_and_Apps_Guide.md](SI5518_and_Apps_Guide.md).

---

## 19. Minimum handover information

When giving this project to another team, provide:

- PetaLinux/Vivado version: **2023.2**.
- XSA + bitstream used by the tested image.
- `project-spec` revision.
- ADI Linux branch + exact commit.
- `meta-adi` branch + exact commit.
- Custom driver source revision.
- ADRV firmware/profile checksums.
- SI5518 firmware/config/patch checksums.
- Boot mode and SD contents.
- Known-good boot log.
- Known-good JESD status/log.
- Known-good RX/TX test and expected result.

Without these version pins, another team may be able to build an image but still not reproduce the known-good board behavior.

---

## 20. References

- [PetaLinux Common Guide](PetaLinux_Common_Guide.md)
- [SI5518 and ADRV9029 Apps Guide](SI5518_and_Apps_Guide.md)
- ADI Linux: <https://github.com/analogdevicesinc/linux>
- ADI meta-adi: <https://github.com/analogdevicesinc/meta-adi>
- ADI HDL: <https://github.com/analogdevicesinc/hdl>
- AMD PetaLinux 2023.2 UG1144: <https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide>

### Release note

ADI **2023_R2** is the matching ADI software/HDL generation for the 2023.2 Xilinx tool release. After a successful board build, replace branch-only references in the handover record with the exact tested commit SHAs.
