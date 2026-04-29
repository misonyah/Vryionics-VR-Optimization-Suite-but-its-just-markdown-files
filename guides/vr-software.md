# VR Software Settings

## SteamVR

### Async Reprojection (ATW)

**Must be enabled.** When the GPU misses a frame deadline, async reprojection synthesizes the missing frame by warping the last completed frame — keeping motion smooth instead of halving the frame rate.

`steamvr.vrsettings` (usually at `C:\Program Files (x86)\Steam\config\steamvr.vrsettings`):
```json
{
  "steamvr": {
    "allowAsyncReprojection": true,
    "motionSmoothing": true
  }
}
```

Or in SteamVR UI: Settings → Video → Motion Smoothing → On, Async Reprojection → On.

### Motion Smoothing

Keep enabled unless your GPU is consistently below 85% utilization and you want to avoid any reprojection artifacts. For most users, Motion Smoothing prevents hard drops to half-rate when scenes get demanding.

### Render Resolution (Supersampling)

If your GPU is above 80% utilization, lower SteamVR resolution to 80–100%. Running at 150%+ SS with a struggling GPU is counterproductive.

Rule: find the highest SS where your GPU stays below ~85% during typical gameplay.

SteamVR → Settings → Video → Render Resolution → Custom → adjust slider.

### OpenXR Runtime

SteamVR can serve as the OpenXR runtime for all apps, but Meta Quest users using Air Link or Link should prefer the Meta OpenXR runtime — it has lower overhead for native Quest connection methods (5–15% frame time improvement typical).

**Set Meta as default OpenXR runtime:**  
Meta Quest app → Settings → General → Set Oculus as active OpenXR runtime.

### SteamVR Beta branch

If you're on the Beta branch (Steam → SteamVR → Properties → Betas) and experiencing new crashes or frame pacing issues, roll back to Stable. Beta introduces regressions regularly.

---

## SteamVR Crash Patterns

