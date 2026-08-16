# Stream Deck VLC Control — Technical Documentation

Complete reference for the `com.rgpaul.vlc` Stream Deck plugin: architecture, action reference,
VLC HTTP API usage, build, deployment, troubleshooting, and how to add new actions.

- **Plugin UUID:** `com.rgpaul.vlc`
- **Manifest version:** 1.1.0 (SDKVersion 2, requires Stream Deck software ≥ 4.1)
- **Binary:** `vlc-remote` (macOS), `vlc-remote.exe` (Windows)
- **Platforms:** macOS ≥ 10.11, Windows 10
- **Upstream:** https://github.com/RGPaul/streamdeck-vlc
- **License:** MIT

This fork adds a **Step Back (-5s)** action on top of upstream.

---

## 1. What it is

A native C++ Stream Deck plugin that controls a running VLC media player over VLC's **HTTP
(Lua) web interface**. The plugin is a standalone executable launched by the Stream Deck
application; it speaks the Stream Deck WebSocket protocol on one side and plain HTTP to VLC on
the other. There is no VLC library linked in — all control happens through VLC's REST-ish
`requests/status.json` endpoint.

---

## 2. Architecture

```
┌──────────────────────┐   WebSocket (localhost, port from argv)   ┌────────────────────┐
│ Elgato Stream Deck   │◄──────────────────────────────────────────►│  vlc-remote        │
│ application          │   events: keyDown, willAppear, …           │  (this plugin)     │
│  ├─ keys / pedal     │   calls: setTitle, setState, logMessage    │                    │
│  └─ Property         │◄──────────────────────────────────────────►│  ├─ VlcStreamDeck  │
│     Inspector (HTML) │   sendToPropertyInspector / globalSettings │  │   Plugin        │
└──────────────────────┘                                            │  ├─ VlcConnection  │
                                                                    │  │   Manager       │
                                                                    │  ├─ VlcStatus      │
                                                                    │  └─ CallBackTimer  │
                                                                    └─────────┬──────────┘
                                                                              │ HTTP GET
                                                                              │ Basic auth
                                                                              ▼
                                                              ┌────────────────────────────┐
                                                              │ VLC media player           │
                                                              │ HTTP interface :8080       │
                                                              │ /requests/status.json      │
                                                              └────────────────────────────┘
```

### Components

| File | Responsibility |
|------|----------------|
| `src/main.cpp` | Entry point. Parses the 8 CLI args Stream Deck passes (`-port`, `-pluginUUID`, `-registerEvent`, `-info`), initialises `ESDLocalizer` with the app language, constructs `ESDConnectionManager` and runs the event loop. Exits with code 1 if `argc != 9`. |
| `src/VlcStreamDeckPlugin.{hpp,cpp}` | The plugin brain. Implements the Stream Deck SDK callbacks (`KeyDownForAction`, `WillAppearForAction`, `WillDisappearForAction`, `DidReceiveGlobalSettings`, …), tracks visible key contexts, dispatches actions, and pushes state/title back to the deck. |
| `src/VlcConnectionManager.{hpp,cpp}` | HTTP client. Builds and sends the `GET /requests/status.json?command=…` requests via Boost.Beast, handles Basic auth, parses the JSON response, and normalises errors into a payload with `error`/`message`/`logMessage`. |
| `src/VlcStatus.{hpp,cpp}` | Value object parsed from VLC's `status.json`: play state, volume, and `information.category.meta` fields (artist, album, artwork_url, title). |
| `src/CallBackTimer.hpp` | Minimal `std::thread` + `std::atomic<bool>` repeating timer. Runs the callback then sleeps `interval` ms. |
| `src/macos_pch.hpp`, `src/windows_pch.hpp` | Precompiled header stubs per platform. |
| `com.rgpaul.vlc.sdPlugin/` | The distributable plugin bundle: manifest, localisation, images, Property Inspector, and the compiled binaries. |

### Threading model

- **Main thread** — the Stream Deck WebSocket event loop (`ESDConnectionManager::Run()`), which
  invokes all `…ForAction` callbacks.
