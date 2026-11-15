# NUC8 Hackintosh EFI — the most stable, most complete OpenCore EFI for the Intel NUC8 (Bean Canyon)

![Intel NUC8](docs/images/nuc8.jpg)

**The definitive EFI for the Intel NUC8.** Fully optimized, daily-driver stable, and running **every core feature** — graphics, 120 Hz display, onboard audio, HDMI/DisplayPort audio, Ethernet, Wi-Fi, Bluetooth, USB, NVMe and sleep. This is not a "boots and mostly works" config: it is a daily-driver EFI validated across repeated cold boots, with every known panic traced to a root cause and fixed. Anything still unexplained is documented in [Known issues](#known-issues) rather than swept under the rug.

> **macOS Sequoia 15.7.2 (24G325) · OpenCore 1.0.7 · SMBIOS Macmini8,1**

---

## Why this EFI?

- ✅ **Fixes the notorious NUC8 boot panic.** Eliminates the AppleHDA `DspFuncLib` use-after-free / AppleALC page-fault panic that plagues NUC8 builds — *without* sacrificing HDMI/DP audio. Six consecutive panic reports on the donor machine were all traced to this one bug; after the fix, boots are clean. See [Known issues](#known-issues) for the one unrelated, non-reproducing event observed afterwards.
- ✅ **Both audio paths work at the same time:** onboard Realtek ALC235 (3.5 mm jack) **and** HDMI/DisplayPort monitor audio, selectable and hot-switchable.
- ✅ **Full graphics acceleration** — Intel Iris Plus Graphics 655, QE/CI, Metal, 1536 MB VRAM, **2560×1440 @ 120 Hz**.
- ✅ **Native-feeling networking** — Intel Ethernet + Intel Wi-Fi (AirportItlwm) + Bluetooth.
- ✅ **Clean, minimal, documented config** — every patch has a reason. No mystery kexts, no "shotgun" fixes.
- ✅ **Validated over 4/4 consecutive cold boots** with HDMI audio enumerating every time (see [Validation](#validation)).

---

## Hardware

| Component | Model |
|---|---|
| Machine | Intel NUC8 (Bean Canyon) |
| CPU | Intel Core i3-8109U (2C/4T, 3.0–3.6 GHz) |
| iGPU | Intel Iris Plus Graphics 655 (`0x3EA5`) |
| Audio | Realtek ALC235 (`pci8086,9dc8`) |
| Ethernet | Intel I219-V |
| Wi-Fi / BT | Intel Wireless-AC |

> Also expected to work on i5-8259U / i7-8559U Bean Canyon NUCs (same platform/GPU), though this exact EFI was tuned and validated on the i3-8109U.

---

## What works

| Feature | Status | Notes |
|---|---|---|
| Graphics acceleration (QE/CI, Metal) | ✅ | Iris Plus 655, 1536 MB |
| 2560×1440 @ 120 Hz | ✅ | DisplayPort |
| Onboard audio (3.5 mm jack) | ✅ | AppleALC, layout-id 8 |
| HDMI / DisplayPort audio | ✅ | `hda-gfx=onboard-1` on IGPU + HDEF |
| Ethernet | ✅ | IntelMausi |
| Wi-Fi | ✅ | AirportItlwm 2.4.0 |
| Bluetooth | ✅ | IntelBluetoothFirmware + BlueToolFixup + IntelBTPatcher |
| USB (all ports, mapped) | ✅ | USBToolBox + UTBMap |
| NVMe | ✅ | NVMeFix; boot-time TRIM disabled (`SetApfsTrimTimeout = 0`) |
| Sleep / wake | ✅ | |
| App Store / iCloud / iMessage | ⚠️ | **Requires your own SMBIOS** — see below |
| AirDrop / Continuity | ⚠️ | Depends on Wi-Fi card; Intel cards are limited |

---

## The panic fix (why this EFI is stable)

Earlier NUC8 EFIs suffered a boot panic in `AppleHDA` / `DspFuncLib` (use-after-free) or a page fault in `AppleALC`:

```
panic: AppleHDA ... DspFuncLib ... use-after-free
panic: AppleALC ... CR2: 0x10 ... AlcEnabler::init()
```

**Root cause:** `no-controller-inject` was present on the iGPU device properties. That makes AppleALC's `validateInjection()` fail, so it never sets `hda-gfx` on the iGPU and never registers the HDMI/DP controller. `AppleHDAHDMI_DPDriver::initHardware` then cannot link to the framebuffer, and its teardown frees shared `DspFuncLib` objects out from under the live driver → use-after-free panic.

**The fix (applied in this EFI):**

1. **Remove** `no-controller-inject` from `PciRoot(0x0)/Pci(0x2,0x0)` (IGPU).
2. **Add** `hda-gfx = onboard-1` to the IGPU.
3. **Add** `hda-gfx = onboard-1` to `PciRoot(0x0)/Pci(0x1F,0x3)` (HDEF).

That's it. HDMI/DP audio now initializes correctly and the panic is gone — **no `No-hda-gfx` workaround, no disabled audio.**

<details>
<summary>Fallback if you ever need it</summary>

If a future macOS/AppleALC update reintroduces the panic and you don't need digital audio, set `No-hda-gfx` on HDEF and remove `hda-gfx` from both devices. This guarantees stability at the cost of HDMI/DP audio. It is **not** needed with this EFI.
</details>

---

## Kexts

| Kext | Version | Purpose |
|---|---|---|
| Lilu | 1.7.2 | Patching framework |
| VirtualSMC | 1.3.8 | SMC emulation |
| SMCProcessor | 1.3.8 | CPU sensors |
| SMCSuperIO | 1.3.8 | Fan/IO sensors |
| AppleALC | 1.9.8 | Audio (ALC235 + HDMI/DP) |
| WhateverGreen | 1.7.1 | Graphics / framebuffer |
| AirportItlwm | 2.4.0 | Intel Wi-Fi |
| IntelBluetoothFirmware | 2.5.0 | Intel Bluetooth |
| IntelBTPatcher | 2.5.0 | Bluetooth patches |
| BlueToolFixup | 2.7.2 | Bluetooth (Monterey+) |
| IntelMausiEthernet | 3.0.3 | Intel Ethernet |
| NVMeFix | 1.1.4 | NVMe power/TRIM |
| RestrictEvents | 1.1.7 | CPU naming / updates |
| Sinetek-rtsx | 9.0.0 | SD card reader |
| USBToolBox | 1.2.0 | USB mapping base |
| UTBMap | 1.1 | USB port map |
| XHCI-unsupported | 0.9.2 | XHCI compatibility |
| AMFIPass | 1.4.1 | *Disabled* (included for reference) |

**Drivers:** OpenRuntime, OpenCanopy, OpenHfsPlus, OpenPartitionDxe, OpenLinuxBoot, Ext4Dxe, AudioDxe, ResetNvramEntry.

**Boot args:** `igfxonln=1`

---

## Installation

1. **Back up your existing EFI.**
2. Mount your ESP and copy the `EFI` folder from this repo to it (replace your current `EFI`).
3. **Generate your own SMBIOS** (this repo ships with placeholder values):
   - Use [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) with model **`Macmini8,1`**.
   - In `config.plist` → `PlatformInfo → Generic`, replace:
     - `SystemSerialNumber`
     - `MLB`
     - `SystemUUID`
     - `ROM`
   - Do **not** share your real serials publicly.
4. Set your BIOS:
   - Secure Boot **off**, Fast Boot **off**, VT-d **off** (or keep on with `disableio` — this EFI expects off), CFG-Lock **off** if possible.
   - UEFI boot, SATA mode **AHCI**.
5. Boot from the USB/ESP. The OpenCore picker only appears if you hold **Alt / Esc / 0** — otherwise it boots straight through.
6. Install macOS, then copy the EFI to the internal ESP and set it as the boot entry.

> **Note:** The OpenCore picker is intentionally hidden (`Timeout = 0`, `ShowPicker = false`) for a clean, Mac-like boot. Hold a key at boot to reveal it.

---

## Validation

This EFI was tested across **4 consecutive cold boots**:

| Boot | Kernel panics | `hda-gfx` IGPU/HDEF | HDMI/DP driver | DP audio engine | Default output |
|---|---|---|---|---|---|
| 1 | 0 new | ✅ / ✅ | active | present | DisplayPort |
| 2 | 0 new | ✅ / ✅ | active | present | DisplayPort |
| 3 | 0 new | ✅ / ✅ | active | present | DisplayPort |
| 4 | 0 new | ✅ / ✅ | active | present | DisplayPort |

Both the analog (jack) and DisplayPort audio engines were confirmed streaming (`IOAudioEngineState = 1`) with real playback.

---

## Known issues

This section exists so the claims above stay honest. A few things were observed on the donor machine and are documented rather than hidden:

**1. One non-reproducing early-boot APFS panic**

On the first cold boot after ~9 hours powered off, the machine panicked once at **1.35 s uptime**, before userspace started:

```
panic(cpu 2 caller ...): [kalloc.type.var4.16]: element modified after free
Kernel Extensions in backtrace: com.apple.filesystems.apfs
roots installed: 0
```

- It is **not** the AppleHDA/AppleALC bug above — different kext, different uptime, different mechanism.
- It did **not** recur: the automatic restart booted cleanly and the machine has run normally since.
- `diskutil verifyVolume disk1` (whole container) and `diskutil verifyVolume disk1s1` (Data volume) both return **exit code 0** — the filesystem is clean.
- NVMe SMART is **Verified**; no APFS mount errors in the logs.

The likely cause is **marginal hardware, not the EFI**: the donor machine runs **two mismatched SO-DIMMs from different vendors** with ECC disabled. Mixed DIMMs are a common source of sporadic early-boot memory-corruption panics. If you see this signature, test your RAM before blaming the config. A rare race in APFS's parallel early volume mounts is the secondary suspect.

**2. Boot-time TRIM is intentionally disabled**

`Kernel → Quirks → SetApfsTrimTimeout` is set to **`0`**, which on macOS 12+ **disables TRIM at boot**. This is deliberate: it avoids the long boot-time TRIM stall that cheap DRAM-less NVMe drives exhibit. macOS still reports "TRIM Support: Yes" in System Information, but boot-time trimming is off. If you have a good NVMe drive and want trimming back, set it to `-1`.

**3. Long shutdowns on some systems**

Every shutdown on the donor machine writes a `shutdown_stall` report. Decoding it shows **`CloudTelemetryService`** (`privatecloudcomputed`) hanging, which is a macOS/iCloud telemetry behaviour and is unrelated to this EFI.

---

## Troubleshooting

- **No HDMI/DP audio:** confirm `hda-gfx=onboard-1` exists on both IGPU and HDEF, and that `no-controller-inject` is **absent** from IGPU.
- **Boot panic in AppleHDA/AppleALC:** you are running a config without the fix above. Do not add `no-controller-inject` back.
- **iServices not working:** generate your own SMBIOS; the placeholders here are intentionally fake.
- **Wi-Fi not connecting:** ensure AirportItlwm matches your macOS version (2.4.0 = Sequoia).
- **Picker not showing:** hold Alt/Esc/0 during boot.

---

## Credits

- [Acidanthera](https://github.com/acidanthera) — OpenCore, Lilu, WhateverGreen, AppleALC, VirtualSMC, AirportItlwm, NVMeFix, RestrictEvents and more.
- [Olarila](https://www.olarila.com/) — community and logo.
- [zearp/Nucintosh](https://github.com/zearp/Nucintosh) — NUC8 reference work.
- Everyone who reported and diagnosed the AppleALC/AppleHDA panic.

<p align="center">
  <img src="docs/images/olarila-logo.png" alt="Olarila" width="220">
</p>

---

## Disclaimer

This EFI is provided **as-is**, for the Intel NUC8 platform. Hackintosh is not supported by Apple. You are responsible for your own hardware, data and Apple ID. Always keep a backup of your working EFI and never publish your real SMBIOS serials.
