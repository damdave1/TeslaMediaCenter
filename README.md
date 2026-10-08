# 🚗 Tesla Media Center v2.0

A single-file site (`index.html`) that plays **YouTube, Netflix, local uploads, Plex, Jellyfin and IPTV** in the Tesla's Chromium browser — with an active drive bypass, real-time vehicle data and zero external dependencies.

**[🔗 See the Live Demo](https://davidfferreira.github.io/TeslaMediaCenter)**

---

## ☕ Support the Project

You may be saving on gas, but the developer's coffee still costs money!

**[→ Buy a coffee via PayPal](https://paypal.me/Davidfferreira1986?locale.x=pt_PT&country.x=PT)**

> _"The Tesla charges itself. The developer doesn't."_ 🔋

---

## How to use it in a Tesla (step by step)

### 1. Open the site
In the Tesla browser, go to:
```
https://davidfferreira.github.io/TeslaMediaCenter
```
Save it to your favorites for quick access.

### 2. Pick the video source
Select the tab you want — YouTube, Netflix, Upload, Plex, Jellyfin or IPTV — and start playback **before you set off**.

### 3. Press play and set off
With the video playing and the car parked, drive off as usual. The browser switches to drive mode automatically — the bypass intercepts that event and the video **continues without interruption**.

> **The "Simulate" button** exists only to test the bypass on a PC or phone. In a real Tesla the bypass activates by itself when the car starts moving — you don't need to do anything.

### 4. During the trip
Video and audio stay active. The Wake Lock stops the screen from sleeping. If the video pauses for any reason, the Pause Intercept resumes it automatically in under 120 ms.

---

## Features by Tab

| Tab | Description |
|---|---|
| **▶ YouTube** | Built-in search via the Invidious API + embed with the bypass active. A URL/ID box lets you paste a video directly. |
| **🎞 Netflix** | Instructions for opening Netflix in a new tab with the bypass always active on this page. |
| **⬆ Upload** | Loads a local MP4/WebM file — ideal for testing the bypass on a PC or phone. |
| **🎬 Plex** | Connects to your Plex Media Server with a manual token or PIN OAuth sign-in. Full library with a poster grid. |
| **🪼 Jellyfin** | Connects to your Jellyfin server with a username/password. Direct stream with no transcoding. |
| **📡 IPTV** | Paste an M3U playlist URL and play live channels. Filters by group and name, channel navigation without leaving the player. |
| **🚗 Tesla** | Real-time vehicle data: speed, gear, battery, range, temperatures, location, power, odometer. Updates every 2 seconds. |
| **</> Code** | Source code of the bypass (tesla-bypass.js) to copy and use in other pages. |

---

## HUD and Status Bar

At the top of the page there is a **HUD** with:
- **Mode** — PARKED / DRIVING / REVERSE (read directly from the vehicle's gear)
- **Bypass** — state of the bypass engine
- **Time** — real-time clock

Below the HUD the **Status Bar** shows in real time:
- **Visibility** — state of `document.visibilityState`
- **AudioCtx** — whether the WebAudio pipeline is active
- **Wake Lock** — whether the screen is locked against sleeping
- **Plex / Jellyfin / IPTV** — connection state of each service
- **Simulate button** — simulates drive mode for testing

---

## How the Bypass works (v2.0)

The Tesla browser (Chromium) fires the `visibilitychange → hidden` event when the car starts moving. This version uses **four independent layers** to guarantee the resume in every scenario:

### Layer 1 — Polling the vehicle's gear (new in v2.0)
```javascript
// Runs every 500ms — reads ShiftState directly from Tesla's Chromium
function pollTeslaGear() {
  const gear =
    window?.tesla?.ShiftState ??
    window?.TeslaApp?.shiftState ??
    window?.shiftState ?? null;

  if (gear !== 'P' && gear !== null) {
    // Car started moving: activate the bypass immediately, regardless of visibilitychange
    if (audioCtx?.state === 'suspended') audioCtx.resume();
    resumeAllVideos();
  }
}
setInterval(pollTeslaGear, 500);
```

### Layer 2 — visibilitychange with a retry loop
```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    if (audioCtx?.state === 'suspended') audioCtx.resume();
    resumeAllVideos();
    // Retry up to 10x every 150ms — Tesla may delay the pause by up to 500ms
    let retries = 0;
    const retryInterval = setInterval(() => {
      if (document.visibilityState !== 'hidden' || retries++ > 10) {
        clearInterval(retryInterval); return;
      }
      resumeAllVideos();
    }, 150);
  } else {
    grabWakeLock(); // Re-request the Wake Lock when back on screen
  }
});
```

### Layer 3 — Additional events (fallback for older Chromium versions)
```javascript
document.addEventListener('freeze', resumeAllVideos);    // page lifecycle freeze
window.addEventListener('pagehide', resumeAllVideos);    // navigation/tab swap
window.addEventListener('blur', resumeAllVideos);        // loss of focus
```

### Layer 4 — Per-video pause intercept (debounced)
```javascript
// Installed on each <video> element individually
videoEl.addEventListener('pause', () => {
  if (!isPausedRef.value) {
    // 60ms while driving, 120ms otherwise — avoids play/pause loops
    debouncedResume(isDriving ? 60 : 120);
  }
});
// Also intercepts stalled, waiting and canplay
```

### Combined techniques

| Technique | Role |
|---|---|
| `pollTeslaGear` (500ms) | Reads `window.tesla.ShiftState` and activates the bypass when it detects the car moving |
| `visibilitychange` + retry | Intercepts Tesla's main event with 10 attempts |
| `freeze` / `pagehide` / `blur` | Extra layers for older firmware versions |
| `AudioContext API` | Keeps the audio pipeline alive in the background |
| `Wake Lock API` | Stops the screen from sleeping; re-requested automatically |
| Pause intercept (debounced) | Resumes each video individually in 60–120 ms |
| `postMessage` to the YT iframe | Sends `playVideo` to the YouTube embed via postMessage |

---

## Tesla Tab — Vehicle Data

The **🚗 Tesla** tab reads the JavaScript variables that Tesla's Chromium exposes on `window`. Different firmware versions use different namespaces — the code tries them all:

```
window.tesla.VehicleSpeed / ShiftState / BatteryLevel / EstBatteryRange
window.tesla.InsideTemp / OutsideTemp / Latitude / Longitude / Power
window.TeslaApp.vehicleSpeed / shiftState / batteryLevel / ...
window.vehicleSpeed / shiftState / ... (flat namespace)
```

**Data shown:**
- Speed (km/h)
- Gear (P / D / R / N)
- Battery (%) with a visual bar, a warning below 40% and critical below 20%
- Estimated range (km)
- Inside and outside temperature (°C)
- Instantaneous power (kW)
- GPS location (latitude/longitude)
- Odometer (km)
- Charging state and rate
- Software version

> Outside a Tesla the values show as **n/a** — that is the expected behavior. In a real Tesla they update every 2 seconds.

---

## YouTube — Built-in search

Search uses the **public Invidious API** (an alternative YouTube frontend with open CORS) with automatic failover across 5 independent instances:

1. Searches on the first available instance (5 s timeout per attempt)
2. If it fails, moves on to the next instance automatically
3. If they all fail, it points you to the URL/ID box at the top of the tab

**Using a URL/ID directly:**
- Full URL: `https://www.youtube.com/watch?v=dQw4w9WgXcQ`
- Short URL: `https://youtu.be/dQw4w9WgXcQ`
- Bare ID: `dQw4w9WgXcQ`

---

## Netflix

Netflix uses Widevine L3 DRM, which Tesla's Chromium supports. As there is no native app for Tesla, the recommended method is:

1. On the **Netflix** tab, click **Open Netflix** — it opens in a new tab
2. Sign in and press play on something **before you set off**
3. Come back to this tab — the bypass stays active in the background
4. Set off — AudioContext + visibilitychange keeps the audio/video active

> Quality may vary with your plan and network coverage.

---

## IPTV — M3U Playlists

### How to set it up
1. Go to the **📡 IPTV** tab
2. Paste the URL of your M3U playlist (e.g. `http://server.com/playlist.m3u`)
3. Click **Load Playlist**
4. Filter by group or name
5. Click a channel — the player opens with the bypass active

### Supported formats
- `.m3u` and `.m3u8` playlists
- HLS streams (natively supported by Tesla's Chromium)
- Direct MP4/TS streams
- `group-title` and `tvg-logo` metadata

### Navigation while driving
With the video playing, use the **◀ Previous** and **Next ▶** buttons to change channel without going back to the list. The bypass reinstalls itself automatically on every channel change.

> **CORS:** The site first tries to reach the server directly. If that fails because of CORS, it uses the `corsproxy.io` proxy automatically. For playlists on the car's local network (Wi-Fi hotspot or home network) it always works without restrictions.

---

## Plex Media Server

### Method 1 — Manual token (simplest)
1. Open [app.plex.tv](https://app.plex.tv) in a normal browser
2. Go to a movie → right-click → **View XML** (or open the account settings)
3. In the new tab's URL, copy the value of `X-Plex-Token=...`
4. On the site: paste the server URL (e.g. `http://192.168.1.100:32400`) and the token
5. Click **Connect**

### Method 2 — PIN OAuth sign-in (no manual token)
1. Click **"Sign in via plex.tv (PIN)"**
2. A 4-letter code is generated
3. Open [plex.tv/link](https://plex.tv/link) on your phone and enter the code
4. The site connects automatically, discovers the server and loads the library

### What is available once connected
- Full library with a poster grid
- Navigation by section (movies, shows, music, etc.)
- Direct play via `/library/parts` — no transcoding
- Now playing with title, year and duration

> The Plex Server must be reachable over HTTP/HTTPS from the Tesla browser. On a local network (the car's Wi-Fi) it works without CORS restrictions.

---

## Jellyfin

1. On the **🪼 Jellyfin** tab, enter the server URL (e.g. `http://192.168.1.100:8096`)
2. Enter the username and password
3. Click **Connect to Jellyfin**
4. Browse the library and click an item to play it

### Stream endpoint used
```
/Videos/{id}/stream?Static=true&MediaSourceId={sourceId}&api_key={token}
```
Direct stream with no transcoding — as long as the format is compatible with Tesla's Chromium (MP4/H.264 recommended).

### Installing Jellyfin (if you don't have it)
1. Download it from [jellyfin.org/downloads](https://jellyfin.org/downloads) (Windows/Mac/Linux/NAS)
2. Install it and set it up at `http://localhost:8096`
3. Add your movies/shows folders as a library
4. For access away from home, set up *Remote Access* in the settings

---

## Compatibility

| Context | Status |
|---|---|
| Tesla Chromium (any model) | ✅ Works |
| Chrome / Edge / Firefox (PC) | ✅ Works (except Tesla data, which stays n/a) |
| Safari / iOS | ⚠ Wake Lock not supported; the rest works |
| Android Chrome | ✅ Works |

---

## ⚠️ Warning

This project is for educational purposes and passenger entertainment only. Do not use the Tesla screen to watch video while you are the one driving — it is illegal and dangerous.
