# CPU Optimizations for VR

## Power Plan

Always use **High Performance** or **Ultimate Performance** for VR. Balanced and Power Saver plans allow P-state frequency scaling that adds 1–10ms latency per transition.

```
; Apply High Performance
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

; Unlock + apply Ultimate Performance
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
powercfg /setactive e9a42b02-d5df-448d-aa00-03f14749eb61
```

**AMD Ryzen:** Use "AMD Ryzen Balanced" instead of generic High Performance — it correctly synchronizes the Infinity Fabric clock.

---

## Core Count

VR workloads run vrserver, vrcompositor, the game engine, physics, audio, and tracking simultaneously. Minimum practical core count for VR:

| Cores | Result |
|---|---|
| < 4 | Critical — VR threads compete with OS; frequent frame drops |
| 4–5 | Warning — marginal; close all background apps before VR |
| 6+ | Adequate |
| 8+ | Comfortable |

---

## Context Switch Rate

If Task Manager shows consistent CPU usage below 70% but VR still stutters, check context switch rate. Over 60,000/sec indicates thread contention from too many background services.

**Check:** Performance Monitor → Add Counter → Thread → Context Switches/sec → _Total.

**Fix:** Close background apps (Discord, browsers, Spotify, streaming software) before VR.

---

## Intel Hybrid CPUs (12th gen Alder Lake+)

Intel 12th gen and later have P-cores (Performance) and E-cores (Efficient). VR runtimes and games must run on P-cores — E-cores are significantly slower for latency-sensitive single-threaded work.

### Windows 11 (Thread Director)
Windows 11 handles P/E routing automatically for most apps. However older SteamVR drivers and some VR tools ignore the hints.

**If you see VR stutters:** manually set CPU affinity for VR processes.

### Windows 10 or manual override
Thread Director does not exist on Windows 10. Manual affinity is required.

**Common P-core affinity masks by generation:**

| CPU | P-cores | Affinity mask |
|---|---|---|
| i5-12600K (6P+4E) | 6P = 12 logical | 0x0FFF |
| i7-12700K (8P+4E) | 8P = 16 logical | 0xFFFF |
| i9-12900K (8P+8E) | 8P = 16 logical | 0xFFFF |
| i7-13700K (8P+8E) | 8P = 16 logical | 0xFFFF |
| i9-13900K (8P+16E) | 8P = 16 logical | 0xFFFF |

Check your specific P-core count and calculate your mask: P-cores come first in Intel's logical ordering.

**Set via launch option (Steam/shortcuts):**
```
cmd /c start /affinity FFFF /high "" %command%
```

**Processes to pin to P-cores:**
- `vrserver.exe`
- `vrcompositor.exe`
- `OVRServer_x64.exe`
- `VRChat.exe`, `BeatSaber.exe`, or your game

**Verify:** Open HWiNFO64 → Per-Core Effective Clock. During VR, P-core clocks should be near max. If E-cores are hot while P-cores idle, routing is wrong.

---

## Intel Raptor Lake 13th/14th Gen — Vmin Degradation

**Affected:** Core i5/i7/i9 13xxx and 14xxx K/KF/KS desktop variants (not mobile).

These chips had a documented voltage/oxidation issue where sustained high Vcore during boost caused gradual stability degradation. Symptoms: random crashes under sustained load (long VR sessions in populated worlds), increasing instability over time.

**Fix:** Update motherboard BIOS to a version that includes **microcode 0x12B** (shipped from mid-2024 onwards). After flashing, load the **Intel Default Settings** power profile in BIOS.

Intel extended the warranty on affected chips to 5 years. If post-microcode you still see instability, initiate an RMA.

---

## Intel Arrow Lake (Core Ultra Series 2)

**Affected:** Core Ultra 2 CPUs launched October 2024.

Launch BIOSes had VR performance regressions — Thread Director mis-routed workloads to E-cores, and some boards had incorrect L2 cache sub-timings.

**Fix:** Update to a BIOS from February 2025 or later. Also ensure Windows 11 24H2 with January 2025 cumulative updates (KB5050094) is installed.

---

## AMD Ryzen 3D V-Cache

If you have an X3D chip, VRChat and VR apps benefit from being pinned to the V-Cache CCD — but the correct launch option depends on whether your chip has one CCD or two. Shipping the wrong mask doesn't just fail to help, it can silently pin the game to the *non*-V-Cache CCD, which is worse than doing nothing.

### Single-CCD X3D parts (5800X3D, 7800X3D, 9800X3D)

The whole chip is the V-Cache CCD, so pinning is unambiguous.

Right-click VRChat in Steam → Properties → General → Launch Options:
```
cmd /c start /affinity FF /high "" %command%
```

`FF` covers this chip's 8 physical cores. (Older guides — including earlier versions of this one — used `FFFF`, which over-pins to 16 logical processors and isn't the correct mask for these 8-core parts.)

### Dual-CCD X3D parts (7900X3D, 7950X3D, 9950X3D) — do NOT hardcode an affinity mask

On these chips only one CCD has the extra V-Cache, and **which logical processor range that CCD occupies varies by motherboard BIOS** — there's no single mask that's correct across boards. A hardcoded mask here is a coin flip: get it wrong and you've pinned VR to the *slower* CCD while believing you optimized it.

Use priority only, no affinity:
```
cmd /c start /high "" %command%
```

If you want to actually pin to the correct CCD, verify which logical processors carry the V-Cache on your specific board first (Task Manager → Details → right-click process → Set affinity, cross-referenced with a tool that reports per-CCD cache size like HWiNFO64), then set that as a custom affinity mask — don't copy a mask from a guide written for a different board's BIOS.

### AMD 3D V-Cache driver (amd3dvcacheSvc)
If installed, you can register apps in the registry:
```
HKLM\SYSTEM\CurrentControlSet\Services\amd3dvcacheSvc\Parameters\Preferences\App\VRChat.exe
  Type = 1  (DWORD)

HKLM\SYSTEM\CurrentControlSet\Services\amd3dvcacheSvc\Parameters\Preferences\App\vrserver.exe
  Type = 1  (DWORD)
```

The launch option approach is more reliable as the driver's redirection can be overridden by Win11 foreground-focus heuristics.

---

## Thermal Throttling

CPU thermal throttling occurs at TjMax (~95–100°C for most modern CPUs). During throttle, the CPU drops below its base clock, severely impacting VR compositor and physics timing.

**Diagnose:** HWiNFO64 → CPU Package Power and CPU Core Temperatures. Look for "Thermal Throttle Limit" events.

**Fixes:**
- Replace dried-out thermal paste (standard CPU paste lasts 3–5 years)
- Upgrade CPU cooler
- Reduce background CPU load
- Check BIOS isn't setting aggressive PL1/PL2 power limits that cause throttle before thermal limits

---

## Single-Thread vs Multi-Thread Bottleneck

If one CPU core is at ~100% while overall CPU usage is low (20–50%), you have a **single-thread bottleneck**. This is usually `vrcompositor`, `vrserver`, or the game's main thread.

**Fix for single-thread bottleneck:** More cores won't help. You need a CPU with faster single-core performance (higher IPC × clock speed). Reduce in-game complexity instead of adding more cores.

---

## CPU Core Parking

On non-High-Performance power plans, Windows may park CPU cores to save power. Parked cores have a wake-up latency.

**Fix:** High Performance or Ultimate Performance power plan sets CPMINCORES to 100%, ensuring all cores stay active.

You can verify with `powercfg /query` — look for `CPMINCORES` (Processor performance core parking minimum cores).
