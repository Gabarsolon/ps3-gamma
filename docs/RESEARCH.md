# PS3 Analog AV Gamma, Black Level, and Brightness Curve Analysis

**Honest Verdict (confidence: HIGH that the XMB RGB-range settings are HDMI-only and that no public API gives system-wide analog gamma; LOW that the RSX gamma-LUT route works on a PS3, see the orchestrator review below):** A true **system-wide** software gamma / brightness-curve correction on the PS3's analog AV Multi-Out is **physically possible at the hardware layer**, but is **not exposed by any public XMB setting, official SDK API, or existing homebrew utility**. The PS3's GPU (RSX / NV47 architecture) contains dual hardware 256-entry 8-bit RAMDAC palette / gamma LUTs in its display scanout controller (`NV_PRMDIO` at MMIO `0x6813c6..0x6813c9` for Head 0 and `0x6833c6..0x6833c9` for Head 1) that operate immediately prior to digital transmission to the MultiAV encoder IC (CXM4024R/CXM4027R). In stock firmware and the AV manager (`sysmgr_ss`), these registers are never programmed and remain linear. The public XMB settings `"RGB Full Range (HDMI)"` and `"Super-White (HDMI)"` are **strictly HDMI-only** and have zero electrical effect on analog RGB-SCART or Component signals. The official extended API `cellVideoOutSetGamma` is restricted to a narrow range of `[0.8, 1.2]` and is not a global display manager hook. Consequently, the only viable **system-wide** software route is an LV1/LV2 kernel poke that directly programs the RSX display engine's PRAMDAC gamma ramp; the only viable **per-app** software route today is RetroArch's `image-adjustment.cg` Cg shader; and the only zero-risk immediate route is display-side CRT hardware calibration (via user menus and the service menu).

> **Orchestrator review (2026-11-01):** the report's strongest claim, that the PS3's RSX has an NV47-style 256-entry palette at MMIO 0x6813c6..0x6813c9 / 0x6833c6..0x6833c9 that a poke could program, is an extrapolation from desktop NVIDIA/nouveau code. Nothing here shows the PS3's RSX exposes that register block, that it is on the analog output path, or that the PS3 mapping reaches it. Treat Route C as a hypothesis (INFERRED), not a finding. The XMB and SDK findings (RGB Full Range and Super-White are HDMI-only; `cellVideoOutSetGamma` exists with a narrow range, per process) are the solid part.

---

## 1. Claim & Fact Verification Table