- **Timer thread** — `CallBackTimer` fires `UpdateTimer()` every **3000 ms**, which may call
  `updateVlcStatus()` and therefore perform a *blocking* HTTP request off the main thread.
- `_visibleContextsMutex` guards the three context sets (`_visiblePlayContexts`,
  `_visibleTitleContexts`, `_allVisibleContexts`) shared between those threads.

`VlcConnectionManager::sendGetRequest` presents a blocking interface to its caller, but drives
the exchange asynchronously (`async_resolve` → `async_connect` → `async_write` → `async_read`)
under a single `io_context::run_for(_requestTimeout)` deadline of **5 s**. A key press on the
main thread and the polling thread can both be inside it at once; each call creates its own
`io_context` and socket, so they share no state, and an unreachable host stalls the calling
thread for at most the timeout.

> The synchronous Beast calls cannot be used here: `beast::tcp_stream::expires_after` applies to
> **async operations only**, so a synchronous `connect()` ignores it and blocks for the OS-level
> TCP timeout instead — measured at **75 s** against an unroutable address.

---

## 3. Action reference

All actions are dispatched by UUID in `VlcStreamDeckPlugin::KeyDownForAction`. Each maps to one
HTTP GET against `/requests/status.json`.

| Action (deck name) | UUID | VLC request | Multi-Action | States |
|---|---|---|---|---|
| Title | `com.rgpaul.vlc.title` | *(none — display only)* | ✗ | 1 (`empty`, 10 pt, middle-aligned) |
| Play | `com.rgpaul.vlc.play` | `?command=pl_play` | ✓ | 1 (`play`) |
| Pause | `com.rgpaul.vlc.pause` | `?command=pl_forcepause` | ✓ | 1 (`pause`) |
| Play / Pause | `com.rgpaul.vlc.playpause` | `pl_play` when state 0, `pl_forcepause` when state 1 | ✗ | 2 (`play`, `pause`) |
| **Step Back** | `com.rgpaul.vlc.sstepback` | `?command=seek&val=-5S` | ✓ | 1 (`sstepback`) |
| Next | `com.rgpaul.vlc.next` | `?command=pl_next` | ✓ | 1 (`next`) |
| Previous | `com.rgpaul.vlc.prev` | `?command=pl_previous` | ✓ | 1 (`prev`) |
| Volume Up | `com.rgpaul.vlc.volumeup` | `?command=volume&val=+10` | ✓ | 1 (`volume_up`) |
| Volume Down | `com.rgpaul.vlc.volumedown` | `?command=volume&val=-10` | ✓ | 1 (`volume_down`) |

Notes:

- **Volume steps are ±10 raw VLC units**, not percent. VLC's scale is 0–512 where 256 = 100 %,
  so one press ≈ 3.9 % of nominal volume.
- **Step Back** uses VLC's relative seek syntax `-5S` (capital `S` = seconds). Changing the step
  size is a one-character edit — see §9.
- Only **Title** and **Play** register per-action context sets, because they are the only actions
  whose key appearance is updated from VLC state.

---

## 4. VLC HTTP API usage

### Endpoint

```
GET http://{host}:{port}/requests/status.json[?command=…&val=…]
Authorization: Basic base64(":" + password)
User-Agent:    Boost.Beast/…
HTTP/1.1
```

VLC's Lua HTTP interface uses an **empty username** and the configured password — hence the
`":" + password` construction in `VlcConnectionManager::sendGetRequest`.

### Response handling

| HTTP status | Behaviour |
|---|---|
| `200 OK` | Body parsed with `nlohmann::json`, returned as `outPayload`, function returns `true`. |
| `401 Unauthorized` | Returns `false`; payload = `{error: true, message: "Authentication to VLC Server failed.", logMessage: …}`. |
| anything else | Returns `false`; payload = `{error: true, message: "Received Unknown Error from Server.", logMessage: "… <code>"}`. |
| connection failure or timeout | Returns `false`; payload = `{error: true, message: "Error connecting to VLC Server.", logMessage: "… <reason>"}` where reason is the Asio error message or `timed out after 5s`. |
| malformed body / any exception | Caught as `std::exception` (covers `nlohmann::json::parse_error`); same error payload. Response bodies are capped at 1 MB by `http::response_parser::body_limit`. |

