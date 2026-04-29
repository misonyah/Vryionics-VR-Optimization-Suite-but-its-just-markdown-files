# Live Session Optimization

Manual steps equivalent to what the VR live optimizer did automatically. Apply before or at the start of a VR session.

---

## Windows Services to Stop During VR

These services consume disk I/O, CPU, or network bandwidth during VR sessions. Stopping them before play and restarting after is safe.

**Stop (PowerShell, run as Admin):**
```powershell
Stop-Service SysMain -Force        # Superfetch / disk prefetch
Stop-Service DiagTrack -Force      # Connected User Experiences & Telemetry
Stop-Service WSearch -Force        # Windows Search Indexer
Stop-Service wuauserv -Force       # Windows Update
Stop-Service MapsBroker -Force     # Downloaded Maps Manager
Stop-Service XblGameSave -Force    # Xbox Game Save
Stop-Service XboxGipSvc -Force     # Xbox Accessory Management
Stop-Service XblAuthManager -Force # Xbox Live Auth Manager
Stop-Service TabletInputService -Force # Touch Keyboard and Handwriting
```

**Restart after session:**
```powershell
Start-Service SysMain, WSearch, wuauserv, DiagTrack
# (others will restart on next reboot or don't need manual restart)
```

---

## Processes to Close Before VR

These are background apps that consume CPU, RAM, or disk I/O without helping VR. Close them manually or via Task Manager before starting your headset.

### Cloud sync
- `onedrive.exe` — OneDrive sync
- `googledrivefs.exe` / `googledrivesync.exe` — Google Drive
- `dropbox.exe` — Dropbox
- `megasync.exe` — MEGA sync
- `icloudservices.exe` — iCloud

### Game launchers (close if not launching from them)
- `epicgameslauncher.exe` — Epic Games
- `origin.exe` / `eadesktop.exe` — EA
- `battle.net.exe` / `battlenet.exe` — Blizzard
- `galaxyclient.exe` — GOG Galaxy
- `ubisoftgamelauncher.exe` — Ubisoft Connect
- `rglauncher.exe` — Rockstar

### Adobe background daemons
- `creativecloud.exe`, `adobeupdateservice.exe`, `adobedesktopservice.exe`

### Remote access
- `teamviewer_service.exe`, `anydesk.exe`

### LED/RGB software
- `razernaminglauncher.exe`, `razercentral.exe`
- `corsairhid.exe`

### Windows Update background workers
- `usocoreworker.exe`, `musnotificationux.exe`

### NVIDIA background services (optional — can cause issues with some setups)
- `nvidia share.exe` (ShadowPlay recording daemon — safe to close if not recording)
- `nvcontainer.exe` (NVIDIA telemetry)

---

## Processes to Protect (Never Kill)

These must be running for VR to work correctly. Do not close them.

**VR runtimes:** `vrserver.exe`, `vrcompositor.exe`, `vrmonitor.exe`, `ovrserver_x64.exe`, `virtualdesktop.streamer.exe`, `alvr_server.exe`  
**VR companions:** `xsoverlay.exe`, `fpsvr.exe`, `vrcx.exe`, `magicchatbox.exe`, `vrcosc.exe`, `openkneeboardapp.exe`  
**Tracking:** `slimevr.exe`, `amethyst.exe`, `driver4vr.exe`, `opentrack.exe`  
**Audio:** `voicemeeter.exe`, `voicemod.exe`, `eartrumpet.exe`  
**Streaming:** `obs64.exe`, `streamlabs desktop.exe`, `medal.exe`  
**Social:** `discord.exe`, `teamspeak.exe`, `mumble.exe`  
**System:** `explorer.exe`, `dwm.exe`, `lsass.exe`, `svchost.exe`

---

## VR Process Priority

Boost VR processes to High CPU priority after launching SteamVR:

```powershell
$ErrorActionPreference = 'SilentlyContinue'
foreach ($name in @('vrserver','vrcompositor','vrdashboard','vrchat','ovrserver_x64')) {
  Get-Process -Name $name | ForEach-Object {
    $_.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::High
  }
}
```

