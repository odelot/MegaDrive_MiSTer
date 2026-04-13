# MegaDrive_MiSTer — RetroAchievements Fork

This is a fork of the official [MegaDrive/Genesis core for MiSTer](https://github.com/MiSTer-devel/MegaDrive_MiSTer) with modifications to support **RetroAchievements** on MiSTer FPGA.

> **Status:** Experimental / Proof of Concept — works together with the [modified Main_MiSTer binary](https://github.com/odelot/Main_MiSTer).

## What's Different from the Original

The upstream MegaDrive core is a Nuked-MD based FPGA Genesis/Mega Drive implementation. This fork adds two new modules and minor wiring changes so the ARM side (Main_MiSTer) can read emulated Genesis RAM for achievement evaluation. **No emulation logic was changed** — the core plays games identically to the original.

### Added Files

| File | Purpose |
|------|--------|
| `rtl/ra_ram_mirror_md.sv` | Reads requested 68K Work RAM addresses from BRAM and writes cached values to DDRAM for the ARM CPU |
| `rtl/ddram_arb_md.sv` | DDRAM arbiter that gives the MegaDrive audio DSP priority while allowing the RA module to use idle cycles |

### Modified Files

| File | Change |
|------|--------|
| `MegaDrive.sv` | Instantiates both new modules, adds a second BRAM read port for RA, wires DDRAM read/write channels |
| `files.qip` | Adds both new `.sv` files to the Quartus project |

### How the RAM Mirror Works

The Genesis has 64 KB of 68K Work RAM. The core uses a **selective address protocol (Option C)**, the same approach used for SNES and PSX:

1. The **ARM binary** writes a list of RAM addresses it needs to evaluate to DDRAM offset `0x40000` (up to 4096 addresses per frame).
2. On each **VBlank**, the FPGA module reads those addresses from the 68K Work RAM BRAM (via a dedicated read port) and accumulates the byte values.
3. The values are written in 8-byte chunks to DDRAM offset `0x48000`, and a response counter is updated so the ARM knows the data is ready.
4. The ARM binary reads the values and feeds them to the rcheevos achievement engine.

The `ddram_arb_md.sv` arbiter ensures the MegaDrive audio DSP always has priority on the DDRAM bus — the RA module only accesses DDRAM during idle cycles.

### Genesis-Specific: FC1004 Address Bit-13 Inversion

The original Genesis hardware (FC1004 / YM6045 chip) inverts bit 13 when addressing 68K Work RAM. The `ra_ram_mirror_md.sv` module replicates this quirk so BRAM lookups return the correct bytes:

```
BRAM address = { byte_addr[15], ~byte_addr[14], byte_addr[13:1] }
```

**Memory region exposed:**

| Region | Address Range | Size | Description |
|--------|-------------|------|-------------|
| 68K Work RAM | $000000–$00FFFF | 64 KB | All game variables, stack, heap |

### DDRAM Layout

```
0x00000   Header:   magic ("RACH") + flags + frame counter + debug info
0x40000   AddrReq:  ARM → FPGA address request list (count + request_id + addresses)
0x48000   ValResp:  FPGA → ARM value response cache (response_id + response_frame + values)
```

All data flows through shared DDRAM at ARM physical address **0x3D000000**.

### Architecture Diagram

```
┌───────────────────────────────────────┐
│       MegaDrive FPGA Core             │
│                                       │
│  68K Work RAM (64KB) in BRAM          │
│  accessed via dedicated read port     │
└─────────────┬─────────────────────────┘
              │  VBlank
              ▼
┌───────────────────────────────────────┐
│     ra_ram_mirror_md.sv               │
│  Reads requested addrs from BRAM      │
│  Applies FC1004 bit-13 inversion      │
│  Writes header + values to DDRAM      │
│                                       │
│     ddram_arb_md.sv                   │
│  Arbitrates DDRAM: audio DSP first,   │
│  RA module on idle cycles             │
└─────────────┬─────────────────────────┘
              │  DDRAM @ 0x3D000000
              ▼
┌───────────────────────────────────────┐
│     Main_MiSTer (ARM binary)          │
│  mmap /dev/mem → reads mirror         │
│  Writes address list → reads values   │
│  rcheevos evaluates achievements      │
└───────────────────────────────────────┘
```

## How to Try It

1. Download the latest MegaDrive core binary (`MegaDrive_*.rbf`) from the [Releases](https://github.com/odelot/MegaDrive_MiSTer/releases) page.
2. Copy the `.rbf` file to `/media/fat/_Console/` on your MiSTer SD card (replacing or alongside the stock MegaDrive core).
3. You will also need the **modified Main_MiSTer binary** from [odelot/Main_MiSTer](https://github.com/odelot/Main_MiSTer) — follow the setup instructions there to configure your RetroAchievements credentials.
4. Reboot your MiSTer, load the MegaDrive core, and open a game that has achievements on [retroachievements.org](https://retroachievements.org/).

## Building from Source

Open the project in Quartus Prime (use the same version as the upstream MiSTer MegaDrive core) and compile. Both `ra_ram_mirror_md.sv` and `ddram_arb_md.sv` are already included in `files.qip`.

## Links

- Original MegaDrive core: [MiSTer-devel/MegaDrive_MiSTer](https://github.com/MiSTer-devel/MegaDrive_MiSTer)
- Modified Main binary (required): [odelot/Main_MiSTer](https://github.com/odelot/Main_MiSTer)
- RetroAchievements: [retroachievements.org](https://retroachievements.org/)

---

# Original MegaDrive Core Documentation

*Everything below is from the upstream [MegaDrive_MiSTer](https://github.com/MiSTer-devel/MegaDrive_MiSTer) README and applies unchanged to this fork.*

## Nuked-MD port for MiSTer

![nukedmd_logo](rtl/nuked-md/nukedmd_logo.png)

[Original Nuked-MD repository](https://github.com/nukeykt/Nuked-MD-FPGA)

## Installing
copy rbf to root of SD card. Put some ROMs (.BIN/.GEN/.MD/.SMS) into MegaDrive folder


## Hot Keys
* F1 - reset to JP(NTSC) region
* F2 - reset to US(NTSC) region
* F3 - reset to EU(PAL)  region


## Auto Region option (Megadrive/Genesis carts only)
There are 2 versions of region detection:

1) File name extension:

* BIN -> JP
* GEN -> US
* MD  -> EU

2) Header. It may not always work as not all ROMs follow the rule, especially in European region.
The header may include several regions - the correct one will be selected depending on priority option.


## Sega Master System

Core supports SMS carts with the same compatibility level as original MegaDrive hardware. Not all SMS carts are compatible with MD hardware.


## Additional features

* Multitaps: 4-way, Team player, J-Cart
* SVP chip (Virtua Racing)
* Audio Filters for Model 1, Model 2, Minimal, No Filter.
* Option to choose between YM2612 and YM3438 (changes Ladder Effect behavior).
* FM chip for SMS carts.
* Composite Blending, smooth dithering patterns in games.
* Border/Borderless modes.
* Support many popular mappers.