### Fields consumed from `status.json`

```jsonc
{
  "state": "playing" | "paused" | "stopped",   // → VlcStatus::PlayState
  "volume": 256,                               // → VlcStatus::volume()
  "information": {
    "category": {
      "meta": {
        "artist":      "…",   // → albumArtist()
        "album":       "…",   // → albumTitle()
        "artwork_url": "…",   // → artworkUrl()
        "title":       "…"    // → songTitle()  ← shown on the Title key
      }
    }
  }
}
```

Everything else in `status.json` (position, length, audio filters, equalizer, …) is ignored.

---

## 5. Settings and the Property Inspector

Settings are stored as **global settings** (shared by every key of this plugin), not per-action
settings. The plugin requests them in `WillAppearForAction` whenever no password is set yet.

| Key | Default in PI | Consumed by |
|---|---|---|
| `vlcHost` | `localhost` | `VlcConnectionManager::setHost` (class default `127.0.0.1`) |
| `vlcPort` | `8080` | `VlcConnectionManager::setPort` (class default `8080`) |
| `vlcPassword` | *(empty)* | `VlcConnectionManager::setPassword` — **required**; polling is skipped entirely while empty |

### PI ⇄ plugin messages

- PI → plugin: `setGlobalSettings` (on any field `onchange`), `getGlobalSettings` (on connect).
- Plugin → PI: `sendToPropertyInspector` with

  ```json
  { "event": "com.rgpaul.vlc.rpc.state", "state": "Connection Successful" | "<error message>" }
  ```

  The PI writes `state` into `#rpc-state` and a local timestamp into `#rpc-state-date`. This is
  the **Status** block at the bottom of the inspector — the fastest way to tell whether the
  plugin can reach VLC.

Files: `propertyinspector/index.html`, `propertyinspector/js/index_pi.js`, plus Elgato's stock
`common.js` / `common_pi.js` and `css/sdpi.css`.

---

## 6. State synchronisation

1. `CallBackTimer` fires `UpdateTimer()` every 3 s.
2. It proceeds only if **all four** hold: the deck connection exists, the VLC connection manager
   exists, `_lastUnsuccessfullCalls < 5`, and at least one key of this plugin is visible.
3. `updateVlcStatus()` skips everything when no password is set; otherwise it GETs `status.json`.
4. On success `_lastUnsuccessfullCalls` resets to 0 and `updateVlcStatus(payload)` runs:
   - every **Play** context gets `SetState(1 /* pause icon */)` while playing, `SetState(0)` otherwise;
   - every **Title** context gets `SetTitle(songTitle, …, HardwareAndSoftware)`.
5. Every key press also feeds its response through the same `processVlcResponse` path, so the
   title/state refresh immediately after any action rather than waiting for the next poll.

**Back-off after repeated failures.** After `kMaxUnsuccessfullCalls` (5) consecutive failures the
plugin does not stop polling — it drops to one attempt every `kRetryTickInterval` (10) ticks,
i.e. every 30 s instead of every 3 s. That keeps the log readable while still recovering on its
own once VLC becomes reachable, so starting VLC after the Stream Deck app now fills the Title
key within half a minute without a key press.

---

## 7. Building from source

### Dependencies

| Dependency | How it's provided | Notes |
|---|---|---|
| **Boost** 1.73 – **1.86** (`system`, `random`) | System install | ⚠️ upper bound — see the box below. Static libs, multithreaded, static runtime (`Boost_USE_STATIC_LIBS/…` in CMakeLists). macOS: `/usr/local/{include,lib}`; Windows: `C:\include`, `C:\lib`; or pass `-DBoost_ROOT=…`. |
| **nlohmann/json** | git submodule → `deps/nlohmann_json/include` | header-only |
| **websocketpp** | git submodule → `deps/websocketpp/include` | header-only; also supplies `base64_encode` |
| **StreamDeckSdk** | vendored in `deps/StreamDeckSdk` | Elgato's C++ sample, modified. `ESDConnectionManager.cpp`, `ESDLocalizer.cpp`, plus `ESDUtilitiesMac.cpp` / `ESDUtilitiesWindows.cpp`. |
| **cmake-modules** | git submodule (`cmake/`, from RGPaul/cmake-modules) | added to `CMAKE_MODULE_PATH` |

