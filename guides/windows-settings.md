# Windows Settings for VR

## MMCSS — CPU Scheduling

MMCSS (Multimedia Class Scheduler Service) controls how Windows allocates CPU time to real-time tasks like VR and audio.

### SystemResponsiveness → 0

**What:** Reserves 0% CPU for background tasks. Default is 20%.  
**Why:** At 20%, MMCSS withholds 20% of CPU time from VR even when nothing else needs it.  
**Registry:**
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

**AMD users:** "AMD Ryzen Balanced" is preferable to generic High Performance — it understands Infinity Fabric frequency properly.

**Why:** Non-HP plans allow P-state transitions that add 1–10ms CPU latency at every frequency change.

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

## Windows Defender — VR folder exclusions

Real-time scanning causes 2–50ms hitches when VRChat loads avatar bundles.

**PowerShell (run as Admin):**
```powershell
Add-MpPreference -ExclusionPath "C:\Program Files (x86)\Steam\steamapps"
Add-MpPreference -ExclusionPath "$env:LOCALAPPDATA\..\LocalLow\VRChat\VRChat"
```

**Security tradeoff:** Only exclude folders you trust. Don't add your entire C: drive.

---

## VR Process Priority at Launch (IFEO)

Sets VR processes to launch at High priority before they even start executing. More reliable than post-launch priority tools.

**Registry (one key per exe):**
```
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\vrserver.exe\PerfOptions
  CpuPriorityClass = 3  (DWORD)   ; 3 = High

HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\vrcompositor.exe\PerfOptions
  CpuPriorityClass = 3  (DWORD)

HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\VRChat.exe\PerfOptions
  CpuPriorityClass = 3  (DWORD)

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
