[简体中文](README.md) · [繁體中文](README.zh-TW.md) · [English](README.en.md) · **日本語**

# Linux の SteamVR で Rebocap を使う

このプロジェクトは、Rebocap のフルボディトラッキングを Wine 上で動かし、ネイティブの Linux 版 SteamVR につなぎます。

> [!TIP]
> Rebocap には**公式 Linux 版**が出ており、ネイティブの Linux 用 SteamVR ドライバーが付属しています。SteamVR を使うなら、
> 公式 Linux 版のほうが簡単で、このプロジェクトはもう必要ありません。
> SteamVR ではなく WiVRn を使う場合は
> [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt) を参照してください。公式 Linux 版と Wine の両方に対応しています。

このリポジトリは [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat) のフォークです。
Bottles／ntsync 環境での `bridge.exe` の問題を修正し、`bridge.quit` で安全に終了できるようにしました（「Bottles／ntsync 付き Wine でブリッジを使う」を参照）。
ビルド済みの `bridge.exe` は [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) からダウンロードできます。

このフォークで変更したのは `bridge/bridge.c` だけです。私自身は `bridge.exe` だけを WiVRn と組み合わせて使っているので、`install.py`、SteamVR ドライバー、SteamVR は使いません。
以下の SteamVR ドライバーの導入・使い方は元のプロジェクトのもので、このフォークでは使用もテストもしていません。

## 仕組み

Rebocap の Windows パッケージは 3 つの部分でできています。

- GUI アプリ：USB レシーバーと通信し、姿勢推定をすべて行う
- SteamVR ドライバーの DLL
- 両者をつなぐ名前付きパイプ

GUI アプリは無改造のまま Wine で動きます。このプロジェクトは、同じプロトコルを話すネイティブの Linux ドライバーで、アプリの SteamVR DLL を置き換えます。

```
レシーバー --/dev/ttyACM*--> rebocap.exe（Wine）<--名前付きパイプ--> bridge.exe（Wine）<--TCP--> driver_rebocap.so（vrserver）
```

このリポジトリには Rebocap 本体のファイルは一切含まれていません。Windows 用インストーラーは Rebocap から各自ダウンロードしてください。
インストールスクリプトがそれを展開し、ドライバー用のアイコンと入力プロファイルをコピーします。プロトコルは [docs/protocol.md](docs/protocol.md) にまとめてあります。

## 必要なもの

- Linux 版 SteamVR、Wine（11.15 で確認）、`innoextract`
- ビルドツール：`cmake`、C++17 対応コンパイラー、`mingw-w64-gcc`、OpenVR のヘッダー（`openvr` パッケージ）、`nlohmann-json`。
  `tools/` と `driver/test/` の開発用ツールには、さらに `protoc` と Python の `protobuf` パッケージが必要です。
- Noto Sans などの静的（可変でない）サンセリフフォント。Wine は可変フォントをうまく描画できません。
- レシーバーのシリアルポートを開く権限。ユーザーが `/dev/ttyACM*` を所有するグループに入っている必要があります（Arch 系は通常 `uucp`、Debian 系は通常 `dialout`）。

## インストール

SteamVR を閉じてから実行します。

```
./install.py /path/to/rebocap_release_vXX.exe
```

スクリプトはツールを `~/.local/share/rebocap-linux` にインストールし（`--home DIR` で変更可能）、
ドライバーを SteamVR に登録して、`rebocap` コマンドとデスクトップエントリーを作成します。

- **更新を取り込んだあと：** 引数なしで `./install.py` を実行します。
  Linux 側のインストールをすべてやり直し、Rebocap 本体のインストールには手を付けません。
- **メーカーの新しいリリース：** 新しいインストーラーを指定して `./install.py` を実行します。
  アプリのプログラムファイルを置き換え、アプリが作ったデータ（デバイス情報、録画）は残します。
- **アンインストール：** `./install.py --uninstall` でドライバーの登録を解除し、ランチャーを削除します。Rebocap 本体のインストールはそのままです。

ドライバーの設定は SteamVR の設定ファイルに保存されます。

> [!WARNING]
> **安全のため、トラッカーとレシーバーのファームウェア更新は Windows（または公式 Linux 版）でのみ行い、Wine 上では行わないでください。**

## 使い方

1. レシーバーを挿して `rebocap` を実行します。アプリとブリッジが起動します。
2. SteamVR を起動してヘッドセットを接続します。
3. Rebocap で VR モードを選びます。ドライバーが接続されると SteamVR のインジケーターが緑になります。
   ヘッドセットを着けてキャリブレーションすると、トラッカーが SteamVR に表示されます（Windows と同じです）。

アプリと SteamVR は、どの順番で起動・再起動してもかまいません。

