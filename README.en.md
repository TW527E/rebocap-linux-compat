[简体中文](README.md) · [繁體中文](README.zh-TW.md) · **English** · [日本語](README.ja.md)

# Rebocap on Linux SteamVR

This project makes Rebocap full-body tracking work under Wine and connects it to native Linux SteamVR.

> [!TIP]
> Rebocap now has an **official Linux version** with its own native Linux SteamVR driver. If you use SteamVR,
> the official Linux version is the simpler choice and you no longer need this project.
> For WiVRn instead of SteamVR, see
> [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt), which supports both the
> official Linux version and the Wine setup.

This repository is a fork of [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat).
It fixes `bridge.exe` under Bottles / ntsync and adds a safe way to stop it with `bridge.quit`
(see "Bridge under Bottles / Wine with ntsync"). A prebuilt `bridge.exe` is on the
[Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) page.

This fork changes only `bridge/bridge.c`. I use only `bridge.exe`, with WiVRn, so I don't need `install.py`, the
SteamVR driver or SteamVR. The SteamVR driver instructions below come from upstream and are not used or tested in this fork.

## How it works

Rebocap's Windows package has three parts:

- the GUI app, which talks to the USB dongle and does all pose estimation
- a SteamVR driver DLL
- a named pipe between them

The app runs under Wine without any changes. This project replaces the app's SteamVR DLL with a native driver
that uses the same protocol:

```
dongle --/dev/ttyACM*--> rebocap.exe (Wine) <--named pipe--> bridge.exe (Wine) <--TCP--> driver_rebocap.so (vrserver)
```

The repository doesn't contain any files from the application itself. You'll need to download the Windows
installer from Rebocap. The install script unpacks it and copies the driver's icons and input profiles from it.
The protocol is documented in [docs/protocol.md](docs/protocol.md).

## Requirements

- SteamVR for Linux, Wine (tested with 11.15), `innoextract`
- Build tools: `cmake`, a C++17 compiler, `mingw-w64-gcc`, OpenVR headers (`openvr` package), `nlohmann-json`.
  The development tools in `tools/` and `driver/test/` also need `protoc` and the Python `protobuf` package.
- A static (non-variable) sans-serif font such as Noto Sans, as Wine renders variable fonts badly.
- Permission to open the dongle's serial port. Your user must be in the group that owns `/dev/ttyACM*`
  (usually `uucp` on Arch-based systems, `dialout` on Debian-based systems).

## Install

Close SteamVR and run:

```
./install.py /path/to/rebocap_release_vXX.exe
```

The script installs the utils into `~/.local/share/rebocap-linux` (change with `--home DIR` if desired),
registers the driver with SteamVR, and creates a `rebocap` command with a desktop entry.

- **Update after pulling changes:** run `./install.py` with no arguments.
  This redoes the entire install process for the Linux side without modifying the Rebocap install.
- **Newer vendor release:** run `./install.py` with the new installer.
  It replaces the app's program files and keeps the data that the app created (device info, recordings).
- **Uninstall:** `./install.py --uninstall` unregisters the driver and removes the launchers. It doesn't touch the Rebocap install.

Driver settings are stored in SteamVR's settings file.

> [!WARNING]
> **For safety, update tracker and receiver firmware only on Windows (or with the official Linux version). Do not do it under Wine.**

## Use

1. Plug in the dongle and run `rebocap`. This starts the app and the bridge.
2. Start SteamVR and connect the headset.
3. Choose VR mode in Rebocap. The SteamVR indicator turns green when the driver is connected.
   Calibrate with the headset on; the trackers appear in SteamVR, the same as on Windows.

You can start and restart the app and SteamVR in any order.

In PC mode the app sends nothing to SteamVR. The app behaves the same on Windows.

### Walk-in-place

Walk-in-place is supported. Enable "Walk-In-Place" in the app's "Walk-In-Place & Swap Pos" section.
The driver sets the walking speed on the thumbstick of the hand controller selected under "Ctrl Swap".
If you push the real stick while walking, it sets the direction. When you stop walking, the real stick works normally again.

By default the driver pushes the stick forward, and the game decides which direction that is.
To walk in the direction your hips face, close SteamVR and run:

```
rebocap-config wip_direction waist
rebocap-config wip_game_reference head         # or: controller, for games that move you where the hand points
```

`wip_game_reference` must match how the game interprets the stick.
If it is wrong, you'll walk at an angle that changes when you turn your head or hand.

### Settings

`rebocap-config` shows and changes the driver's settings in steamvr.vrsettings.

| key | default | meaning |
|---|---|---|
| `bridge_host`, `bridge_port` | `127.0.0.1`, `36850` | where `bridge.exe` listens |
| `input_hooks` | `true` | lets walk-in-place and hand pose replacement change the hand controllers' input and poses |
| `wip_direction` | `game` | `game` or `waist` |
| `wip_game_reference` | `head` | `head` or `controller`. Used only with `waist`. |
| `universe_fallback` | `true` | use raw tracking space if SteamVR's play space cannot be found |

### App on one machine, SteamVR on another

The bridge and the driver communicate over TCP, so the app (with the dongle) can run on a
different machine from SteamVR. This setup is not tested yet.

- On the app's machine, start the app with `BRIDGE_BIND=0.0.0.0 rebocap`.
- On the SteamVR machine, run `rebocap-config bridge_host <address of the app's machine>`.

The connection has no authentication and no encryption. Use it only on a network you trust.

### Bridge under Bottles / Wine with ntsync

`bridge.exe` also works on its own, without SteamVR. For example,
[rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) (a fork of
[akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge)) uses it to feed WiVRn.
This fork was tested with Rebocap in a Flatpak Bottles bottle (runner soda-11.0-10, ntsync enabled).
That Wine build has problems the bridge now avoids:

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

Or download `bridge.exe` from [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest).

Start it in the same bottle as Rebocap, while Rebocap is running:

```
flatpak run --command=bottles-cli com.usebottles.bottles run -b <bottle> -e "<path>/bridge.exe"
```

Or add it to the bottle as a program in the Bottles GUI.

`bridge.exe` takes `bridge.exe [bind_addr] [port]`; the defaults are `127.0.0.1 36850`.

## Related projects

| Project | Purpose |
|---|---|
| [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt) | Complete guide and launcher for Linux + WiVRn + VRChat full-body tracking |
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | CLI that sends Rebocap's trackers into WiVRn through SolarXR |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi patch so Rebocap in Wine recognizes the receiver |

---

Reverse engineered, analyzed, and written with some LLM assistance.
All functionality and code was verified and tested by a human with the actual hardware.