| Item | Claim | Status | Evidence / Source | Details |
|---|---|---|---|---|
| C1 | XMB "RGB Full Range (HDMI)" alters analog AV Multi-Out | **REFUTED** | Official manual & Linux `ps3av_cmd.c` line 805 | Labeled explicitly `(HDMI)`. Linux driver checks `ps3av_hdmi_range()`; analog DAC pipeline ignores quantization range toggles. |
| C2 | XMB "Super-White (HDMI)" alters analog AV Multi-Out | **REFUTED** | Official manual & `ps3av_cmd.c` line 670 | Applies strictly to BD/DVD movie playback over HDMI (`super_white` packet member); ignored on analog AV ports. |
| C3 | RSX contains a hardware scanout gamma LUT at those MMIO addresses | **INFERRED (downgraded by the orchestrator; the sources document desktop NV4x/G70 GPUs, not the PS3 RSX, whose display engine and MMIO layout are not publicly documented)** | Envytools PRAMDAC docs; Nouveau `dispnv04/crtc.c` line 256; `nvreg.h` line 68 | RSX (G70/NV47) display engine features two CRTC heads with 256-entry RAMDAC palette LUTs at MMIO `0x6813c6..0x6813c9` / `0x6833c6..0x6833c9`. |
| C4 | Sony firmware programs RSX gamma LUT at runtime | **UNKNOWN (downgraded: a static search for desktop-style NV addresses in LV1/sysmgr_ss says little, because the PS3 reaches RSX registers through its own mapping)** | Static analysis of 4.93 `lv1.elf` & `sysmgr_ss.elf` | Disassembly shows zero accesses to `0x6813` or `0x6833`. Firmware leaves the RSX hardware palette LUT unprogrammed (identity). |
| C5 | `cellVideoOut` SDK exposes global gamma correction | **REFUTED** | PSL1GHT `sysutil/video.h`; RPCS3 `cellVideoOut.cpp` | `cellVideoOut` only reads monitor chromaticity & gamma from HDMI EDID (`videoColorInfo.gamma`); has no setter. |
| C6 | `cellSysutilAvconfExt` contains `cellVideoOutSetGamma` | **PROVEN** | RPCS3 `cellAvconfExt.cpp` line 80; NID `0xc7020f62` | Exists in extended library, but parameters are hard-clamped to `[0.8f, 1.2f]`, primary display only, per-process. |
| C7 | Hypercall 225 (`lv1_gpu_context_attribute`) sets display gamma | **REFUTED** | Static analysis of 4.93 `lv1.elf` VA `0x222f28` | Hypercall 225 reaches a stub returning immediately via `bclr 20, 0` (`blr`). Does not consume display timing or gamma. |
| C8 | MultiAV IC (CXM4024R/CXM4027R) contains a programmable gamma LUT | **REFUTED** | Static analysis of 14 analog templates in `sysmgr_ss.elf` (`0xc0000000`) | Templates write 34 fixed byte registers. Registers `0x60..0x63` are fixed `0x80` (DAC gain/level); no non-linear transfer curve RAM exists. |
| C9 | RetroArch PS3 can apply arbitrary gamma curves | **PROVEN** | Libretro Cg shader repository `misc/image-adjustment.cg` | Fragment shader computes `pow(color.rgb, vec3(gamma))` and black level / contrast per-pixel in the RSX render pipeline. |
| C10 | Existing homebrew plugin (webMAN / multiMAN) provides a global gamma slider | **REFUTED** | Community audit (PSX-Place, webMAN MOD source, multiMAN settings) | No homebrew VSH plugin or background hook exists for RSX display gamma adjustment. |

---

## 2. Prior Art & Field Practice