Requires **CMake ≥ 3.18** and **C++17**. Conan is optional — if `conanbuildinfo.cmake` exists in
the build dir it is picked up automatically.

```bash
git clone --recurse-submodules https://github.com/RGPaul/streamdeck-vlc.git
# or, in an existing clone:
git submodule update --init --recursive
```

> 🔴 **Boost must be ≤ 1.86.** The vendored websocketpp 0.8.x uses Asio APIs (`io_service`,
> `io_service::strand`, `expires_from_now`) that Boost **removed in 1.87**; against 1.87+ the
> build fails with dozens of errors in `websocketpp/transport/asio/*.hpp` — note the tell-tale
> `member reference type 'strand_ptr' (aka 'int') is not a pointer`. On macOS install a pinned,
> keg-only Boost and point CMake at it:
>
> ```bash
> brew install boost@1.85
> cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
>   -DCMAKE_INCLUDE_PATH=/usr/local/opt/boost@1.85/include \
>   -DCMAKE_LIBRARY_PATH=/usr/local/opt/boost@1.85/lib \
>   -DBoost_ROOT=/usr/local/opt/boost@1.85 \
>   -DBoost_NO_BOOST_CMAKE=ON
> ```
>
> The explicit `CMAKE_INCLUDE_PATH` is required because `CMakeLists.txt` calls
> `include_directories(BEFORE "/usr/local/include")`, which would otherwise put a newer
> system-wide Boost ahead of the pinned one. Verify the configure step prints
> `Found Boost: … 1.85.0 … found components: system random` — the components line matters; if it
> reads `Could NOT find Boost`, `Boost_LIBRARIES` is empty and you are relying on header-only
> fallbacks.

### macOS

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release   # plus the Boost flags above
cmake --build build --config Release
```

Produces `build/vlc-remote`. Requires Xcode (command line tools) and Boost.

> **Architecture matters.** The shipped binary is **x86_64 only**. On Apple Silicon it runs under
> Rosetta 2 (fine), but if you rebuild, add `-DCMAKE_OSX_ARCHITECTURES=arm64` (or
> `"arm64;x86_64"` for a universal binary) and make sure your Boost build matches the target arch.
> Check with `lipo -archs vlc-remote`.

### Windows 10

```powershell
cmake -G "Visual Studio 16 2019" -A x64 -S . -B build64
cmake --build build64 --config Release
```

### Packaging

Copy the built binary into the bundle next to the manifest:

```bash
cp build/vlc-remote com.rgpaul.vlc.sdPlugin/vlc-remote
chmod +x com.rgpaul.vlc.sdPlugin/vlc-remote
```

`com.rgpaul.vlc.sdPlugin.zip` in the repo root is a prebuilt snapshot of that bundle (it also
carries `vlc-remote-orig`, the pre-fork 2023 binary, and `manifest.json-back`, an earlier
manifest — neither is used at runtime; the manifest only ever references `CodePathMac` /
`CodePathWin`).

---

## 8. Installation and deployment

### Plugin location

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/com.elgato.StreamDeck/Plugins/com.rgpaul.vlc.sdPlugin` |
| Windows | `%APPDATA%\Elgato\StreamDeck\Plugins\com.rgpaul.vlc.sdPlugin` |

Only directories ending in `.sdPlugin` are scanned — anything else in `Plugins/` is ignored.

### Manual install (macOS)

```bash
DEST=~/Library/Application\ Support/com.elgato.StreamDeck/Plugins
cp -R com.rgpaul.vlc.sdPlugin "$DEST/"
chmod +x "$DEST/com.rgpaul.vlc.sdPlugin/vlc-remote"
xattr -dr com.apple.quarantine "$DEST/com.rgpaul.vlc.sdPlugin"   # ← see below
osascript -e 'quit app "Elgato Stream Deck"'; sleep 3; open -a "Elgato Stream Deck"
```

