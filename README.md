# Lenovo ThinkBook 16p Gen 4 — internal speakers on macOS Tahoe (OpenCore)

Working internal speakers on a **Lenovo ThinkBook 16p Gen 4** (Intel 13th gen, RTX 4060)
running **macOS Tahoe 26.7.3** with OpenCore. The speakers are driven by two
**Cirrus Logic CS35L41** smart amplifiers on I²C, fed by a **Realtek ALC287** codec.
Without this, macOS shows only the headphone jack (or "Internal Speakers" that make no sound).

> ⚠️ Experimental, community-made, no warranty.

## Does it fit my laptop?

**Only if your audio subsystem ID is exactly `17AA:38AB`.** The amplifier driver is
deliberately locked to that ID, to bus `I2C7` and to amp addresses `0x40`/`0x41`,
because the amplifier firmware/tuning is board-specific — loading the wrong tuning on
a speaker amp can damage speakers.

| Your machine | Status |
|---|---|
| ThinkBook 16p Gen 4, subsystem `17AA:38AB` | ✅ This repo (tested) |
| ThinkBook 16p Gen 4, subsystem `17AA:38A9` | ❌ Not supported as-is (different tuning files, needs its own port) |
| ThinkBook 16p Gen 5 (`17AA:38F9`/`38FA`) | Use upstream [CirrusAudioFixup](https://github.com/hoaug-tran/CirrusAudioFixup) — already listed there |
| Other ThinkBooks / other brands | ❌ Not supported. Check the upstream CirrusAudioFixup support list |

The AppleALC part (layout **66**) on its own is generic for **ALC287 machines whose
internal speaker is on pin `0x17`**, but on its own it does not power the CS35L41 amps.

**How to check your subsystem ID**
- Windows: Device Manager → *Sound, video and game controllers* → *Realtek* → Details →
  *Hardware Ids* → look for `SUBSYS_38AB17AA`.
- macOS (needs any HDA driver loaded): `ioreg -l | grep IOHDACodecFunctionSubsystemID` → `397031595` (= `0x17AA38AB`).
  Note the PCI `subsystem-id` of `HDEF` is different (`384D` on this unit) — don't use that one.
- Linux: `cat /proc/asound/card*/codec#0 | grep Subsystem` → `0x17aa38ab`.

## What's included

| Path | What it is |
|---|---|
| `EFI/OC/Kexts/AppleALC.kext` | AppleALC 1.9.8 built with a new **ALC287 layout 66** (see below) |
| `EFI/OC/Kexts/ThinkBookCirrusInit.kext` | CirrusAudioFixup 1.2.1 patched for `17AA:38AB` (bundle id stays `com.hoaugtr.CirrusAudioFixup`) |
| `EFI/OC/Kexts/ThinkBookI2C7Match.kext` | Codeless kext: makes VoodooI2C attach to the I2C7 controller `8086:51D9` where the amps live |
| `config-snippet.plist` | The exact `Kernel → Add`, `Kernel → Block` and `DeviceProperties` entries to merge into your config.plist |
| `patches/` | Source diffs so you can audit/rebuild everything |
| `licenses/` | Licenses of the redistributed components |
| `SHA256SUMS.txt` | Hashes of the kext binaries |

**Not included — download it yourself:**
[SimpleHDA for Tahoe 1.0.2](https://github.com/laobamac/SimpleHDA) (`SimpleHDA.kext`).
macOS Tahoe removed AppleHDA; SimpleHDA brings it back. It contains Apple components and
is not redistributable here.

**Required kexts you probably already have:** Lilu 1.7.2+, VoodooI2C 2.9.1 (+ VoodooI2CHID, VoodooGPIO, VoodooI2CServices).

## Install

1. Copy the three kexts from `EFI/OC/Kexts/` plus `SimpleHDA.kext` into your `EFI/OC/Kexts/`.
2. Merge `config-snippet.plist` into your `config.plist` (ProperTree / OCAT). **Order matters** in `Kernel → Add`:
   - `Lilu.kext` → `AppleALC.kext` → `SimpleHDA.kext` → … other Lilu plugins …
   - `VoodooI2CServices` → `VoodooGPIO` → `VoodooI2C.kext` → **`ThinkBookI2C7Match.kext`** → `VoodooInput` → `VoodooI2CHID`
   - **`ThinkBookCirrusInit.kext`** anywhere after the VoodooI2C entries (last is fine).
3. Set `DeviceProperties → Add → PciRoot(0x0)/Pci(0x1F,0x3) → layout-id` = `42000000` (data, = 66).
   Remove any `alcid=` boot-arg.
4. If you previously installed **VoodooHDA** in `/Library/Extensions`, either remove it or keep
   the `Kernel → Block` entry from the snippet that stops it loading.
5. Reboot, then *System Settings → Sound → Output → Internal Speakers*. Start at low volume.

Your speaker calibration is read at boot from the UEFI variable that Windows/Lenovo
firmware stores on your own laptop — nothing per-unit is shipped in this repo.
If you have never booted Windows on the machine, the driver may have no calibration.

## What was changed, and why

### 1. AppleALC — ALC287 layout 66
Stock layout 11 is the only ALC287 layout that uses speaker pin `0x17` (`0x17 → DAC 0x03`,
headphone `0x21 → DAC 0x02`), matching what Linux does for this board
(`ALC287_FIXUP_MG_RTKC_CSAMP_CS35L41_I2C_THINKPAD`). However, its internal-speaker path
carries a software DSP chain copied from a microphone profile — **noise reduction plus a
32-band EQ where every band is cut** (incl. a −3.9 dB shelf at 8.5 kHz). On this laptop
that made the speakers quiet and muffled.

Layout 66 = layout 11 with **only the `IntSpeaker → SignalProcessing` block removed**.
Headphone, mic and line-in are unchanged. Patch: `patches/AppleALC/`.

### 2. CirrusAudioFixup — 17AA:38AB support
Against upstream commit `c275ffe`:
- Embeds the 17AA:38AB CS35L41 firmware + left/right tuning from linux-firmware
  (byte-identical to the Windows driver's `17AA38AB_221222_V1_A0.bin` / `A1.bin`).
  The board reports **speaker ID 1**, which shares the same tuning files as ID 0.
- Locks board adoption to subsystem `17AA:38AB`.
- Talks to the amps through **stock VoodooI2C** (`transferI2C`) instead of a custom provider,
  restricted to the `SPKR` device on `I2C7` at `0x40`/`0x41`.
- Adds a matrix test case. Patch: `patches/CirrusAudioFixup/`.

## Tested on
- ThinkBook 16p Gen 4, i9-13900H, RTX 4060 Laptop, subsystem `17AA:38AB`
- macOS Tahoe 26.7.3 (25G342), OpenCore, SMBIOS MacBookPro16,2
- AppleALC 1.9.8 (custom), SimpleHDA 1.0.2, Lilu 1.7.3, VoodooI2C 2.9.1

## Known limitations
- **Quieter / less bass than Windows.** Windows adds Lenovo/Dolby processing on top; macOS
  has none. A system-wide EQ such as [eqMac](https://github.com/bitgapp/eqMac) helps
  (try +4–6 dB at 60–125 Hz). Linux users of this model report the same gap.
- **Internal microphone array** (Intel Smart Sound DMIC) does not work.
- Sleep/wake behaviour of the amps is not fully validated.
- The driver publishes many `Cirrus_*` diagnostic properties in IORegistry — handy for bug reports.

## Credits & licenses
- [CirrusAudioFixup](https://github.com/hoaug-tran/CirrusAudioFixup) by Tran Kinh Hoang (hoaug-tran) — non-commercial license, see `licenses/CirrusAudioFixup-LICENSE.txt`. **Do not sell or paywall this.**
- CS35L41 firmware © Cirrus Logic, redistributed unmodified under `licenses/Cirrus-Logic-firmware-LICENSE.txt`.
- [AppleALC](https://github.com/acidanthera/AppleALC) by vit9696 & acidanthera — BSD-3, see `licenses/AppleALC-LICENSE.txt`.
- [SimpleHDA](https://github.com/laobamac/SimpleHDA) by laobamac, [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C), [Lilu](https://github.com/acidanthera/Lilu).
- Linux `sound/hda` and linux-firmware for the board configuration and tuning data.