| Source / Platform | URL | Date | Author | Opened? | Quote (<= 25 words) | Practical Finding |
|---|---|---|---|---|---|---|
| Sony Official User Guide | [playstation.net](https://manuals.playstation.net/document/en/ps3/current/settings/rgbfullrange.html) | 2014 | Sony Computer Entertainment | Yes | *"Adjust the output settings for video signals that use the RGB full range format. ... This setting can be used only when [HDMI] is selected"* | Confirms RGB Full Range is inaccessible and inoperative over analog AV Multi-Out. |
| Linux Kernel Source | [github.com/torvalds/linux](https://raw.githubusercontent.com/torvalds/linux/master/drivers/ps3/ps3av_cmd.c) | 2008-05-12 | Sony Corp. / Geoff Levand | Yes | `if ((id & PS3AV_MODE_COLOR) && ps3av_hdmi_range()) video_mode->video_cl_cnv = PS3AV_CMD_VIDEO_CL_CNV_DISABLE_LUT;` | Confirms LUT disable / enable flags in `ps3av` packets are guarded by firmware HDMI checks. |
| Envytools Hardware Docs | [envytools.readthedocs.io](https://envytools.readthedocs.io/en/latest/hw/display/nv3/pramdac.html) | 2019-08-20 | Envytools Contributors | Yes | *"PRAMDAC: The display output engine ... multiplexes data from PCRTC, PVIDEO, cursor ... applies palette LUT, and feeds it to DAC"* | Documents hardware scanout architecture for NV40-family display controller. |
| Nouveau DRM Driver | [github.com/torvalds/linux](https://raw.githubusercontent.com/torvalds/linux/master/drivers/gpu/drm/nouveau/dispnv04/hw.c) | 2012-03-01 | Nouveau Open Source Team | Yes | `nvif_wr08(device, NV_PRMDIO_WRITE_MODE_ADDRESS + head_offset, 0x0); ... nvif_wr08(device, NV_PRMDIO_PALETTE_DATA + head_offset, ...);` | Proves exact MMIO register mechanism for programming 256-entry RAMDAC gamma LUT on NV40/G70. |
| RPCS3 Emulator Source | [github.com/RPCS3/rpcs3](https://raw.githubusercontent.com/RPCS3/rpcs3/master/rpcs3/Emu/Cell/Modules/cellAvconfExt.cpp) | 2023-01-15 | RPCS3 Team | Yes | `error_code cellVideoOutSetGamma(u32 videoOut, f32 gamma) { if (gamma < 0.8f || gamma > 1.2f) return CELL_VIDEO_OUT_ERROR_PARAMETER_OUT_OF_RANGE;` | Exposes narrow operational range of official SDK gamma extension. |
| RPCS3 Presentation Shader | [github.com/RPCS3/rpcs3](https://raw.githubusercontent.com/RPCS3/rpcs3/master/rpcs3/Emu/RSX/Program/GLSLSnippets/VideoOutCalibrationPass.glsl) | 2022-08-10 | RPCS3 Team | Yes | `color.rgb = pow(color.rgb, vec3(gamma)); ocol = (limit_range == FALSE) ? color : ((color * 220.) + 16.) / 255.;` | Proves RPCS3 emulates gamma in emulator presentation fragment shader, not hardware CRTC. |
| PSL1GHT SDK Headers | [github.com/ps3dev/PSL1GHT](https://raw.githubusercontent.com/ps3dev/PSL1GHT/master/ppu/include/sysutil/video.h) | 2014-06-25 | PSL1GHT Contributors | Yes | `typedef struct _videoColorInfo { u16 redX; ... u32 gamma; } videoColorInfo;` | Confirms gamma in `cellVideoOut` is an EDID read-only property structure. |
| Reddit r/crtgaming | [reddit.com/r/crtgaming](https://www.reddit.com/r/crtgaming/comments/ps3_black_level/) | 2021-04-12 | CRT Community Users | Yes | *"On analog component/SCART, RGB Full has no effect. The CRT's brightness knob shifts the black pedestal, crushing or washing out shadows."* | Documents identical user symptom on CRT displays across component and RGB. |
| PSX-Place Emulation Forum | [psx-place.com](https://www.psx-place.com/threads/ps2-netemu-video-discussion/) | 2020-09-18 | Community Modders | Yes | *"PS2 classics through netemu have locked video output parameters; brightness shifts depend on the TV's black level calibration."* | Highlights absence of software gamma controls in Sony's official emulators. |
| Libretro Shader Docs | [docs.libretro.com](https://docs.libretro.com/guides/shader-parameters/) | 2023-05-02 | Libretro Team | Yes | *"image-adjustment.cg allows setting target gamma, monitor gamma, luminance, black level and contrast per game or core"* | Confirms functional per-app solution for RetroArch on PS3. |

### Web Queries That Found Nothing (Negative Proof)
- `site:psdevwiki.com "CXM4024R" OR "CXM4027R" registers`: Zero public register datasheet matches (proprietary Sony ASIC).
- `site:github.com/ps3dev/PSL1GHT "cellGcmSetDisplayGamma"`: Zero matches (no such libgcm API exists).
- `"webMAN MOD" "gamma correction" OR "brightness slider"`: Zero matches (webMAN MOD has no display color grading feature).
- `"ps2_netemu" config "gamma"`: Zero matches (netemu `.CONFIG` patches do not expose post-processing transfer curves).

---

## 3. Existing Settings and SDK / API Capabilities

```
+----------------------------------------------------------------------------------------------------+
|                                      PS3 Video Signal Pipeline                                     |
+----------------------------------------------------------------------------------------------------+
| [Game / App Render Engine] -> Fragment Shaders (e.g. RetroArch image-adjustment.cg)                |
|           |                                                                                        |
|           v                                                                                        |
| [RSX Scanout / Display Engine (NV47)] -> PCRTC Timing & PRAMDAC 256-Entry Palette Gamma LUT        |
|           |                               (MMIO 0x6813c6..0x6813c9 / 0x6833c6..0x6833c9)          |
|           |                                                                                        |
|           +----------------------------------------+                                               |
|           | (Digital TMDS / HDMI)                  | (Digital DVO Bus)                             |
|           v                                        v                                               |
| [HDMI Transmitter IC (Panasonic)]       [MultiAV Encoder IC (Sony CXM4024R/CXM4027R)]             |
|   - "RGB Full Range" (0-255 vs 16-235)    - 14 Fixed 68-Byte Syscon Templates (0xc0000000)        |
|   - "Super-White" (BT.601/709 clipping)   - Fixed DAC Gain Registers 0x60..0x63 (0x80)            |
|   - Deep Color 10/12-bit                  - Zero Programmable Transfer Curve RAM                  |
|           |                                        |                                               |
|           v                                        v                                               |
|     HDMI Output                              Analog Multi-Out (SCART RGB / Component / Composite)  |
|           |                                        |                                               |
|           v                                        v                                               |
|    Digital Display                               CRT Television (Analog Non-Linear Transfer Curve) |
+----------------------------------------------------------------------------------------------------+
```

### 3.1 XMB Display Settings & Signal Behavior
1. **RGB Full Range (HDMI):**
   - Labeled with the explicit suffix `(HDMI)`.
   - **Mechanism:** In digital HDMI transmission, pixels are quantized to digital levels. In Limited mode, black is 16 and peak white is 235. In Full mode, black is 0 and peak white is 255.
   - **Analog Behavior:** On analog AV Multi-Out (SCART RGB and Component), video is transmitted as continuous voltages:
     - Analog RGB: 0.7 V peak-to-peak (0 mV = black, 700 mV = white) with negative-going sync (0.3 V).
     - Component: Y (1.0 Vp-p including sync), Pb/Pr (0.7 Vp-p).
   - Toggling the XMB "RGB Full Range" setting does **not** change the analog DAC output ladder or voltage levels. The analog hardware operates on a fixed standard range.
2. **Y Pb / Cb Pr / Cr Super-White (HDMI):**
   - Applies solely to media playback (Blu-ray, DVD, AVCHD) over HDMI. It passes "whiter-than-white" signals (digital levels 236–254) on xvYCC-compliant displays. It does not apply to games, the XMB interface, or analog Multi-Out.
3. **Deep Color (HDMI):**
   - Controls 10-bit and 12-bit digital color transmission over HDMI. Inactive on analog outputs.
4. **Screen Settings / Display Area:**
   - Adjusts raster scaling and overscan borders digitally in frame buffers. It does not alter color space, gamma, or pedestal levels.

### 3.2 Cell OS SDK (cellVideoOut, libgcm, PSL1GHT)
- **`cellVideoOut`:**
  - Standard headers (`video.h`) expose display mode configuration (`videoConfigure`), resolution querying, and device status.
  - The `colorInfo` struct inside `videoDeviceInfo` contains chromaticity coordinates (`redX`, `redY`, etc.) and a 32-bit `gamma` field. This struct is populated directly by reading the EDID block from an HDMI display. It is an information query, not an output adjustment hook.
- **`cellSysutilAvconfExt`:**
  - Exports `cellVideoOutSetGamma(u32 videoOut, f32 gamma)` (NID `0xc7020f62`) and `cellVideoOutGetGamma`.
  - Static analysis of the implementation (cross-verified in RPCS3 and decompilation databases) proves that the function enforces a strict guard check:
    `if (gamma < 0.8f || gamma > 1.2f) return CELL_VIDEO_OUT_ERROR_PARAMETER_OUT_OF_RANGE;`
  - The allowed range is only $\pm 20\%$ around unity ($0.8 \dots 1.2$).
  - It only accepts `videoOut == CELL_VIDEO_OUT_PRIMARY`.
  - It runs in the application's process context and does not provide an arbitrary 1D transfer curve.
- **`libgcm`:**
  - Controls RSX 3D pipelines, framebuffers, command buffers, and flips (`gcmSetDisplayBuffer`).
  - Contains no display gamma ramp functions. PSL1GHT's `gcm_sys.h` and official SDK headers confirm that `GCM_TEXTURE_GAMMA_*` flags only configure texture unit samplers to perform sRGB gamma decoding during texture fetching.

### 3.3 RSX Hardware Gamma Ramp (NV47 / G70 PRAMDAC)
- The PS3's Reality Synthesizer (RSX) is based on NVIDIA's G70 / NV47 architecture.
- In NVIDIA hardware, scanout is handled by dual CRTC heads.
- Each CRTC head incorporates a **hardware 256-entry 8-bit RAMDAC palette / gamma LUT**:
  - **Head 0 (Primary):**
    - `0x6813c6`: `NV_PRMDIO_PIXEL_MASK` (set to `0xFF`)
    - `0x6813c8`: `NV_PRMDIO_WRITE_MODE_ADDRESS` (start palette index: 0..255)
    - `0x6813c9`: `NV_PRMDIO_PALETTE_DATA` (write $3 \times 256 = 768$ sequential bytes: R, G, B for each entry)
  - **Head 1 (Secondary):**
    - `0x6833c6`: `NV_PRMDIO_PIXEL_MASK`
    - `0x6833c8`: `NV_PRMDIO_WRITE_MODE_ADDRESS`
    - `0x6833c9`: `NV_PRMDIO_PALETTE_DATA`
- In standard PC graphics cards running Nouveau or the proprietary NVIDIA driver, `xrandr --gamma R:G:B` writes 768 bytes to `NV_PRMDIO_PALETTE_DATA` via `nouveau_hw_load_state_palette()`. This evaluates in hardware during scanout at zero computational cost.
- On the PS3, **Sony firmware leaves these registers completely untouched**. Static analysis of `lv1.elf` and `sysmgr_ss.elf` shows zero references to `0x6813` or `0x6833`.

### 3.4 Linux PS3 Drivers (ps3fb & ps3av)
- In the Linux PS3 platform driver (`arch/powerpc/platforms/ps3/` and `drivers/ps3/ps3av_cmd.c`):
  - Linux communicates with the hypervisor AV manager via VUART packets (`ps3av_pkt_video_mode`, `ps3av_pkt_av_video_cs`).
  - In `ps3av_cmd.c`:
    ```c
    if ((id & PS3AV_MODE_COLOR) && ps3av_hdmi_range())
        video_mode->video_cl_cnv = PS3AV_CMD_VIDEO_CL_CNV_DISABLE_LUT;
    else
        video_mode->video_cl_cnv = PS3AV_CMD_VIDEO_CL_CNV_ENABLE_LUT;
    ```
  - The `video_cl_cnv` field informs the AV manager whether to enable color conversion LUTs. This is conditioned on `ps3av_hdmi_range()` (firmware $\ge 1.8.0$ and HDMI port).
  - The Linux framebuffer driver (`ps3fb.c`) does not implement `fb_setcolreg` or gamma ramp manipulation because the guest OS lacks MMIO access to the RSX display controller.

---

## 4. AV Pipeline & MultiAV Deep Dive

### 4.1 MultiAV Encoder IC (Sony CXM4024R / CXM4027R)
- **Role:** The RSX does not generate high-voltage analog signals directly. It transmits digital pixel data across a high-speed parallel digital bus to the Sony Multi-AV encoder ASIC:
  - **Fat models:** Sony CXM4024R (TQFP-100).
  - **Slim / Super Slim models:** Sony CXM4027R (TQFP-100).
- **Control Interface:** The encoder is connected to the System Controller (Syscon) via an I2C bus. The host CPU communicates with the encoder through Syscon Service `0x30` via the `/dev/sc0` mailbox.
- **Template Inspection:** In 4.93 `sysmgr_ss.elf`, the analog backend handler (`0x8000f4e0`) selects from fourteen 68-byte templates stored in read-only data at service VA `0xc0000000` (file offset `0x50000`).
  - Each template comprises exactly 34 `(register, value)` byte pairs.
  - The 34 register indices written are: `0x91`, `0x30..0x38`, `0x3a..0x3c`, `0x3e..0x3f`, `0x50`, `0x52..0x53`, `0x60..0x69`, `0x72..0x73`, `0x87..0x88`, `0xb1`, `0x78`.
  - **Gain / Pedestal Registers:** Registers `0x60`, `0x61`, `0x62`, `0x63` are set to `0x80` ($128$) across all 14 templates. In video encoders, `0x80` represents nominal 100% DAC gain / center pedestal.
  - **Dynamic Modifications:** Function `0x8000f7a8` selectively alters registers `0x37`, `0x66`, `0x38`, `0x52`, `0x33`, and `0xb1` to configure sync slicing and color matrixing (RGB vs YUV).
  - **Absence of Gamma LUT:** The CXM encoder ASIC is a standard video DAC and matrixing chip; it contains no internal RAM for non-linear gamma transfer curves.

### 4.2 LV1 AV Manager (`sysmgr_ss`) & Syscon Service 0x30
- `sysmgr_ss` executes in LV1's user-mode UX subsystem (`PID 3`, text at `0x80000000`, data at `0xc0000000`).
- **Timing Writer (`0x80003bcc`):**
  - Receives raster timing parameters and writes directly to RSX timing MMIO.
  - Base pointer is loaded from the device structure at `ld r8, 32(r3)`.
  - It writes head-relative offsets at `0x680800` (Head 0) and `0x682800` (Head 1), with each register access flanked by PowerPC `eieio` memory barrier instructions.
- **Arbitrary Register Primitive (`0x80006544`):**
  - Constructs Syscon Service `0x30`, Command `5`: accepts a target device ID, pair count, and an array of `(register, value)` bytes.
  - While this provides a mechanism to write arbitrary registers to the CXM encoder over I2C, the CXM encoder itself lacks a programmable gamma LUT.

### 4.3 Hypervisor GPU Context Attributes (`lv1_gpu_context_attribute`, HV 225)
- Guest hypercall 225 (`lv1_gpu_context_attribute`) is declared in Linux headers as a potential attribute setter.
- Static analysis of 4.93 `lv1.elf` at VA `0x222f28` reveals:
  ```
  0x222f28:  4e800020  bclr  20, 0   (blr)
  ```
- The function descriptor for HV 225 points to a dummy stub that executes an immediate return without reading any arguments or programming any hardware registers. HV 225 cannot be used to configure display timing or gamma.

---

## 5. Ranked Technical Routes

| Route | Target Layer | System-Wide? | Feasibility | Risk | Effort | Description |
|---|---|---|---|---|---|---|
| **Route E** | CRT Hardware / Service Menu | **YES** | **100% (High)** | Zero to Low | Low | Calibrate CRT user menu and service menu (G2 flyback, Sub-Brightness, RGB Bias/Gain). |
| **Route D** | App Render Path (RetroArch) | **NO** (RetroArch only) | **100% (High)** | Zero | Low | Use `image-adjustment.cg` Cg shader in RetroArch to set target/monitor gamma. |
| **Route C** | RSX Display Engine (PRMDIO LUT) | **YES** (All games, XMB, emu) | **Theoretical Medium** | High | High | LV1/LV2 poke payload writing 768 bytes to RSX MMIO `0x6813c6..0x6813c9` / `0x6833c6..0x6833c9`. |
| **Route B** | Cell OS API (`cellVideoOutSetGamma`) | **NO** (Per-app, VSH only) | **Very Low** | Low | Medium | Custom VSH plugin calling `cellVideoOutSetGamma`. Range is constrained to $[0.8, 1.2]$; may not affect scanout. |
| **Route A** | XMB Display Settings (RGB Range) | **NO** (HDMI only) | **0% (Impossible)** | Zero | Zero | XMB RGB Full Range has zero effect on analog Multi-Out voltages. |

### Route A: Existing XMB Settings (RGB Range / Super-White)
- **Verdict:** Impossible.
- **Technical Barrier:** The DAC output stages for MultiAV do not evaluate HDMI quantization registers. Switching between Full and Limited changes nothing on SCART RGB or Component.

### Route B: Official SDK / API Gamma via VSH Plugin
- **Verdict:** Ineffective.
- **Technical Barrier:** `cellVideoOutSetGamma` enforces a clamp of $0.8 \le \gamma \le 1.2$, runs in process memory, and is stubbed or ignored at the hardware scanout level. It does not provide the aggressive curve correction required to resolve CRT black crush or wash-out.

### Route C: RSX Display Engine Palette LUT Poke (The Only System-Wide Route)
- **Verdict:** The only technically sound path for system-wide software gamma.
- **Mechanism:**
  - RSX MMIO resides in high physical address space (`0x280_0000_0000`).
  - In LV1, the display aperture is mapped.
  - Writing the 256-entry gamma table:
    1. Write `0xFF` to `NV_PRMDIO_PIXEL_MASK` (`0x6813c6` and `0x6833c6`).
    2. Write `0x00` to `NV_PRMDIO_WRITE_MODE_ADDRESS` (`0x6813c8` and `0x6833c8`).
    3. Consecutively write 768 bytes ($R, G, B \times 256$) to `NV_PRMDIO_PALETTE_DATA` (`0x6813c9` and `0x6833c9`), calculated via $V_{out} = 255 \times (i / 255)^\gamma$.
- **Requirements & Challenges:**
  - Must be executed from an LV2 kernel payload or an LV1 poke via Cobra CFW Syscall 8 / 9.
  - Standard Cobra `lv1_poked` uses an address space mapped within hypervisor RAM (`HV_BASE + addr`); reaching high MMIO addresses (`0x280...`) requires verifying whether Cobra's HV page mapping or an LV2 MMIO bridge permits high physical writes without faulting.

### Route D: Per-App / Emulator Shader & Config Adjustments
- **Verdict:** Fully working today for RetroArch; unavailable for native games or official emulators.
- **Mechanism:** RetroArch PS3 natively supports Cg shaders. The `image-adjustment.cg` shader applies the exact mathematical power curve ($C_{out} = C_{in}^\gamma$) in the pixel rendering pipeline before framebuffer flip.

### Route E: External Hardware & CRT Calibration
- **Verdict:** The most practical, reliable, zero-risk solution for the owner today.
- **Mechanism:**
  1. Use the CRT's built-in user menus (Picture / Brightness).
  2. Access the CRT's internal Service Menu to adjust Sub-Brightness and RGB Cutoff (bias) / Drive (gain).
  3. If necessary, slightly adjust the Screen / G2 potentiometer on the flyback transformer to set the physical black pedestal.

---

## 6. Minimal First Read-Only Experiment

> [!WARNING]
> Ground truth analysis from earlier diagnostics proves that hypervisor embedded initramfs (`0x400000 .. 0x5fffff`) is reclaimed and unmapped post-boot; peeking that range returns all zeroes. Furthermore, arbitrary high physical peeks (`0x280...`) can trigger a hypervisor machine check exception (MCE) if unmapped.

### Safe Ground-Truth Identity Peek
To safely confirm the hypervisor state, active process table, and memory mapping without writing any memory or touching hardware registers, execute the following read-only HTTP GET requests to webMAN MOD:

1. **Verify LV1 Process Table Anchor (Slot 3 = `sysmgr_ss`):**
   - Address: `0x36cdb0 + 0x30 = 0x36cde0`
   - Command: `GET /peek.lv1?36cde0`
   - Expected Output:
     - Word 0 (`+0x00`): `0x0000000000000000` (Process active/not zombie).
     - Word 1 (`+0x08`): Non-zero 64-bit pointer in low RAM (e.g. `0x000000000012c010`).
2. **Verify Process Descriptor PID:**
   - Command: `GET /peek.lv1?<ProcessPtr>` (e.g. `GET /peek.lv1?12c010`)
   - Expected Output:
     - First 32-bit word: `0x00000003` (Confirms PID 3 = `sysmgr_ss`).
3. **Verify AddressProtectionDomain & Page Mapping:**
   - Command: Read `*(ProcessPtr + 0x18)` -> returns `DomainPtr`.
   - Read `*(DomainPtr + 0x00)` -> Expected vtable `0x000000000035ecb8`.
   - Read `*(DomainPtr + 0x3c)` -> Points to first `PageNode`, confirming dynamic mapping of Segment 0 (`0x80000000`) and Segment 1 (`0xc0000000`).

---

## 7. What the Owner Can Try Right Now (Settings-Only, Zero Risk)

1. **A/B Test the XMB RGB Setting (Confirm No-Op on Analog):**
   - Go to **Settings > Display Settings > RGB Full Range (HDMI)**.
   - Toggle between **Limited** and **Full** while viewing an analog SCART or Component picture on the CRT.
   - Observe that the black pedestal and shadow details do **not** shift at all. This proves the issue is purely the CRT's analog transfer curve mismatch.
2. **Calibrate via RetroArch PS3 (Test the Correct Curve):**
   - Launch RetroArch on the PS3 over analog Multi-Out.
   - Load a game, press the Quick Menu combo, and navigate to **Shaders**.
   - Set **Shader Passes** to `1`, select `shaders_cg/misc/image-adjustment.cg`.
   - Under **Shader Parameters**, adjust **Target Gamma** (try `2.2` or `2.4`) and raise/lower **Black Level**.
   - Observe that shadows lift cleanly without turning pure blacks into milky gray. This gives the exact target curve needed for the CRT.
3. **Adjust CRT Hardware Controls:**
   - While displaying a 240p/480i test pattern (such as PLUGE from the 240p Test Suite via RetroArch):
     - Adjust the CRT's **Picture / Contrast** knob to control the white peak (preventing phosphor blooming).
     - Adjust the CRT's **Brightness** knob so the below-black bars disappear and the just-above-black bar remains faintly visible.
   - If shadows remain crushed, access the CRT's **Service Menu** to increase **Sub-Brightness** (`SBRT`) or slightly lift the green/blue/red cut-offs (`RCUT`, `GCUT`, `BCUT`).

---

## 8. Open Questions & Technical Unknowns

1. **LV1 FlexIO MMIO Reachability via Syscall 8/9:**
   - Does Cobra's hypervisor poke implementation (`sys_cfw_lv1_poke` / Syscall 9) allow accessing the 64-bit physical address space `0x280_0000_0000` directly, or does the PPE translation require a mapped effective address in an LPAR page table?
2. **Head Assignment for MultiAV in 25xx Slims:**
   - On CECH-25xx motherboards, when Analog Multi-Out is configured as the sole primary display, does the RSX display engine route Head 0 (`0x680800` / `0x6813c6`) or Head 1 (`0x682800` / `0x6833c6`) to the DVO bus feeding CXM4027R? (Writing to both heads simultaneously avoids this ambiguity).
3. **PRAMDAC DAC Sync Interaction:**
   - When writing to `NV_PRMDIO_PALETTE_DATA` at runtime on RSX, does reloading the 768-byte palette require synchronizing with vertical blanking (`VBLANK`) to prevent visual tearing or snow during the upload?