> 🔴 **Gatekeeper quarantine is the single most common install failure.** `vlc-remote` is
> **unsigned and unnotarised**. If the bundle arrived via AirDrop, a browser download, or a
> messaging app, macOS stamps it with `com.apple.quarantine` and silently refuses to execute it —
> Stream Deck shows the actions in the UI but the plugin process never starts and nothing
> happens on key press. Clear it with `xattr -dr com.apple.quarantine <bundle>` and restart the
> Stream Deck app. Verify with `xattr -l …/vlc-remote` (should print no quarantine line).

### Verifying the plugin is alive

```bash
pgrep -fl vlc-remote     # should show: …/vlc-remote -port <n> -pluginUUID <uuid> …
```

Stream Deck launches a plugin process only when at least one of its actions is placed on a
profile, and passes exactly 8 arguments (`-port`, `-pluginUUID`, `-registerEvent`, `-info`).

---

## 9. Enabling VLC's web interface

### macOS

1. VLC → Preferences → **Interface** → "HTTP web interface": tick the checkbox and set a
   password (**mandatory** — a blank password makes the plugin skip every request).
2. Restart VLC.
3. Verify at <http://localhost:8080> — leave the username blank, enter the password. The VLC web
   UI should load.

Preferences live in `~/Library/Preferences/org.videolan.vlc/vlcrc`; the relevant keys are
`extraintf=http`, `http-password=…`, and the commented `#http-port=8080` / `#http-host=`.

### Windows

1. Tools → Preferences → Show settings: **All** → Interface → **Main interfaces** → tick **Web**.
2. Under Main interfaces → **Lua** → set a password in **Lua HTTP**.
3. Save, restart VLC, allow the Windows firewall prompt.

### Then in Stream Deck

Add any VLC action to a key, open it, and set: **Host** `localhost`, **Port** `8080`,
**Password** = the VLC password. The Status line should read *Connection Successful*.

---

## 10. Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| Keys do nothing, no plugin process | Quarantined / non-executable binary | `xattr -dr com.apple.quarantine`, `chmod +x`, restart Stream Deck (§8) |
| PI Status: *Authentication to VLC Server failed.* | HTTP 401 — wrong password, or username entered somewhere | Password must match VLC's Lua HTTP password; username is always empty |
| PI Status: *Error connecting to VLC Server.* | TCP connect failed — VLC not running, web interface disabled, wrong host/port, firewall | `nc -z 127.0.0.1 8080`; check `extraintf=http` in `vlcrc`; restart VLC |
| Title key blank, actions work | Poller is in 30 s back-off, or the stream carries no `title` metadata | Wait ~30 s, or press any VLC key to resume fast polling (§6) |
| Actions appear but nothing on the deck responds | No physical deck/pedal enumerated | `system_profiler SPUSBDataType \| grep -i "stream deck"` — Elgato decks use vendor ID `0x0fd9`. Bus-powered pedals often fail behind chained USB hubs/docks; plug directly into the machine and swap the cable |
| Works in browser, not from the plugin | Host mismatch (`localhost` vs `127.0.0.1` vs LAN IP) | Try the literal `127.0.0.1` |
| Erratic behaviour after an update | Stale plugin process | Quit and relaunch the Stream Deck app; a reboot is the documented last resort |

### Logs

- Stream Deck application logs: `~/Library/Logs/ElgatoStreamDeck/StreamDeck*.log`
- The plugin writes through `LogMessage`, so its lines land in those same logs. Useful strings:
  `key pressed: com.rgpaul.vlc.…`, `will appear for action: …`, `received global settings`,
  `device did connect: …`, and `<action> failed: <logMessage>` for every failed request.

---

## 11. Known issues and quirks

### Fixed in this fork

The following were present in upstream and are resolved here (see §14 for the changelog):

1. ~~**`Play` sends `pl_pause`**~~ — `keyPressedPlay()` now calls `sendPlay()`, and both the Pause
   key and the pause half of Play/Pause use `pl_forcepause` so they can never resume playback.
2. ~~**Property Inspector throws on load**~~ — `index.html`'s wrapper now carries
   `id="mainWrapper"`, which `index_pi.js` expects.
