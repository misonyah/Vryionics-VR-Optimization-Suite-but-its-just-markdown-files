# Hardware Checklist for VR

## RAM

### Minimum: 16GB

| RAM | VR result |
|---|---|
| <8GB | Critical — pagefile usage is constant; 100–500ms stutter spikes |
| 8–15GB | Warning — VRChat + SteamVR + OS fills this; pagefile under load |
| 16GB | Adequate |
| 32GB+ | Comfortable for heavy VR sessions with streaming/Discord running |

VRChat alone can consume 4–8GB. Add SteamVR (~1GB), vrcompositor, Discord, and OS overhead and you're easily at 12–14GB before the game loads anything.

### Enable XMP / EXPO

If your RAM runs at JEDEC default (typically 2133MHz or 2400MHz) but is rated for 3200MHz, 4800MHz, etc., you're leaving performance on the table.

**Enable in BIOS:**
- Intel: Advanced → Memory → XMP → Profile 1 (or simply "Enabled")
- AMD: Advanced → Memory → EXPO or DOCP → Profile 1

**Why it matters:** Higher RAM speed improves integrated GPU performance, AMD Infinity Fabric clock synchronization, and bandwidth-heavy operations.

A 200MHz+ gap between XMP speed and current configured speed is worth fixing.

### Dual-channel

Single-channel RAM halves memory bandwidth. Dual-channel doubles the memory bus width from 64-bit to 128-bit.

**Effect:**
- Integrated GPUs: critical — shares system RAM bandwidth
- Discrete GPUs: meaningful for CPU-side physics (VRChat PhysBones, Unity job scheduler) and CPU↔GPU transfers

**How to confirm dual-channel:** CPU-Z → Memory tab → Channels → should show "Dual". Or HWiNFO64 → Memory.

**Fix:** Check your motherboard manual. On ATX boards, dual-channel typically requires slots A2+B2 (slots 2 and 4 counting from the CPU), not A1+B1 (1 and 3).

### Nonpaged Pool

Nonpaged pool stores kernel-mode memory that cannot be swapped. If >600MB, you likely have virtualization software (Hyper-V virtual switch, VirtualBox drivers, WSL2 networking) or aggressive antivirus eating kernel memory.

Check with Sysinternals RAMMap or poolmon.exe.

---

## Storage

### VR must be on SSD

| Drive type | Avatar load time |
|---|---|
| HDD | 10–60 seconds per avatar |
| SATA SSD | 2–8 seconds |
| NVMe SSD | 1–3 seconds |

VRChat avatar bundle loading is the most I/O-intensive VR workload. HDD seek time (5–12ms) vs NVMe (<0.02ms) is the difference between smooth world joins and multi-second freezes.

To move Steam library: Steam → Settings → Storage → Add drive → move games.

### Free Space

Keep at least 10–15% free on the VR drive. NTFS performance degrades below 10% free due to fragmentation and pagefile operations.

Quick wins for space:
- Clear VRChat cache: VRChat Settings → Performance → Cache Management → Clear Caches (re-downloads on next visit)
- Clear shader cache: NVIDIA Control Panel → Help → Debug → Flush Shader Cache; or delete `%LOCALAPPDATA%\NVIDIA\DXCache`
- Steam: right-click game → Properties → Local Files → Browse → look for large log files

### NVMe Power Saving

NVMe APST (Autonomous Power State Transition) allows the drive to enter low-power sleep states. Wake latency ranges from 10ms (PS3) to 100ms+ (PS4). During VR sessions with burst storage access, this causes the "random single-frame stutter" that VRChat users know well.

**Disable StorPort idle PM (PowerShell, run as Admin):**
```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\StorPort" -Name "IdlePowerState" -Value 0 -Type DWord
```

Or use the NVMe Idle Power Management setting in Power Options (if visible in your system's advanced power settings).

Reboot to apply.

---

## USB

### USB generation for tethered headsets

Meta Quest Link, Pico Connect, and other USB-tethered headsets require **USB 3.0 (5 Gbps) or better**. USB 2.0 (480 Mbps) cannot sustain the video stream.

Visual indicator: USB 3.x ports are blue (or have a "SS" label on the port). USB 2.0 ports are black.

For highest reliability: use a port on your motherboard's **native USB controller** (Intel or AMD chipset). Avoid:
- USB hubs
- Generic add-in cards
- ASMedia controllers (see below)

### ASMedia USB controllers

ASMedia ASM1042/1142/1143/2142 controllers are commonly deployed on AM4/AM5 consumer motherboards as secondary USB. They have documented intermittent audio dropouts and USB-reset events during sustained high-bandwidth Meta Quest Link sessions (400–500 Mbps).

**How to identify:** Device Manager → View → Devices by connection → expand Universal Serial Bus controllers.

**Fix priority:**
1. Move headset cable to an Intel or AMD native USB port (check your motherboard's I/O panel diagram — the CPU/chipset-direct ports are usually labeled)
2. Install latest ASMedia driver from your motherboard vendor website (not Windows Update's generic driver)
3. Add a PCIe x1 USB 3.0 card with Intel, Renesas, or NEC controller ($20–40)

### Generic USB controllers

Generic/unknown USB host controllers (cheap PCIe cards with unlabeled chips) have inconsistent interrupt timing, causing tracking micro-stutters even when the headset physically stays connected.

If you added a USB card for your headset, verify it uses a quality controller. Renesas μPD720201/202, Intel xHCI, and NEC controllers are reliable.

---

## Minimum VR specs

For reference:
- **CPU:** 6+ physical cores, 3.5GHz+ single-core
- **RAM:** 16GB dual-channel
- **GPU:** NVIDIA RTX 2060 / AMD RX 5700 or better for smooth VR at native resolution
- **Storage:** NVMe SSD for VR install
- **USB:** USB 3.2 Gen 1 (5 Gbps) minimum for tethered headsets
