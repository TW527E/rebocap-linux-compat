**简体中文** · [繁體中文](README.zh-TW.md) · [English](README.en.md) · [日本語](README.ja.md)

# 在 Linux SteamVR 上使用 Rebocap

本项目让 Rebocap 全身追踪在 Wine 下运行，并接入原生 Linux SteamVR。

> [!TIP]
> Rebocap 已经推出**官方 Linux 版**，自带原生 Linux SteamVR 驱动。如果你用的是 SteamVR，
> 建议直接用官方 Linux 版，不再需要本项目。
> 不用 SteamVR、改用 WiVRn 的话，请看
> [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt)，它同时支持官方 Linux 版和 Wine 方案。

本仓库 fork 自 [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat)，
修正了 `bridge.exe` 在 Bottles／ntsync 下的问题，并加入 `bridge.quit` 安全退出方式（详见「在 Bottles／带 ntsync 的 Wine 下使用 bridge」）。
编译好的 `bridge.exe` 可以从 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下载。

本 fork 只修改了 `bridge/bridge.c`。我自己只用其中的 `bridge.exe` 搭配 WiVRn，用不到 `install.py`、SteamVR 驱动和 SteamVR；
下面 SteamVR 驱动的安装和使用说明来自原项目，本 fork 没有使用也没有测试。

## 工作原理

Rebocap 的 Windows 安装包分成三部分：

- GUI 程序：负责和 USB 接收器通信，并完成全部姿态解算
- SteamVR 驱动 DLL
- 两者之间的 named pipe

GUI 程序不经任何修改直接在 Wine 下运行。本项目用一个使用相同协议的原生 Linux 驱动，取代原本的 SteamVR DLL：

```
接收器 --/dev/ttyACM*--> rebocap.exe（Wine）<--named pipe--> bridge.exe（Wine）<--TCP--> driver_rebocap.so（vrserver）
```

本仓库不包含 Rebocap 程序本身的任何文件，你需要自行从 Rebocap 官网下载 Windows 安装包。
安装脚本会解包安装包，并从中复制驱动需要的图标和输入配置文件。通信协议见 [docs/protocol.md](docs/protocol.md)。

## 需求

- Linux 版 SteamVR、Wine（已在 11.15 测试）、`innoextract`
- 构建工具：`cmake`、支持 C++17 的编译器、`mingw-w64-gcc`、OpenVR 头文件（`openvr` 软件包）、`nlohmann-json`。
  `tools/` 和 `driver/test/` 里的开发工具另外需要 `protoc` 和 Python 的 `protobuf` 包。
- 一个静态（非可变）的无衬线字体，例如 Noto Sans。Wine 渲染可变字体效果很差。
- 打开接收器串口的权限：你的用户必须在拥有 `/dev/ttyACM*` 的组里（Arch 系通常是 `uucp`，Debian 系通常是 `dialout`）。

## 安装

先关闭 SteamVR，然后运行：

```
./install.py /path/to/rebocap_release_vXX.exe
```

脚本会把工具安装到 `~/.local/share/rebocap-linux`（可以用 `--home DIR` 修改），
向 SteamVR 注册驱动，并创建 `rebocap` 命令和桌面启动项。

- **拉取更新后：** 不带参数运行 `./install.py`。
  会重新执行 Linux 这一侧的完整安装流程，不会改动 Rebocap 本体的安装。
- **厂商发布新版本时：** 用新的安装包运行 `./install.py`。
  会替换程序文件，保留程序产生的数据（设备信息、录制内容）。
- **卸载：** `./install.py --uninstall` 会注销驱动并删除启动项，不会动 Rebocap 本体的安装。

驱动设置保存在 SteamVR 的设置文件里。

> [!WARNING]
> **为了安全，追踪器和接收器的固件更新请只在 Windows（或官方 Linux 版）上进行，不要在 Wine 下更新。**

## 使用

1. 插上接收器并运行 `rebocap`，它会同时启动程序和 bridge。
2. 启动 SteamVR 并连接头显。
3. 在 Rebocap 里选择 VR 模式。驱动连上之后，SteamVR 的指示灯会变绿。
   戴着头显校准，追踪器就会出现在 SteamVR 里（和 Windows 上一样）。

程序和 SteamVR 的启动、重启顺序不限。

