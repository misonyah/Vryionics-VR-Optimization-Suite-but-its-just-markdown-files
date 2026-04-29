# GPU Optimizations for VR

## Hardware Accelerated GPU Scheduling (HAGS)

**What:** Moves GPU memory scheduling from the CPU driver to the GPU hardware. Reduces frame time variance by 0.5–2ms.  
**Supported:** NVIDIA GTX 10xx+, AMD RX 400+, Intel Arc/Iris Xe. All GPUs with WDDM 2.7+ drivers.

**Enable:** Settings → System → Display → Graphics → Hardware-accelerated GPU scheduling → On.

**Registry equivalent:**
```
HKLM\SYSTEM\CurrentControlSet\Control\GraphicsDrivers
  HwSchMode = 2  (DWORD)   ; 1 = disabled, 2 = enabled
```

Reboot required.

---

## Resizable BAR (NVIDIA) / Smart Access Memory (AMD)

Allows the CPU to access the full GPU VRAM instead of 256MB windows. Improves texture streaming by 5–15% in GPU-limited scenarios.

### Prerequisites (both vendors)
1. BIOS → enable **Above 4G Decoding** (required first)
2. BIOS → enable **Resizable BAR** (NVIDIA) or **Smart Access Memory** / **SAM** (AMD)

### NVIDIA — after BIOS change
Open NVIDIA Control Panel → Help → Debug → Enable Resizable BAR Support.

### AMD — after BIOS change
Radeon Software → Performance → Tuning → confirm AMD Smart Access Memory shows **Enabled**.
AMD SAM requires Ryzen 4000+ or Intel 11th gen+.

---

## GPU MSI Interrupts + Interrupt Priority

Configures the GPU to use Message Signaled Interrupts (lower latency than legacy IRQ) and process those interrupts at High priority. Typically reduces frame latency by 1–2ms.

**Find your GPU's PNP Device ID:**
```powershell
Get-PnpDevice | Where-Object { $_.FriendlyName -like "*NVIDIA*" -or $_.FriendlyName -like "*Radeon*" } |
  Select-Object FriendlyName, InstanceId
```

The InstanceId is your PNP path (e.g. `PCI\VEN_10DE&DEV_2684&...`).

**Registry (replace `{PNPDeviceID}` with the actual path):**
```
HKLM\SYSTEM\CurrentControlSet\Enum\{PNPDeviceID}\Device Parameters\Interrupt Management\Affinity Policy
  DevicePriority = 3  (DWORD)   ; 3 = High

HKLM\SYSTEM\CurrentControlSet\Enum\{PNPDeviceID}\Device Parameters\Interrupt Management\MessageSignaledInterruptProperties
  MSISupported = 1  (DWORD)
```

Reboot required. Verify in Device Manager → GPU → Properties → Resources — should show "Message-Based Interrupt".

---

## GPU Drivers

Keep GPU drivers updated. Updates frequently include:
- VR compositor optimizations
- NVENC/AMF/QSV encoder improvements (critical for wireless VR)
- OpenXR/OpenVR bug fixes

**NVIDIA:** GeForce Experience → Drivers, or [nvidia.com/drivers](https://www.nvidia.com/en-us/drivers/)  
**AMD:** Radeon Software → Updates  
**Intel Arc:** Intel Arc Control → Software Updates

Older than 6 months is a concern; older than 12 months means you're missing significant VR improvements.

---

## PCIe Slot

Your GPU must be in the primary PCIe x16 slot. A x4 or x8 secondary slot limits bandwidth.

Check: Device Manager → Display Adapters → GPU → Properties → Details → Bus number.  
Or use GPU-Z — the Bus Interface field shows current link width (e.g. "PCIe x16 4.0 @ x16 4.0").

Target: Gen 3 x16 minimum. Gen 4 x16 is ideal.

---

## Thermal Throttling

Modern GPUs start throttling at 83–90°C (varies by model). In VR this manifests as sudden frame drops.

**Symptoms:** GPU clock drops mid-session, GPU-Z shows "GPU temperature limit" throttle reason.

**Fixes:**
- Repaste the GPU heatsink (often dried out after 3+ years)
- Ensure GPU fans are spinning (not silent at idle)
- Positive case pressure: more intake than exhaust fans
- Mild GPU undervolt reduces heat at equal or better performance (MSI Afterburner → Ctrl+F voltage curve)

---

## VRAM

When VRAM fills, the GPU pages textures to system RAM via PCIe — 10–50× slower. In VR: severe frame spikes.

**Warning threshold:** >88% VRAM usage.

**Fixes:**
- Lower SteamVR render resolution
- Reduce texture quality in-game
- Disable MSAA (use TAA instead — less VRAM, better image quality in VR)
- Clear shader caches (NVIDIA Control Panel → Help → Debug → Flush Shader Cache)

---

## GPU Encoder (wireless VR)

For wireless VR (Air Link, Virtual Desktop, ALVR), the GPU's hardware encoder is used to compress video frames. At >90% encoder utilization, encode latency spikes cause visible artifacts.

**Fixes:**
- Reduce streaming bitrate by 20–30%
- Lower render resolution
- Use H.265/HEVC over H.264 (better quality/bitrate ratio, lighter on encoder)
- If using Intel Arc: switch to AV1 in Virtual Desktop settings for ~30% better compression

---

## Integrated GPU (laptops)

On laptops, VR apps may silently route to the integrated GPU. Verify and fix:

Settings → System → Display → Graphics → add each VR exe → Options → **High performance**.

Critical executables:
- `vrserver.exe` (SteamVR)
- `vrcompositor.exe` (SteamVR)
- `OVRServer_x64.exe` (Meta/Oculus)
- `VirtualDesktop.Streamer.exe` (if using Virtual Desktop)
- Your VR game exe

**NVIDIA Optimus:** Also set in NVIDIA Control Panel → Manage 3D Settings → Program Settings → "High-performance NVIDIA processor" for each.

Verify: launch SteamVR → status window top-right should show your discrete GPU name.

---

## HDR overhead

Windows HDR enables a desktop compositor tone-mapping pass that adds ~1–3ms GPU time per frame and can cause color-space issues in VR previews. Disable HDR on your monitor(s) while doing VR unless your headset supports native HDR.

Settings → System → Display → HDR → Off.
