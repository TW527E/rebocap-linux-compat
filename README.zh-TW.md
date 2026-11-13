[简体中文](README.md) · **繁體中文** · [English](README.en.md) · [日本語](README.ja.md)

# 在 Linux SteamVR 上使用 Rebocap

這個專案讓 Rebocap 全身追蹤在 Wine 下運作，並接上原生的 Linux SteamVR。

> [!TIP]
> Rebocap 已經推出**官方 Linux 版**，內建原生的 Linux SteamVR 驅動。如果你用的是 SteamVR，
> 直接用官方 Linux 版比較簡單，不再需要這個專案。
> 不用 SteamVR、改用 WiVRn 的話，請看
> [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt)，它同時支援官方 Linux 版和 Wine 方案。

本 repo fork 自 [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat)，
修正了 `bridge.exe` 在 Bottles／ntsync 下的問題，並加上用 `bridge.quit` 安全結束的方式（詳見「在 Bottles／有 ntsync 的 Wine 下使用 bridge」）。
編譯好的 `bridge.exe` 可以到 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下載。

這個 fork 只改了 `bridge/bridge.c`。我自己只用裡面的 `bridge.exe` 搭配 WiVRn，用不到 `install.py`、SteamVR 驅動和 SteamVR；
下面 SteamVR 驅動的安裝和使用說明來自原專案，這個 fork 沒有使用也沒有測試。

## 運作方式

Rebocap 的 Windows 安裝包分成三個部分：

- GUI 程式：負責跟 USB 接收器溝通，並做所有的姿態解算
- SteamVR 驅動 DLL
- 兩者之間的 named pipe

GUI 程式不做任何修改，直接在 Wine 下執行。這個專案用一個走相同協定的原生 Linux 驅動，取代原本的 SteamVR DLL：

```
接收器 --/dev/ttyACM*--> rebocap.exe（Wine）<--named pipe--> bridge.exe（Wine）<--TCP--> driver_rebocap.so（vrserver）
```

本 repo 不包含 Rebocap 程式本身的任何檔案，需要自己到 Rebocap 官網下載 Windows 安裝檔。
安裝腳本會解開安裝檔，並從裡面複製驅動需要的圖示和輸入設定檔。通訊協定見 [docs/protocol.md](docs/protocol.md)。

## 需求

- Linux 版 SteamVR、Wine（在 11.15 測試過）、`innoextract`
- 建置工具：`cmake`、支援 C++17 的編譯器、`mingw-w64-gcc`、OpenVR 標頭檔（`openvr` 套件）、`nlohmann-json`。
  `tools/` 和 `driver/test/` 裡的開發工具另外需要 `protoc` 和 Python 的 `protobuf` 套件。
- 一個靜態（非可變）的無襯線字型，例如 Noto Sans。Wine 顯示可變字型的效果很差。
- 開啟接收器序列埠的權限：你的使用者必須在擁有 `/dev/ttyACM*` 的群組裡（Arch 系通常是 `uucp`，Debian 系通常是 `dialout`）。

## 安裝

先關掉 SteamVR，再執行：

```
./install.py /path/to/rebocap_release_vXX.exe
```

腳本會把工具裝到 `~/.local/share/rebocap-linux`（可以用 `--home DIR` 改位置），
向 SteamVR 註冊驅動，並建立 `rebocap` 指令和桌面捷徑。

- **拉了新的更新之後：** 不加參數執行 `./install.py`。
  會重新跑一次 Linux 這邊的完整安裝流程，不會動到 Rebocap 本體的安裝。
- **廠商推出新版時：** 用新的安裝檔執行 `./install.py`。
  會取代程式檔案，保留程式產生的資料（裝置資訊、錄製內容）。
- **解除安裝：** `./install.py --uninstall` 會取消註冊驅動並刪除捷徑，不會動到 Rebocap 本體的安裝。

驅動的設定存在 SteamVR 的設定檔裡。

> [!WARNING]
> **為了安全，追蹤器和接收器的韌體更新請只在 Windows（或官方 Linux 版）上做，不要在 Wine 下更新。**

## 使用方式

1. 插上接收器，執行 `rebocap`，它會同時啟動程式和 bridge。
2. 啟動 SteamVR，連上頭顯。
3. 在 Rebocap 選 VR 模式。驅動連上後，SteamVR 的指示燈會變綠。
   戴著頭顯校準，追蹤器就會出現在 SteamVR 裡（跟 Windows 上一樣）。