PC 模式下程序不会向 SteamVR 发送任何数据，Windows 上也是如此。

### 原地踏步行走（Walk-in-place）

支持原地踏步行走。在程序的「Walk-In-Place & Swap Pos」区域启用「Walk-In-Place」。
驱动会把行走速度写到「Ctrl Swap」所选手柄的摇杆上。
走动时推动真实摇杆可以决定方向；停止走动后，真实摇杆恢复正常。

默认情况下驱动把摇杆往前推，往哪个方向走由游戏决定。
想朝髋部面向的方向走，先关闭 SteamVR，再运行：

```
rebocap-config wip_direction waist
rebocap-config wip_game_reference head         # 或 controller：适用于朝手指方向移动的游戏
```

`wip_game_reference` 必须和游戏解读摇杆的方式一致。
设错的话，你会斜着走，而且角度会随着转头或转手改变。

### 设置

`rebocap-config` 可以查看和修改 steamvr.vrsettings 里的设置。

| 键 | 默认值 | 含义 |
|---|---|---|
| `bridge_host`、`bridge_port` | `127.0.0.1`、`36850` | `bridge.exe` 监听的地址 |
| `input_hooks` | `true` | 允许原地踏步行走和手部姿势替换修改手柄的输入与姿势 |
| `wip_direction` | `game` | `game` 或 `waist` |
| `wip_game_reference` | `head` | `head` 或 `controller`，只在 `waist` 时使用 |
| `universe_fallback` | `true` | 找不到 SteamVR 的游玩区时，改用原始追踪空间 |

### 程序和 SteamVR 在不同电脑上

bridge 和驱动之间走 TCP，所以程序（连着接收器）可以和 SteamVR 跑在不同电脑上。这种用法尚未测试。

- 在运行程序的电脑上，用 `BRIDGE_BIND=0.0.0.0 rebocap` 启动程序。
- 在运行 SteamVR 的电脑上，运行 `rebocap-config bridge_host <程序所在电脑的地址>`。

这个连接没有认证也没有加密，请只在信任的网络里使用。

### 在 Bottles／带 ntsync 的 Wine 下使用 bridge

`bridge.exe` 也能单独使用，不需要 SteamVR。例如
[rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge)（fork 自 [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge)）就用它把数据送进 WiVRn。
本 fork 在 Flatpak Bottles 里的 Rebocap 上测试过（runner soda-11.0-10，启用 ntsync）。这个 Wine 版本有以下问题，bridge 现在都会避开：

- 阻塞式 socket 调用会失败：`accept` 和 `recv` 返回 `WSA_IO_PENDING`，`select` 返回 -1。
  现在 socket 改为非阻塞，等待改用 `WSAEventSelect` 事件。
- overlapped 方式读 pipe 时，可能在 Wine 填好 I/O 状态之前就回报成功、读到 0 字节，导致消息丢失；
  一直挂起的读取也可能永远拿不到状态。现在 bridge 只在 `PeekNamedPipe` 回报有数据时才读取，并等待最终状态。
- 从主机端 `kill` Wine 进程会让 wineserver 崩溃，连带 Rebocap 一起挂掉。要停止 bridge，
  请在 `bridge.exe` 旁边创建 `bridge.quit` 文件，bridge 会删除这个文件并自行退出。

单独构建：

```
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32
```

也可以直接从 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下载 `bridge.exe`。

在 Rebocap 运行时，在同一个 bottle 里启动它：

```
flatpak run --command=bottles-cli com.usebottles.bottles run -b <bottle> -e "<path>/bridge.exe"
```

或者在 Bottles 界面里把它加成这个 bottle 的程序。

`bridge.exe` 支持参数 `bridge.exe [绑定地址] [端口]`，默认是 `127.0.0.1 36850`。

## 相关项目

| 项目 | 用途 |
|---|---|
| [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt) | Linux + WiVRn + VRChat 全身追踪的完整教程与启动脚本 |
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | 把 Rebocap 的追踪器经 SolarXR 送进 WiVRn 的 CLI |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi 补丁，让 Wine 里的 Rebocap 识别接收器 |

---

本项目经逆向工程与分析完成，编写过程中使用了部分 LLM 辅助。
所有功能和代码都由人在实际硬件上验证和测试过。