3. ~~**`saveSettings()` assumes `settings` is defined**~~ — `settings` initialises to `{}`, the
   `didReceiveGlobalSettings` handler tolerates a missing payload, and `saveSettings()` returns
   early until `uuid` is known.
4. ~~**Polling never self-heals**~~ — replaced with the 30 s back-off described in §6.
5. ~~**No timeout**~~ — 5 s deadline over async operations (§2).
6. ~~**No response-body size limit**~~ — 1 MB `body_limit`.
7. ~~**A malformed response body killed the plugin**~~ — `nlohmann::json::parse` throws
   `parse_error`, which derives from `std::exception`, not `beast::system_error`; the old
   handler caught only the latter, so a non-JSON 200 response propagated an uncaught exception
   out of a `std::thread` and terminated the process. The catch is now `const std::exception&`.
8. ~~**Localisation gaps**~~ — `de.json` gained a `sstepback` entry (and German names for
   next/prev/volume, which were untranslated); `en.json`'s `sstepback` said *"Short Step -"* with
   the tooltip *"Jump to next track."* (copy-paste from `next`), now *"Step Back"* /
   *"Jump back 5 seconds."*

### Still open

1. **Password is stored in Stream Deck's global settings in plaintext** and sent as HTTP Basic
   auth over an unencrypted connection. Fine for `localhost`; do not point this at a VLC instance
   across an untrusted network.
2. **`manifest.json` is minified** (one long line) except for the hand-added Step Back block —
   diffs against upstream are noisy. `manifest.json-back` preserves an older, formatted copy.
3. **The bundled binary is x86_64 only** and unsigned — see §7 and §8.
4. **websocketpp 0.8.x does not compile against Boost ≥ 1.87**, which removed the deprecated
   `io_service` and `expires_from_now` Asio APIs. See §7 for the pinned-Boost workaround.

---

## 12. Adding a new action

The Step Back (-5s) action is the worked example — it touched exactly five places:

1. **`VlcConnectionManager.hpp`** — declare `bool sendSstepBack(nlohmann::json&) const;`
2. **`VlcConnectionManager.cpp`** — implement it with the VLC target string:
   ```cpp
   auto const target = "/requests/status.json?command=seek&val=-5S";
   return sendGetRequest(target, outPayload);
   ```
3. **`VlcStreamDeckPlugin.hpp` / `.cpp`** — add `keyPressedSstepBack(const nlohmann::json&)` and
   an `else if (inAction == "com.rgpaul.vlc.sstepback")` branch in `KeyDownForAction`.
4. **`manifest.json`** — add an `Actions[]` entry with `UUID`, `Name`, `Icon`, `States[].Image`,
   `Tooltip`, `SupportedInMultiActions`.
5. **Images** — `images/actions/<name>.png` + `@2x` (20×20 / 40×40) and
   `images/keys/<name>.png` + `@2x` (72×72 / 144×144).

Then rebuild, copy the binary into the bundle, and restart Stream Deck.

**Useful VLC commands** for new actions (all on `/requests/status.json`):

| Command | Effect |
|---|---|
| `seek&val=+30S` / `-5S` | relative seek in seconds (`M` = minutes, `H` = hours, bare number = seconds) |
| `seek&val=50%` | absolute seek to percentage |
| `pl_stop` | stop playback |
| `pl_forcepause` / `pl_forceresume` | unambiguous pause/resume (better than `pl_pause` for a dedicated Play key) |
| `pl_random`, `pl_loop`, `pl_repeat` | toggle shuffle / loop / repeat |
| `fullscreen` | toggle fullscreen |
| `key&val=…` | send an arbitrary VLC hotkey action |
| `volume&val=255` | absolute volume (0–512, 256 = 100 %) |
| `pl_empty`, `pl_delete&id=…` | playlist management |

---

## 13. Repository map

