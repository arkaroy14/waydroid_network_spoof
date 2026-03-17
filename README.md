# 📶 Waydroid Network Spoofer

An LSPosed/Xposed module that makes apps believe they are on **WiFi + Cellular (LTE)**
instead of Ethernet — primarily for Waydroid, works on any rooted Android device.

---

## Why this exists

Waydroid exposes all network traffic through a virtual Ethernet adapter (`eth0`,
`TRANSPORT_ETHERNET`, `getType()=9`). Many apps hard-check for WiFi or Cellular
before enabling features (video streaming, payments, content unlock). This module
intercepts the relevant Android network APIs at runtime and replaces every Ethernet
signal with real-looking WiFi + LTE values — no system modification required.

---

## What it spoofs

| API | Real Waydroid value | Spoofed value |
|---|---|---|
| `hasTransport(ETHERNET)` | `true` | `false` |
| `hasTransport(WIFI)` | `false` | `true` |
| `hasTransport(CELLULAR)` | `false` | `true` |
| `NetworkInfo.getType()` | `9` (ETHERNET) | `1` (WIFI) |
| `NetworkInfo.getTypeName()` | `"Ethernet"` | `"WIFI"` |
| `NetworkInterface eth0` | `eth0` | `wlan0` |
| `NetworkInterface eth1` | `eth1` | `rmnet_data3` |
| `NetworkInterface eth2+` | `eth2+` | `rmnet_data4+` |
| `WifiManager.isWifiEnabled()` | `false` | `true` |
| `WifiManager.getWifiState()` | `1` | `3` (ENABLED) |
| `TelephonyManager.getNetworkType()` | `0` | `13` (LTE) |
| `TelephonyManager.getDataState()` | `- ` | `2` (CONNECTED) |
| `ConnectivityManager.getActiveNetworkInfo().mType` | `9` | `1` |

**Active transport:** WiFi wins when both present — matches real Android dual-stack behaviour.

---

## Included APKs

| APK | Purpose |
|---|---|
| `WaydroidNetworkSpoof` | The Xposed module — install and scope to target apps |
| `WaydroidNetworkTest` | Diagnostic companion — shows every transport/capability value |

---

## Setup

### Spoofer module

1. Install `WaydroidNetworkSpoof.apk`
2. Open **LSPosed → Modules** → enable `Network Spoofer`
3. Tap the module → **Scope** → add your target app(s)
4. Force-stop / restart the target app
5. Done — no configuration needed, both WiFi + Cellular are always active

### Verify with the test app

1. Install `WaydroidNetworkTest.apk` — **do not add it to the spoofer scope**
2. Run it before enabling the spoofer to see the raw Waydroid values
3. Add your target app to scope, restart it
4. Rerun the test app scoped to itself if you want to confirm the hooks work end-to-end
   (or scope the test app temporarily and rerun)

Expected output after spoofing:

```
✅ TRANSPORT_WIFI     : true
✅ TRANSPORT_CELLULAR : true
❌ TRANSPORT_ETHERNET : false
ℹ️ type int           : 1  (WIFI)
ℹ️ interface          : wlan0
```

---

## How it works

### Hook targets

```
android.net.NetworkCapabilities   hasTransport(int)
android.net.NetworkInfo           getType(), getTypeName()
android.net.ConnectivityManager   getActiveNetworkInfo()
android.net.wifi.WifiManager      getWifiState(), isWifiEnabled()
android.telephony.TelephonyManager  getNetworkType(), getDataNetworkType(), getDataState()
java.net.NetworkInterface         getNetworkInterfaces(), getName(), getDisplayName()
```

### NetworkInterface special handling

`NetworkInterface.getName()` is a **native method** — Xposed cannot hook it directly.
The module instead intercepts `getNetworkInterfaces()` (a Java method), builds a
`WeakHashMap<NetworkInterface, String>` of name overrides as the enumeration is walked,
then intercepts `getName()` and `getDisplayName()` to return from the map.
`java.net.NetworkInterface` is also a bootstrap class, so it must be referenced as
`java.net.NetworkInterface::class.java` and hooked via `XposedBridge.hookAllMethods`
rather than `XposedHelpers.findAndHookMethod` with `lpparam.classLoader`
(the latter silently fails on bootstrap classes).

---

## Requirements

- Android 8.0+ (API 26+)
- LSPosed (recommended) or EdXposed
- Root (required by LSPosed)

---

## Limitations

- Hook takes effect only after the target app process restarts
- `NetworkCallback`-based listeners (apps using `registerNetworkCallback`) may still
  receive the real underlying `Network` object from the system; the module patches
  the query APIs but does not forge `Network` objects
- DNS still resolves through the real Ethernet interface — only the transport type
  reported to apps is changed
- Does not affect system UI network indicator (that reads from a separate system service)

---

## 📱 Screenshots (Before Hook)
![before_wifi_spoof](https://github.com/user-attachments/assets/066a0408-62c2-4ab7-9c3c-e530696293c2)

## 📱 Screenshots (Before Hook)
![after_wifi_spoof](https://github.com/user-attachments/assets/cecb97d5-8202-4346-bba3-e9d5e1dc6e49)