Alternatively, set IFEO to apply this at launch automatically — see [Windows Settings → VR Process Priority](windows-settings.md#vr-process-priority-at-launch-ifeo).

---

## Throttle Background Process Priorities

Lower background apps to BelowNormal priority so they can't compete with VR for CPU time:

```powershell
$ErrorActionPreference = 'SilentlyContinue'
$protect = @('vrserver','vrcompositor','vrchat','ovrserver_x64','steam','discord','obs64','explorer','dwm','audiodg')

Get-Process | Where-Object {
  $lower = $_.ProcessName.ToLower()
  -not ($protect -contains $lower)
} | ForEach-Object {
  try { $_.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::BelowNormal } catch {}
}
```

Restore after VR session:
```powershell
Get-Process | ForEach-Object {
  try { $_.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::Normal } catch {}
}
```

---

## Windows Timer Resolution — Lock to 0.5ms

Windows default timer tick is 15.6ms. VR compositors request a finer resolution (1ms or 0.5ms) to reduce frame delivery jitter. You can hold the 0.5ms request for the duration of a VR session.

**PowerShell script — run once at VR start:**
```powershell
# This keeps running until you kill it (Ctrl+C)
# Run in a separate PowerShell window and minimize it

Add-Type @"
using System;
using System.Runtime.InteropServices;
public class TimerRes {
    [DllImport("ntdll.dll", SetLastError=true)]
    public static extern int NtSetTimerResolution(uint DesiredResolution, bool SetResolution, out uint CurrentResolution);
}
"@

$cur = [uint32]0
# 5000 * 100ns = 0.5ms
[TimerRes]::NtSetTimerResolution(5000, $true, [ref]$cur) | Out-Null
Write-Host "Timer locked to 0.5ms. Press Ctrl+C to release."
Start-Sleep -Seconds 86400
```

Kill this process when done with VR to release the lock.

**Note on Windows 11:** See [Windows Settings → Win11 Global Timer Resolution](windows-settings.md#windows-11-global-timer-resolution) for the registry tweak that makes SteamVR's own timer request apply globally.

---

## Standby List Flush

The Windows standby list is cached memory from recently-closed apps. It's normally beneficial but can compete with VR working sets and cause sudden multi-frame stutters. Flush it before or during VR sessions.

**Requires:** Admin rights + SeProfileSingleProcessPrivilege (admin has this by default)

**Using Sysinternals RAMMap:** File → Empty → Empty Standby List

**PowerShell (admin):**
```powershell
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class StandbyFlusher {
    [DllImport("ntdll.dll")]
    public static extern int NtSetSystemInformation(int InfoClass, IntPtr Info, int Length);
    [DllImport("advapi32.dll", SetLastError=true)]
    public static extern bool OpenProcessToken(IntPtr ProcessHandle, uint DesiredAccess, out IntPtr TokenHandle);
    [DllImport("advapi32.dll", SetLastError=true)]
    public static extern bool LookupPrivilegeValueW(string SystemName, string Name, out long Luid);
    [DllImport("kernel32.dll")]
    public static extern IntPtr GetCurrentProcess();
    [DllImport("advapi32.dll", SetLastError=true)]
    public static extern bool AdjustTokenPrivileges(IntPtr TokenHandle, bool DisableAllPrivileges,
        ref TOKEN_PRIVS NewState, int BufferLength, IntPtr PreviousState, IntPtr ReturnLength);
    [DllImport("kernel32.dll")]
    public static extern bool CloseHandle(IntPtr handle);
    [StructLayout(LayoutKind.Sequential)]
    public struct TOKEN_PRIVS { public uint Count; public long Luid; public uint Attr; }
}
"@

$tok = [IntPtr]::Zero
[StandbyFlusher]::OpenProcessToken([StandbyFlusher]::GetCurrentProcess(), 0x28, [ref]$tok) | Out-Null
$luid = [long]0
[StandbyFlusher]::LookupPrivilegeValueW($null, "SeProfileSingleProcessPrivilege", [ref]$luid) | Out-Null
$p = New-Object StandbyFlusher+TOKEN_PRIVS
$p.Count = 1; $p.Luid = $luid; $p.Attr = 2
[StandbyFlusher]::AdjustTokenPrivileges($tok, $false, [ref]$p, 0, [IntPtr]::Zero, [IntPtr]::Zero) | Out-Null

$cmd = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(4)
[System.Runtime.InteropServices.Marshal]::WriteInt32($cmd, 4)  # MemoryPurgeStandbyList
[StandbyFlusher]::NtSetSystemInformation(80, $cmd, 4) | Out-Null
[System.Runtime.InteropServices.Marshal]::FreeHGlobal($cmd)
[StandbyFlusher]::CloseHandle($tok) | Out-Null
Write-Host "Standby list flushed."
```

Alternatively, use the much simpler Sysinternals **RAMMap** tool — File → Empty → Empty Standby List.

---

## EcoQoS (Windows 11 — push background apps to Efficiency cores)

On Windows 11 with Intel 12th gen+ or AMD Ryzen hybrid chips, you can push background apps to E-cores, freeing P-cores for VR.

```powershell
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class EcoQoS {
    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool SetProcessInformation(
        IntPtr hProcess,
        int ProcessInformationClass,
        ref int ProcessInformation,
        int ProcessInformationSize);
    [DllImport("kernel32.dll")]
    public static extern IntPtr OpenProcess(uint access, bool inherit, int pid);
    [DllImport("kernel32.dll")]
    public static extern bool CloseHandle(IntPtr h);
}
"@

$protect = @('vrserver','vrcompositor','vrchat','ovrserver_x64','steam','discord','obs64','explorer','dwm','audiodg')

Get-Process | Where-Object { -not ($protect -contains $_.ProcessName.ToLower()) } | ForEach-Object {
  try {
    $h = [EcoQoS]::OpenProcess(0x1F0FFF, $false, $_.Id)
    if ($h -ne [IntPtr]::Zero) {
      # ProcessPowerThrottling = 4, PROCESS_POWER_THROTTLING_EXECUTION_SPEED = 1
      $info = 1
      [EcoQoS]::SetProcessInformation($h, 4, [ref]$info, 4) | Out-Null
      [EcoQoS]::CloseHandle($h) | Out-Null
    }
  } catch {}
}
Write-Host "EcoQoS applied to background processes."
```

---

## Pre-Session Checklist

Quick reference before putting on the headset:

- [ ] Power plan set to High Performance
- [ ] Close cloud sync apps (OneDrive, Google Drive, Dropbox)
- [ ] Close game launchers not needed (Epic, EA, Blizzard)
- [ ] Close browser tabs / browser if possible
- [ ] Disable Xbox Game Bar recording if not using it
- [ ] Set VR process priorities to High (or rely on IFEO tweak)
- [ ] Flush standby list (RAMMap → Empty → Empty Standby List)
- [ ] For wireless: confirm Wi-Fi on 5GHz or 6GHz, power saving disabled
- [ ] Check GPU/CPU temps are normal before long session