PC モードではアプリは SteamVR に何も送りません。Windows でも同じ動作です。

### その場歩き（Walk-in-place）

その場歩きに対応しています。アプリの「Walk-In-Place & Swap Pos」で「Walk-In-Place」を有効にしてください。
ドライバーは「Ctrl Swap」で選んだコントローラーのスティックに歩行速度を書き込みます。
歩いている間に実際のスティックを倒すと、その方向に進みます。歩くのをやめると、スティックは通常どおりに戻ります。

既定ではドライバーはスティックを前に倒し、どちらへ進むかはゲームが決めます。
腰が向いている方向へ歩きたい場合は、SteamVR を閉じてから次を実行します。

```
rebocap-config wip_direction waist
rebocap-config wip_game_reference head         # または controller：手の向きに移動するゲーム向け
```

`wip_game_reference` は、ゲームがスティックをどう解釈するかに合わせる必要があります。
合っていないと斜めに歩き、その角度が頭や手の向きによって変わります。

### 設定

`rebocap-config` で steamvr.vrsettings の設定を表示・変更できます。

| キー | 既定値 | 意味 |
|---|---|---|
| `bridge_host`, `bridge_port` | `127.0.0.1`, `36850` | `bridge.exe` が待ち受けるアドレス |
| `input_hooks` | `true` | その場歩きと手の姿勢の置き換えが、コントローラーの入力と姿勢を変更できるようにする |
| `wip_direction` | `game` | `game` または `waist` |
| `wip_game_reference` | `head` | `head` または `controller`。`waist` のときだけ使用 |
| `universe_fallback` | `true` | SteamVR のプレイエリアが見つからないとき、生のトラッキング空間を使う |

### アプリと SteamVR を別のマシンで動かす

ブリッジとドライバーは TCP で通信するので、アプリ（レシーバーをつないだマシン）と SteamVR を別のマシンで動かせます。この構成はまだテストしていません。

- アプリ側のマシンで `BRIDGE_BIND=0.0.0.0 rebocap` としてアプリを起動します。
- SteamVR 側のマシンで `rebocap-config bridge_host <アプリ側マシンのアドレス>` を実行します。

この接続には認証も暗号化もありません。信頼できるネットワークでのみ使ってください。

### Bottles／ntsync 付き Wine でブリッジを使う

`bridge.exe` は SteamVR なしでも単体で使えます。たとえば
[rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge)（[akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge) のフォーク）は、これを使って WiVRn にデータを送っています。
このフォークは Flatpak 版 Bottles のボトル内の Rebocap で確認しました（runner soda-11.0-10、ntsync 有効）。この Wine ビルドには次の問題があり、ブリッジはそれを回避しています。

- ブロッキングのソケット呼び出しが失敗します。`accept` と `recv` は `WSA_IO_PENDING` を返し、`select` は -1 を返します。
  ソケットをノンブロッキングにし、待機は `WSAEventSelect` のイベントで行うようにしました。
- overlapped でパイプを読むと、Wine が I/O ステータスを書き込む前に 0 バイトで成功を返すことがあり、メッセージが失われます。
  保留のままの読み取りはステータスを受け取れないこともあります。`PeekNamedPipe` がデータありと報告したときだけ読み、最終ステータスを待つようにしました。
- ホスト側から Wine のプロセスを `kill` すると wineserver がクラッシュし、Rebocap も巻き込まれて落ちます。ブリッジを止めるには、
  `bridge.exe` の隣に `bridge.quit` ファイルを作ってください。ブリッジはそのファイルを削除して終了します。

単体でのビルド：

```
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32
```

[Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) から `bridge.exe` をダウンロードすることもできます。

Rebocap が動いている状態で、同じボトル内で起動します。

```
flatpak run --command=bottles-cli com.usebottles.bottles run -b <bottle> -e "<path>/bridge.exe"
```

または Bottles の GUI で、そのボトルのプログラムとして追加してください。

`bridge.exe` の引数は `bridge.exe [バインドアドレス] [ポート]` で、既定値は `127.0.0.1 36850` です。

## 関連プロジェクト

| プロジェクト | 用途 |
|---|---|
| [TW527E/rebocap-linux-wivrn-fbt](https://github.com/TW527E/rebocap-linux-wivrn-fbt) | Linux + WiVRn + VRChat フルボディトラッキングの手順書と起動スクリプト |
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | Rebocap のトラッカーを SolarXR 経由で WiVRn に送る CLI |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | Wine 上の Rebocap がレシーバーを認識できるようにする setupapi パッチ |

---

リバースエンジニアリングと解析によって作成し、執筆には一部 LLM の支援を受けています。
すべての機能とコードは、実際のハードウェアで人が検証・テストしています。
