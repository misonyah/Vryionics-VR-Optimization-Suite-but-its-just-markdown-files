# Network & Wi-Fi for Wireless VR

Applies to: Meta Air Link, Virtual Desktop, ALVR, or any other PC-to-headset wireless streaming.

---

## Wi-Fi Band

| Band | VR verdict |
|---|---|
| 2.4GHz | **Unusable for VR.** Max ~150–300 Mbps, severe congestion, only 3 non-overlapping channels. |
| 5GHz | **Minimum acceptable.** 866+ Mbps link speed required. |
| 6GHz (Wi-Fi 6E/7) | **Ideal.** Currently uncongested spectrum, up to 1200MHz bandwidth, zero legacy device interference. |

Quest 3 and Quest Pro support 6GHz. Most older headsets are 5GHz only.

**Target signal strength:** >70% (-65 dBm or better) at your play area.

---

## Router setup

- **PC → router:** wired Ethernet only. No Wi-Fi on the PC side.
- **Headset → router:** dedicated access point in or near the play space — ideally on a 5GHz or 6GHz band not shared with other devices.
- **Channel selection:** avoid congested channels. On 5GHz, DFS channels (100–144) are typically much less crowded than 36–48.
- **Channel width:** 80MHz on 5GHz. 160MHz where supported.
- **Router placement:** line-of-sight to play area, <4m if possible.

**Recommended hardware:** Wi-Fi 6E access point dedicated to VR. Popular picks: TP-Link EAP673, ASUS ROG Rapture GT6E, TP-Link Deco XE75.

---

## Wi-Fi Adapter (PC side)

The PC's Wi-Fi adapter chipset matters for wireless VR even at good signal strength. Realtek and older Broadcom adapters have documented driver issues sustaining the 100–200 Mbps UDP streams VR uses.

| Chipset | VR rating |
|---|---|
| Intel AX200, AX210, BE200 | Excellent |
| Qualcomm FastConnect series | Excellent |
| MediaTek MT7922, MT7921 | Good |
| Realtek RTL8852/8822 | Poor — known UDP burst issues |
| Broadcom (older laptop) | Mixed |

**Upgrade path:**
- Desktop M.2 slot: Intel AX210 (~$25), Intel BE200 (~$40, Wi-Fi 7)
- Laptop M.2 (upgradeable): same chips above
- USB dongle: TP-Link AXE5400 USB, ASUS USB-AX56

**Workarounds for poor chipsets:**
- Disable Wi-Fi power saving (see below)
- Force 5GHz-only SSID (prevent 2.4GHz fallback)
- Keep router within line-of-sight, <4m from play space
- Uncheck "Allow the computer to turn off this device" in Device Manager → adapter → Power Management

---

## Wi-Fi Power Saving — disable it

Wi-Fi adapter power saving causes 10–50ms wake latency when the adapter "dozes" between packet bursts.

**Disable via Device Manager:**
Device Manager → Network Adapters → Wi-Fi adapter → Properties → Power Management → uncheck "Allow the computer to turn off this device to save power"

**Also in Advanced properties:**
Device Manager → Wi-Fi adapter → Properties → Advanced → Power Saving Mode → set to **Disabled** or **Maximum Performance**

**Power Plan setting:**
Power Options → Change plan settings → Change advanced power settings → Wireless Adapter Settings → Power Saving Mode → **Maximum Performance**

---

## Gateway Latency

Target: <5ms to your router.

Test: `ping <router-ip>` (usually `ping 192.168.1.1`). VR total motion-to-photon budget:
- Encode: ~5–8ms
- Wi-Fi transmission: measured above
- Decode: ~2ms
- Display scan-out: ~4ms

At >8ms gateway latency you're already spending your entire motion-to-photon budget on Wi-Fi alone.

---

## Channel Congestion

5GHz has more channels than 2.4GHz but still gets congested in apartment buildings.

**Check competing networks:** use Wi-Fi Analyzer (free Windows app) to see which channels nearby APs are on, then pick the least-used channel.

**DFS channels (100–144 on 5GHz):** fewer devices use these due to radar detection requirements. Worth trying if non-DFS is congested.

---

## Nagle Algorithm (TCP)

For wireless VR, disabling TCP Nagle reduces latency by 1–5ms by stopping coalescing of small control packets.

See [Windows Settings → Nagle Algorithm](windows-settings.md#tcp-nagle-algorithm-wireless-vr-only) for the registry steps. Apply to your Wi-Fi adapter's interface GUID.

---

## Streaming App Settings

### Virtual Desktop
- Codec: HEVC/H.265 for best quality/bitrate; AV1 if you have Intel Arc or RTX 4000+ for ~30% better compression
- Bitrate: 150–300 Mbps depending on router/headset capability
- Enable "Sliced Encoding" if using NVENC (reduces encode latency)

### Meta Air Link
- Set render resolution in the Meta Quest PC app (not SteamVR)
- Enable dynamic bitrate in Air Link settings
- Bitrate cap is ~200 Mbps — lower than Virtual Desktop
- Does not support AV1 codec as of 2025

### ALVR
- H.265/HEVC preferred
- Use NVENC encoder if available
- If experiencing packet loss, reduce bitrate first before any other change
- Use nightlies for the latest codec improvements

---

## Wi-Fi 6E upgrade opportunity

If you're on 5GHz and using Quest 3 or Quest Pro, a Wi-Fi 6E router (6GHz band) is the single biggest wireless VR upgrade available. The 6GHz band currently has no competing devices in most homes, providing dedicated spectrum with consistent throughput.

Worth doing if:
- You have a compatible headset (Quest 3, Quest Pro, or other 6GHz-capable model)
- Your current 5GHz has channel congestion issues
- You stream at high bitrates (150+ Mbps) and see compression artifacts