If SteamVR crashes, the log files (`%ProgramFiles(x86)%\Steam\logs\`) contain the cause. Common signatures:

| Symptom | Likely cause | Fix |
|---|---|---|
| `0xc0000005` access violation | Third-party SteamVR driver hooking wrong memory | Remove non-default drivers from `steamapps\common\SteamVR\drivers\` |
| `0xc0000409` stack overrun | Overlay conflict (RTSS, MSI Afterburner, Discord overlay) | Disable overlays one at a time |
| Error 306/307 | DirectX hook conflict | Same as above — MSI Afterburner, Special K, RTSS |
| Error 108/109/300/301 | Init failure — stale processes or USB issue | Kill `vrserver.exe`/`vrcompositor.exe` in Task Manager, fully exit Steam, relaunch |
| Error 309 | Shared IPC compositor failure | Same as 300/301; also try rebooting |
| `DXGI_ERROR_DEVICE_REMOVED` | GPU TDR (driver crash/reset) | Roll back GPU driver; reduce supersampling; check PSU headroom |

---

## Conflicting Overlays and Injectors

The following tools hook into DirectX's presentation path and are the #1 documented cause of SteamVR Error 306/307 and `0xc0000409` crashes:

- RivaTuner Statistics Server (RTSS) / MSI Afterburner OSD
- Special K
- Discord overlay
- Steam in-game overlay (less common but possible)
- NVIDIA overlay (NvContainer)
- OBS browser capture mode

**Fix:** Exclude SteamVR processes in their settings, or close them before VR:
- RTSS: add `vrcompositor.exe` to the exclusion list (⚙ icon → Add)
- Discord: Settings → Game Overlay → disable "Enable in-game overlay" or add SteamVR to exceptions

---

## Memory Integrity (HVCI) and Legacy VR Drivers

If SteamVR fails to find your headset or shows driver errors:

Check: Settings → Privacy & Security → Windows Security → Device Security → Core Isolation details → Memory Integrity.

If it's **On**, it may be blocking an older driver. Affected hardware:
- HTC Vive / Vive Pro (pre-2022 drivers)
- HP Reverb G1/G2, WMR headsets
- Samsung Odyssey Plus, Lenovo Explorer
- Oculus Rift CV1, Rift S

Test: turn Memory Integrity Off, reboot, test VR. If it works, find the updated driver that passes HVCI validation (usually available from 2022 onwards) and re-enable.

---

## VRChat config.json

Location: `%USERPROFILE%\AppData\LocalLow\VRChat\VRChat\config.json`

Create or edit this file. Recommended settings for VR performance:

```json
{
  "dynamic_bone_max_affected_transform_count": 32,
  "dynamic_bone_max_collider_check_count": 8,
  "avatar_culling_enabled": true,
  "avatar_culling_distance": 25,
  "cache_size": 20480,
  "cache_expiry_delay": 30
}
```

### What each setting does

**`dynamic_bone_max_affected_transform_count`** (controls PhysBones simulation cap)  
- `0` = uncapped — in populated worlds with 20+ players, this becomes the single biggest CPU bottleneck (10,000+ bone transforms per frame at 90Hz)
- `32` = recommended VR setting — cuts main-thread CPU 60–80% in populated worlds with minimal visual difference
- `64` = conservative cap — still much better than uncapped

**`dynamic_bone_max_collider_check_count`**  
- `0` = uncapped collider checks — causes CPU spikes around avatars with full-body collision meshes
- `8` = eliminates worst-case spike while keeping physics functional

**`avatar_culling_enabled` + `avatar_culling_distance`**  
- Stops rendering avatars beyond 25 meters
- In busy worlds with 20+ players, uncullled far avatars account for 15–30% of total GPU time

**`cache_size`** (in MB, so 20480 = 20GB)  
- Default is ~10GB; VRChat purges frequently-visited avatars/worlds when it fills
- 20GB keeps ~50–100 avatar/world bundles cached between sessions

**`cache_expiry_delay`** (days)  
- 30 days keeps your regular avatars cached between weekly sessions

### In-game overrides
Settings → Performance Options → Avatar Performance Limiter overrides `dynamic_bone_max_*` when set. Set in-game to "Very Poor" to get a similar effect without editing config.json.

---

## VRChat MSAA

VRChat MSAA at 4× or 8× is extremely expensive in VR — each pixel is rendered 4× or 8× times. At VR resolution (~2160×2160 per eye), this kills GPU headroom.

**Recommendation:** 1× MSAA + SteamVR sharpening filter or headset upscaling provides better perceived quality at far lower GPU cost.

Set in VRChat: Settings → Graphics → Anti-Aliasing.

---

## VRChat Mirror Resolution

VRChat mirrors re-render the entire visible scene from a new viewpoint for each mirror. At high resolution this is effectively doubling GPU workload whenever mirrors are visible.

**Recommendation:** 256–512px mirror resolution is invisible in VR (your headset resolution limits how much detail you can perceive in a mirror). Cuts mirror rendering cost by 75–90%.

Set in VRChat Settings or via config.json `camera_res_height`.

---

## Connection Method Settings

### Meta Air Link
Best practices:
- PC → router via Ethernet
- Quest on dedicated 5GHz or 6GHz SSID
- Set render resolution in Meta Quest PC app (not SteamVR)
- Enable dynamic bitrate

### Virtual Desktop
Best practices:
- HEVC codec, 80% quality
- 150–300 Mbps bitrate (match Wi-Fi capacity)
- Enable Sliced Encoding for NVENC (reduces encode latency)
- PC on wired Ethernet

### ALVR
Best practices:
- Use nightly builds
- H.265/HEVC encoder
- NVENC if available
- Reduce bitrate first if experiencing packet loss
- Uses SteamVR as runtime — ensure vrserver is running

### WMR (HP Reverb G2, Samsung Odyssey, etc.)
- Install "OpenXR Tools for Windows Mixed Reality"
- Set WMR as OpenXR runtime
- In WMR Portal: Settings → Headset display → 90Hz mode
- Connect headset to Intel USB controller (not ASMedia expansion card)
- Disable SteamVR Motion Smoothing when using WMR reprojection (to avoid double-processing)

### PSVR2 on PC
- Requires USB 3.0 (≥5 Gbps) + DisplayPort 1.4
- Eye-tracked foveated rendering does NOT work on PC (PS5-exclusive feature)
- Compatible with SteamVR via OpenXR