```
.
├── CMakeLists.txt                 # build definition (C++17, Boost, submodule includes)
├── cmake/                         # submodule: RGPaul/cmake-modules
├── deps/
│   ├── StreamDeckSdk/             # vendored Elgato C++ SDK (modified)
│   ├── nlohmann_json/             # submodule
│   └── websocketpp/               # submodule
├── src/
│   ├── main.cpp                   # entry point, arg parsing, event loop
│   ├── VlcStreamDeckPlugin.{hpp,cpp}   # SDK callbacks, action dispatch, context tracking
│   ├── VlcConnectionManager.{hpp,cpp}  # HTTP client for VLC
│   ├── VlcStatus.{hpp,cpp}        # status.json value object
│   ├── CallBackTimer.hpp          # 3 s polling timer
│   └── {macos,windows}_pch.hpp    # precompiled headers
├── com.rgpaul.vlc.sdPlugin/       # the plugin bundle
│   ├── manifest.json              # actions, UUIDs, code paths, versions
│   ├── en.json, de.json           # localisation
│   ├── images/{actions,keys}/     # 1x + @2x icons
│   └── propertyinspector/         # settings UI (HTML/CSS/JS)
├── com.rgpaul.vlc.sdPlugin.zip    # prebuilt bundle incl. binaries
├── images/                        # README screenshots
└── LICENSE                        # MIT
```

Note that the checked-in `com.rgpaul.vlc.sdPlugin/` directory contains **no binaries** — those
live only in the zip and in an installed copy. `.gitignore` keeps build output out of the tree.

---

## 14. Changelog (this fork)

### Step Back action

Adds `com.rgpaul.vlc.sstepback` → `seek&val=-5S`, with icons and manifest entry.

### Reliability and correctness pass

| Area | Change | File |
|---|---|---|
| Play key | `sendPause()` → `sendPlay()` — the Play key played nothing, it toggled | `VlcStreamDeckPlugin.cpp` |
| Pause key | `pl_pause` → `pl_forcepause`, so it can never resume | `VlcStreamDeckPlugin.cpp`, `VlcConnectionManager.{hpp,cpp}` |
| Play/Pause | pause half now uses `pl_forcepause` (state is already known, no need to toggle) | `VlcStreamDeckPlugin.cpp` |
| Request timeout | sync Beast calls → async chain under `run_for(5s)`; unreachable host went from **75 s → 5 s** | `VlcConnectionManager.cpp` |
| Response cap | `http::response_parser::body_limit(1 MB)` | `VlcConnectionManager.cpp` |
| Crash | `catch (beast::system_error&)` → `catch (const std::exception&)`, so a non-JSON body no longer terminates the process | `VlcConnectionManager.cpp` |
| Polling | hard stop after 5 failures → 30 s back-off that self-heals | `VlcStreamDeckPlugin.{hpp,cpp}` |
| Thread safety | `UpdateTimer()` now takes `_visibleContextsMutex` before reading `_allVisibleContexts` (it was an unsynchronised read from the timer thread) | `VlcStreamDeckPlugin.cpp` |
| Counter overflow | `_lastUnsuccessfullCalls` saturates instead of wrapping past 255 back into fast polling | `VlcStreamDeckPlugin.cpp` |
| PI crash | added `id="mainWrapper"`, which `index_pi.js` already referenced | `propertyinspector/index.html` |
| PI robustness | `settings` defaults to `{}`; missing payload tolerated; `saveSettings()` waits for `uuid` | `propertyinspector/js/index_pi.js` |
| Localisation | `de.json` gained `sstepback` + German names for next/prev/volume; `en.json`'s `sstepback` name and tooltip corrected | `de.json`, `en.json` |

Verified against a live VLC 3.x on `localhost:8080`: every command returns HTTP 200 in ~22 ms,
`pl_forcepause` on an already-paused player keeps it paused, a wrong password yields the
*Authentication to VLC Server failed.* payload, and an unroutable host fails in 5001 ms.

## 15. Credits

- Original plugin: **Ralph-Gordon Paul** — https://github.com/RGPaul/streamdeck-vlc
- Stream Deck C++ SDK: **Corsair Memory, Inc. / Elgato** — https://github.com/elgatosf/streamdeck-cpp
- MIT licensed. Not affiliated with Elgato or the VideoLAN Organization; Stream Deck and VLC are
  their respective trademarks.