程式和 SteamVR 的啟動、重啟順序不限。

PC 模式下程式不會送任何資料給 SteamVR，Windows 上也一樣。

### 原地踏步行走（Walk-in-place）

支援原地踏步行走。在程式的「Walk-In-Place & Swap Pos」區塊啟用「Walk-In-Place」。
驅動會把走路速度寫到「Ctrl Swap」所選手把的搖桿上。
走路時推真實的搖桿可以決定方向；停下來之後，真實的搖桿就恢復正常。

預設情況下驅動把搖桿往前推，往哪個方向走由遊戲決定。
想往髖部面對的方向走，先關掉 SteamVR，再執行：

```
rebocap-config wip_direction waist
rebocap-config wip_game_reference head         # 或 controller：適用於往手指方向移動的遊戲
```

`wip_game_reference` 必須和遊戲解讀搖桿的方式一致。
設錯的話，你會斜著走，而且角度會隨著轉頭或轉手改變。

### 設定

`rebocap-config` 可以查看和修改 steamvr.vrsettings 裡的設定。

| 鍵 | 預設值 | 意義 |
|---|---|---|
| `bridge_host`、`bridge_port` | `127.0.0.1`、`36850` | `bridge.exe` 監聽的位址 |
| `input_hooks` | `true` | 讓原地踏步行走和手部姿勢替換可以改動手把的輸入與姿勢 |
| `wip_direction` | `game` | `game` 或 `waist` |
| `wip_game_reference` | `head` | `head` 或 `controller`，只在 `waist` 時使用 |
| `universe_fallback` | `true` | 找不到 SteamVR 的遊玩區時，改用原始追蹤空間 |

### 程式和 SteamVR 在不同電腦上

bridge 和驅動之間走 TCP，所以程式（接著接收器）可以和 SteamVR 跑在不同電腦上。這種用法還沒測試過。

- 在執行程式的電腦上，用 `BRIDGE_BIND=0.0.0.0 rebocap` 啟動程式。
- 在執行 SteamVR 的電腦上，執行 `rebocap-config bridge_host <程式那台電腦的位址>`。

這個連線沒有驗證也沒有加密，請只在信任的網路裡使用。

### 在 Bottles／有 ntsync 的 Wine 下使用 bridge

`bridge.exe` 也能單獨使用，不需要 SteamVR。例如
[rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge)（fork 自 [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge)）就用它把資料送進 WiVRn。
本 fork 在 Flatpak Bottles 裡的 Rebocap 上測試過（runner soda-11.0-10，啟用 ntsync）。這個 Wine 版本有以下問題，bridge 現在都會避開：

- 阻塞式 socket 呼叫會失敗：`accept` 和 `recv` 回傳 `WSA_IO_PENDING`，`select` 回傳 -1。
  現在 socket 改成非阻塞，等待改用 `WSAEventSelect` 事件。
- 用 overlapped 方式讀 pipe 時，可能在 Wine 填好 I/O 狀態之前就回報成功、讀到 0 bytes，訊息因此遺失；
  一直卡在等待的讀取也可能永遠拿不到狀態。現在 bridge 只在 `PeekNamedPipe` 回報有資料時才讀，並等待最終狀態。
- 從主機端 `kill` Wine 程序會讓 wineserver 當掉，Rebocap 也跟著掛掉。要停止 bridge，
  請在 `bridge.exe` 旁邊建立 `bridge.quit` 檔案，bridge 會刪掉這個檔案並自行結束。

單獨建置：

```
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32
```

也可以直接到 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下載 `bridge.exe`。

在 Rebocap 執行時，在同一個 bottle 裡啟動它：

```
flatpak run --command=bottles-cli com.usebottles.bottles run -b <bottle> -e "<path>/bridge.exe"
```

或在 Bottles 介面裡把它加成這個 bottle 的程式。

`bridge.exe` 的參數是 `bridge.exe [綁定位址] [連接埠]`，預設是 `127.0.0.1 36850`。

## 相關專案

| 專案 | 用途 |
|---|---|
| [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt) | Linux + WiVRn + VRChat 全身追蹤的完整教學與啟動腳本 |
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | 把 Rebocap 的追蹤器透過 SolarXR 送進 WiVRn 的 CLI |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi 補丁，讓 Wine 裡的 Rebocap 認得接收器 |

---

本專案透過逆向工程與分析完成，撰寫過程中使用了部分 LLM 輔助。
所有功能和程式碼都由人在實際硬體上驗證和測試過。
