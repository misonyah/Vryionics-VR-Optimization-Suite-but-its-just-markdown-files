# Windows Settings for VR

## MMCSS — CPU Scheduling

MMCSS (Multimedia Class Scheduler Service) controls how Windows allocates CPU time to real-time tasks like VR and audio.

### SystemResponsiveness → 0 (disputed — likely no effect)

**What:** Reserves 0% CPU for background tasks. Default is 20%.  
**Why it's on this list:** The classic advice is that at 20%, MMCSS withholds CPU time from VR even when nothing else needs it.  
**Why it's disputed:** On modern Windows builds the value gets normalized internally regardless of what you set — multiple independent testers have measured no change with this applied. Treat it as a legacy myth fix, not a real optimization.  
**Registry (if you want to try it anyway):**
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
  SystemResponsiveness = 0 (DWORD)
```
**Default:** 20

### NetworkThrottlingIndex → disabled

**What:** Stops Windows throttling network packets when multimedia is active.  
**Why:** Default caps network at ~10 packets/ms — counterproductive for wireless VR streaming.  
**Registry:**
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile
  NetworkThrottlingIndex = 0xFFFFFFFF (DWORD)   ; = 4294967295
```
**Default:** 10

### Games task priority

**What:** Sets the MMCSS "Games" task to highest scheduling category.  
**Registry:**
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games
  Priority             = 6       (DWORD)
  GPU Priority         = 8       (DWORD)
  Scheduling Category  = High    (REG_SZ)
```
**Defaults:** Priority=2, Category=Medium

---

## Power Plan

```
; High Performance
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

; Ultimate Performance (unlock first if not available)
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
powercfg /setactive e9a42b02-d5df-448d-aa00-03f14749eb61
```

**Why:** Non-HP plans allow P-state transitions that add 1–10ms CPU latency at every frequency change.

**AMD X3D CPUs are the exception — use "AMD Ryzen Balanced" instead, not High Performance.** On X3D chips, High Performance overrides the CPPC hints that route threads onto the V-Cache CCD, running hotter for worse VR frame times. SteamVR also force-switches you to High Performance on launch, so you need to either delete your other power plans or switch back manually every session. See [CPU → Power Plan](cpu.md#power-plan) for the full explanation. This does not apply to Intel or non-X3D AMD CPUs.

---

## Windows 11 25H2 — known FPS regression

If you updated to 25H2 (2025 Update) and VR/game FPS dropped noticeably afterward (drops of tens of FPS have been widely reported, both NVIDIA and AMD), you're hitting a documented regression, not a config problem on your end. Fix:
1. Install `KB5095093` (or later cumulative update) — Microsoft's fix for the 25H2 performance regression.
2. Update your GPU driver — NVIDIA shipped emergency drivers (596.xx+) specifically addressing 25H2/24H2 conflicts; AMD has equivalent updates.

Check your build with `winver`. Don't chase registry tweaks for a stutter/FPS-drop that's actually this.

---

## Windows 11 Global Timer Resolution

**Affects:** Windows 11 build 22621 (22H2) and later only.

**What:** On Win11, `timeBeginPeriod(1)` calls from apps (like SteamVR) no longer raise the *global* timer to 0.5ms — only that app's timer is affected. This flag restores the old behavior.

**Registry:**
```
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\kernel
  GlobalTimerResolutionRequests = 1  (DWORD)
```

Reboot required.

---

## Xbox Game Bar / DVR

**What:** Removes background recording hooks that run in every application.  
**Registry:**
```
HKCU\System\GameConfigStore
  GameDVR_Enabled = 0  (DWORD)

HKCU\Software\Microsoft\Windows\CurrentVersion\GameDVR
  AppCaptureEnabled = 0  (DWORD)
