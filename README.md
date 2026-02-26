# Begonia OTG Kernel with SukiSU

Custom Linux kernel for Xiaomi Redmi Note 8 Pro (begonia) with **OTG power fix** and **SukiSU root** support.

## Problem Solved

The stock kernel does not provide VBUS power to USB OTG devices that trigger **Debug Accessory Mode** (devices with Rd resistors on both CC pins, like wireless microphone receivers). This kernel fixes that by:

1. Forcing **Source (SRC) mode** - phone always provides power
2. Enabling **VBUS output for Debug Accessory Mode**
3. Triggering **USB host mode** for debug accessory devices
4. Disabling **VCONN** to prevent overcurrent faults
5. Bypassing **legacy cable detection** that caused disconnect loops

## Features

- **SukiSU v4.1.x** (builtin branch) - Kernel-based root solution
- **OTG Power Fix** - 5V/1500mA VBUS for debug accessory devices
- **USB Host Mode** - Proper device enumeration for audio/input devices
- **Stable Connection** - No connect/disconnect loops

## Supported Devices

| Device | Codename | SoC |
|--------|----------|-----|
| Xiaomi Redmi Note 8 Pro | begonia | MediaTek Helio G90T (MT6785) |

## Requirements

- Unlocked bootloader
- Custom recovery (TWRP, OrangeFox, etc.)
- Android 14/15/16 AOSP-based ROM
- Magisk (optional, for additional root features)

## Installation

1. Download the latest release: `begonia-otg-sukisu-final.zip`
2. Boot to recovery mode
3. Flash the zip
4. Reboot

## Verified Working

- Wireless microphone receivers (Jieli Technology USB Composite Device)
- USB audio devices
- USB input devices

## Technical Details

### Changes Made

| File | Description |
|------|-------------|
| `arch/arm64/boot/dts/mediatek/mt6360_pd.dtsi` | Set `role_def=2` (SRC Only), disable VCONN |
| `drivers/misc/mediatek/typec/tcpc_begonia/tcpc_mt6360.c` | Disable VCONN at hardware level |
| `drivers/misc/mediatek/typec/tcpc_begonia/tcpci_typec.c` | Enable VBUS for debug accessory mode |
| `drivers/extcon/mediatek/usb-tcpc.c` | Enable OTG host mode for debug accessory |
| `drivers/misc/mediatek/typec/tcpc_begonia/tcpci_alert.c` | Set USB HOST role for debug accessory |
| `drivers/misc/mediatek/typec/tcpc_begonia/inc/tcpci_core.h` | Increase legacy cable suspect threshold |
| `drivers/usb/mtu3/mtu3_dr.c` | Set 1500mA OTG boost current limit |

### Type-C Role Values

The MT6360 Type-C controller uses these role definitions:

| Value | Role | Description |
|-------|------|-------------|
| 0 | UNKNOWN | Undefined |
| 1 | SNK | Sink only (consume power) |
| 2 | SRC | Source only (provide power) ← **Our setting** |
| 3 | DRP | Dual Role Port |
| 4 | TRY_SRC | Prefer source mode |
| 5 | TRY_SNK | Prefer sink mode |

### Debug Accessory Mode

USB Type-C Debug Accessory Mode is triggered when **both CC pins have Rd resistors**. This is used for:
- Factory test equipment
- Debug dongles
- Some wireless audio receivers

The Type-C spec doesn't define VBUS behavior for this mode. Most implementations don't provide power. Our fix enables 5V/1500mA VBUS so OTG devices work.

### VCONN Explained

VCONN powers e-marked cables and active accessories. It was causing overcurrent faults because:
1. The mic receiver doesn't need VCONN
2. MT6360 detected a fault and disconnected

We disabled VCONN completely since it's not needed for passive OTG devices.

## Build Instructions

### Prerequisites

- Arch Linux (or similar)
- clang, lld (system toolchain)
- aarch64-linux-gnu-gcc (cross-compiler)
- arm-linux-gnueabi-gcc (32-bit cross-compiler)

### Build

```bash
# Clone repository
git clone https://github.com/ishan-parihar/begonia_kernel_dev.git
cd begonia_kernel_dev

# Build
chmod +x build.sh
./build.sh -b

# Output: out/arch/arm64/boot/Image.gz-dtb
```

### Create Flashable ZIP

```bash
cp out/arch/arm64/boot/Image.gz-dtb /path/to/AnyKernel3/
cd /path/to/AnyKernel3/
zip -r9 ../kernel.zip *
```

## Kernel Configuration

Key config options enabled:

```
CONFIG_KSU=y                    # SukiSU root
CONFIG_KPM=n                    # KPM disabled for stability
CONFIG_KSU_MANUAL_HOOK=n        # Manual hook disabled
CONFIG_TYPEC=y                  # Type-C support
CONFIG_USB_MTU3=y               # MediaTek USB3 controller
CONFIG_USB_GADGET=y             # USB gadget support
CONFIG_USB_HOST=y               # USB host support
```

## Troubleshooting

### Device not detected

1. Check VBUS with USB meter (should be ~5V)
2. Check dmesg: `dmesg | grep -iE "tcpc|typec|usb"`
3. Verify kernel version: `uname -a` (should show `PoWeR` in version)

### Audio not working

1. Check device detection: `cat /proc/asound/cards`
2. USB audio device should appear as card 1+

### Root not working

1. Install SukiSU manager from: https://github.com/SukiSU-Ultra/SukiSU-Ultra
2. Grant permissions in manager app

## Credits

- **SukiSU-Ultra** - https://github.com/SukiSU-Ultra/SukiSU-Ultra
- **WuXing90** - Original kernel source
- **MediaTek** - MT6785 documentation

## License

GPL-2.0-only

## Disclaimer

USE AT YOUR OWN RISK. I am not responsible for any damage to your device.
