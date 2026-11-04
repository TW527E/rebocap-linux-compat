# rebocap-linux-compat (TW527E fork)

**This fork is used without SteamVR.** I only use `bridge.exe` from it, to feed Rebocap's tracking to
[WiVRn](https://github.com/WiVRn/WiVRn) through [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge):

```
Rebocap (Wine / Bottles) <--named pipe--> bridge.exe (Wine) <--TCP--> rebocap-wivrn-bridge CLI --SolarXR--> WiVRn --> xrizer --> VRChat
```

For that setup you don't need `install.py`, the SteamVR driver, or SteamVR. The full guide, launch script and
troubleshooting notes are in [TW527E/rebocap-linux-vrchat-fbt](https://github.com/TW527E/rebocap-linux-vrchat-fbt).

What this fork changes: only `bridge/bridge.c`. It works around socket and named-pipe bugs in Wine builds with ntsync
(tested in a Flatpak Bottles bottle, runner soda-11.0-10) and adds `bridge.quit` for a clean exit.
See [Bridge under Bottles / Wine with ntsync](#bridge-under-bottles--wine-with-ntsync) below.

Get `bridge.exe` from [Releases](https://github.com/TW527E/rebocap-linux-compat/releases), or build it:

```
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32
```

Everything below is the upstream README from
[dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat). Its SteamVR driver path
is not used or tested in this fork.

---

**這個 fork 不是拿來接 SteamVR 的。** 我只用裡面的 `bridge.exe`，搭配
[akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge) 把 Rebocap 的追蹤資料送進 WiVRn，
不需要 `install.py`、SteamVR 驅動，也不需要 SteamVR。完整的教學、啟動腳本和除錯紀錄在
[TW527E/rebocap-linux-vrchat-fbt](https://github.com/TW527E/rebocap-linux-vrchat-fbt)。

這個 fork 只改了 `bridge/bridge.c`：修正 Wine + ntsync（Flatpak Bottles，runner soda-11.0-10）下 socket 和 named pipe 的問題，
並加上 `bridge.quit` 讓 bridge 可以正常退出。編好的 `bridge.exe` 在 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases)。
下面是原專案的 README，裡面 SteamVR 驅動的部分這個 fork 沒有使用也沒有測試。

---

## Upstream: Rebocap on Linux SteamVR

This project attempts to make Rebocap full body tracking work under Wine, and connects it to native Linux SteamVR.

Rebocap's Windows package has three parts:

The app runs under Wine without any changes.
This project replaces the application's steamvr DLL with a native driver that uses the same protocol.

- the GUI app, which talks to the USB dongle and does all pose estimation
- a SteamVR driver DLL
- a named pipe between them

```
dongle --/dev/ttyACM*--> rebocap.exe (Wine) <--named pipe--> bridge.exe (Wine) <--TCP--> driver_rebocap.so (vrserver)
```

The repository doesn't contain any files from the application itself. You'll need to download the Windows installer from Rebocap.
The install script unpacks it and copies the driver's icons and input profiles from it.

### Requirements

- SteamVR for Linux, Wine (tested with 11.15), `innoextract`
- Build tools: `cmake`, a C++17 compiler, `mingw-w64-gcc`, OpenVR headers (`openvr` package), `nlohmann-json`.
  The development tools in `tools/` and `driver/test/` also need `protoc` and the Python `protobuf` package.
- A static (non-variable) sans-serif font such as Noto Sans as Wine renders variable fonts badly.
- Permission to open the dongle's serial port. Your user must be in the group that owns `/dev/ttyACM*` (usually `uucp` on Arch-based systems, `dialout` on Debian-based systems).

### Install

Close SteamVR and run:

```
./install.py /path/to/rebocap_release_vXX.exe
```

The script installs the utils into `~/.local/share/rebocap-linux` (change with `--home DIR` if desired),
registers the driver with SteamVR, and creates a `rebocap` command with desktop entry.

- **Update after pulling changes:** run `./install.py` with no arguments.
  This will redo the entire install process for the Linux side of things without modifying the Rebocap install.
- **Newer vendor release:** run `./install.py` with the new installer.
  It replaces the app's program files and keeps the data that the app created (device info, recordings).
- **Uninstall:** `./install.py --uninstall` unregisters the driver and removes the launchers. Doesn't touch the Rebocap install.

Driver settings are stored in SteamVR's settings file.

***For safety do any firmware updates for the trackers and the receiver only on Windows. Do not do them under Wine.***

### Use

1. Plug in the dongle and run `rebocap`. This starts the app and the bridge.
2. Start SteamVR and connect the headset.
3. Choose VR mode in Rebocap. The SteamVR indicator turns green when the driver is connected.
   Calibrate with the headset on, the trackers should appear in SteamVR (same like)

You can start and restart the app and SteamVR in any order.

In PC mode the app sends nothing to SteamVR. The app behaves the same on Windows.

#### Walk-in-place

Walk-in-place is supported. Enable "Walk-In-Place" in the app's "Walk-In-Place & Swap Pos" section.
The driver sets the walking speed to the thumbstick of the hand controller selected under "Ctrl Swap".
If you push the real stick while walking, it sets the direction. When you stop walking, the real stick works normally again.

By default the driver pushes the stick forward, and the game decides which direction that is.
To walk in the direction your hips face, close SteamVR and run:

```
rebocap-config wip_direction waist
rebocap-config wip_game_reference head         # or: controller, for games that move you where the hand points
```

`wip_game_reference` must match how the game interprets the stick.
If it is wrong, you'll walk at an angle that changes when you turn your head or hand.

#### Settings

`rebocap-config` allows showing and changing settings for steamvr.vrsettings.

| key                          | default              | meaning                                                                                                          |
|------------------------------|----------------------|------------------------------------------------------------------------------------------------------------------|
| `bridge_host`, `bridge_port` | `127.0.0.1`, `36850` | where `bridge.exe` listens                                                                                       |
| `input_hooks`                | `true`               | lets walk-in-place and hand pose replacement change the hand controllers' input and poses (see "Things to know") |
| `wip_direction`              | `game`               | `game` or `waist`                                                                                                |
| `wip_game_reference`         | `head`               | `head` or `controller`. Used only with `waist`.                                                                  |
| `universe_fallback`          | `true`               | use raw tracking space if SteamVR's play space cannot be found                                                   |

#### App on one machine, SteamVR on another

The bridge and the driver communicate over TCP, so the app (with the dongle) can run on a
different machine from SteamVR. This setup is not tested yet.

- On the app's machine, start the app with `BRIDGE_BIND=0.0.0.0 rebocap`.
- On the SteamVR machine, run `rebocap-config bridge_host <address of the app's machine>`.

The connection has no authentication and no encryption. Use it only on a network you trust.


#### Bridge under Bottles / Wine with ntsync

`bridge.exe` also works on its own, without SteamVR. For example,
[rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge) uses it to feed WiVRn. It was tested with
Rebocap in a Flatpak Bottles bottle (runner soda-11.0-10, ntsync enabled). That Wine build has problems the
bridge now avoids:

- Blocking socket calls fail. `accept` and `recv` return `WSA_IO_PENDING` and `select` returns -1. Sockets are
  now non-blocking, and waits use `WSAEventSelect` events.
- An overlapped pipe read can report success with 0 bytes before Wine fills in the I/O status, which drops the
  message. A read that stays pending may never get its status. The bridge now reads only after `PeekNamedPipe`
  reports data, and waits for the final status.
- Killing a Wine process from the host (`kill`) crashed wineserver, which took Rebocap down with it. To stop the
  bridge, create a `bridge.quit` file next to `bridge.exe`. The bridge deletes the file and exits.

Build it on its own with:

```
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32
```

Start it in the same bottle as Rebocap, while Rebocap is running:

```
flatpak run --command=bottles-cli com.usebottles.bottles run -b <bottle> -e "<path>/bridge.exe"
```

Or add it to the bottle as a program in the Bottles GUI.

Reverse engineered, analyzed, and written with some LLM assistance.
All functionality and code was verified and tested by a human with the actual hardware.