```

---

## Windows Game Mode

**Enable it** — Settings → Gaming → Game Mode → On.

Game Mode suspends Windows Update delivery during gameplay and improves GPU scheduling for the foreground game. Contrary to old advice, it's beneficial for VR on Windows 11.

---

## Disable USB Selective Suspend (USB-tethered headsets only)

**Applies to:** Meta Link, Pico Connect, or any USB-tethered headset.

Power Options → Change plan settings → Change advanced power settings → USB settings → USB selective suspend setting → **Disabled**.

**Why:** D3 wake latency is 10–50ms. At 90Hz, the headset needs continuous data every 11ms.

---

## Disable Hyper-V

When Hyper-V is enabled, Windows itself runs as a hypervisor guest. This:
- Virtualizes the TSC (Time Stamp Counter)
- Adds ~1ms+ interrupt latency
- Disrupts VR compositor frame scheduling

To disable: **Turn Windows Features on or off** → uncheck:
- Hyper-V
- Virtual Machine Platform
- Windows Hypervisor Platform

Reboot required.

**Note:** Disabling these also disables WSL2 and Android subsystem.

---

## TCP Nagle Algorithm (wireless VR only)

Nagle coalesces small TCP packets, adding up to 200ms latency waiting for a full segment. For wireless VR pose/control packets, this adds measurable latency.

**Find your Wi-Fi adapter's GUID:**
```
Get-NetAdapter | Where-Object { $_.Name -like "*Wi-Fi*" } | Select-Object InterfaceGuid
```

**Registry (replace `{adapter-guid}` with the actual GUID):**
```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{adapter-guid}
  TcpAckFrequency = 1  (DWORD)
  TCPNoDelay      = 1  (DWORD)
```

---

## Windows Update — prevent auto-reboot

Stops Windows from force-restarting while you're in a session.

**Registry:**
```
HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU
  NoAutoRebootWithLoggedOnUsers = 1  (DWORD)
  AUOptions                     = 2  (DWORD)   ; 2 = notify before download
```

---

## Delivery Optimization — disable P2P upload

Stops Windows using your connection to seed updates to other PCs.

**Registry:**
```
HKLM\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization
  DODownloadMode = 0  (DWORD)   ; 0 = HTTP only, no peers
```

---

## Windows Defender — VR folder exclusions (not recommended)

Real-time scanning does cause 2–50ms hitches when VRChat loads avatar bundles, but excluding your Steam library or VRChat cache from Defender creates a predictable, world-writable "safe haven" directory — anything that lands in an excluded folder (including a malicious avatar asset or a dropped payload) runs unscanned. That's a standing security hole for a few milliseconds of load-time smoothness, and there's no way to write this up responsibly as a "quick fix." If you still want to do it after weighing that tradeoff, the mechanism is `Add-MpPreference -ExclusionPath`, but scope it as narrowly as possible and never exclude a whole drive.

---

## VR Process Priority at Launch (IFEO)

Sets VR processes to launch at High priority before they even start executing. More reliable than post-launch priority tools.

**Don't include `vrserver.exe` or `vrcompositor.exe` here.** Raising the compositor's own priority above the workload it's compositing is a priority-inversion antipattern — it doesn't help, and can make frame delivery worse. Boost the game/client processes only.

**Registry (one key per exe):**
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\VRChat.exe\PerfOptions
  CpuPriorityClass = 3  (DWORD)   ; 3 = High

HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\OVRServer_x64.exe\PerfOptions
  CpuPriorityClass = 3  (DWORD)
```

---

## Windows 11 EcoQoS / Power Throttling

On Win11 with non-High-Performance power plans, Windows can silently throttle VR processes to efficiency cores via EcoQoS. The fix is simply: **use High Performance or Ultimate Performance power plan**.

---

## Virtualization overhead

If you have Hyper-V, VirtualBox, or WSL2 drivers loaded, they intercept CPU instructions and increase interrupt latency even with no VMs running. Disable them before VR sessions.

Check active drivers:
```powershell
Get-Service | Where-Object { $_.Name -match 'vmms|vboxdrv|WslService' }
```
