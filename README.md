# ret-sync

**ret-sync** は Reverse-Engineering Tools SYNChronization の略で、デバッガー
（WinDbg/GDB/LLDB/OllyDbg/OllyDbg2/x64dbg）と逆アセンブラー
（IDA/Ghidra/Binary Ninja）を同期するプラグイン集です。静的解析と動的解析、
それぞれの長所を同時に利用することを目的としています。

デバッガーと動的解析には、レジスターやメモリーなどの実行時コンテキストと、
WinDbg の `!peb`、`!drvobj`、`!address` などの専用機能があります。一方、
逆アセンブラーと静的解析には、モジュール全体の俯瞰、コード解析、シグネチャー、
型、グラフ表示、デコンパイル、解析結果の永続化といった利点があります。

主な機能は次のとおりです。

* デバッガーの状態に合わせてグラフ表示とデコンパイル表示を同期
* ASLR を意識せずに利用可能（アドレスは実行時にリベース）
* コメントやコマンド出力をデバッガーから逆アセンブラーへ転送
* 複数の IDB/GPR を同時に同期し、複数モジュール間を容易に追跡
* デバッガーと逆アセンブラーを別ホストや仮想マシン上で実行可能

**ret-sync** は [qb-sync](https://github.com/quarkslab/qb-sync) のフォークです。

---

## 目次

* [リポジトリ構成](#リポジトリ構成)
* [前提条件](#前提条件)
* [バイナリリリース](#バイナリリリース)
* [設定](#設定)
* [インストール](#インストール)
  * [IDA](#ida)
  * [Ghidra](#ghidra)
  * [Binary Ninja](#binary-ninja)
  * [WinDbg](#windbg)
  * [GDB](#gdb)
  * [LLDB](#lldb)
  * [OllyDbg と x64dbg](#ollydbg-と-x64dbg)
* [使い方](#使い方)
* [Python ライブラリー](#python-ライブラリー)
* [既知の問題と制限](#既知の問題と制限)
* [ライセンス](#ライセンス)

## リポジトリ構成

デバッガー側のプラグイン：

* `ext_windbg/sync`：WinDbg 拡張のソース（ビルド後は `sync.dll`）
* `ext_gdb/sync.py`：GDB プラグイン
* `ext_lldb/sync.py`：LLDB プラグイン
* `ext_olly1`：OllyDbg 1.10 プラグイン
* `ext_olly2`：OllyDbg 2 プラグイン
* `ext_x64dbg`：x64dbg プラグイン

逆アセンブラー側のプラグイン：

* `ext_ida/SyncPlugin.py`：IDA プラグイン
* `ext_ghidra`：Ghidra プラグイン
* `ext_bn/retsync`：Binary Ninja プラグイン

単体利用向けライブラリー：

* `ext_lib/sync.py`：Python ライブラリー

## 前提条件

IDA および GDB プラグインには、有効な Python 環境が必要です。Python 2.7
以降と Python 3 に対応しています。各製品の対応状況については
[既知の問題と制限](#既知の問題と制限)も参照してください。

## バイナリリリース

WinDbg/OllyDbg/OllyDbg2/x64dbg 向けのビルド済みバイナリは、
[Azure DevOps のパイプライン](https://dev.azure.com/bootlegdev/ret-sync-release/_build/latest/ret-sync-release-CI?definitionId=8?branchName=master)
から取得できます。最新ビルドを選択し、`Related` セクションの成果物を確認してください。

![パイプラインの成果物](img/pipeline.png)

Ghidra 用のビルド済み ZIP は `ext_ghidra/dist` にも収録されています。
ファイル名に記載されたバージョンの Ghidra でのみ使用してください。

## 設定

一般的な構成（デバッガーと逆アセンブラーが同じホストにあり、モジュール名が一致する構成）
では、設定なしで動作します。必要に応じて、ユーザーのホームディレクトリーに `.sync`
という INI 形式の設定ファイルを作成できます。IDA と Ghidra は、IDB または Ghidra
プロジェクトのディレクトリーにある `.sync` を先に読み込みます。ローカル設定がある場合、
グローバル設定は無視されます。`.sync` は自動では作成されません。

### リモートデバッグ

逆アセンブラー側とデバッガー側の両方に、到達可能な実 IP アドレスと同じポートを設定します。
Ghidra 側では `.sync` をホームディレクトリーに置きます。

```ini
[INTERFACE]
host=192.168.128.1
port=9234
```

`0.0.0.0` は使用しないでください。この値は待ち受けと接続の双方に使われるため、
接続時に予期しないエラーとなります。

### モジュール名の別名

解析ファイル名と、デバッガーが認識する実際のモジュール名が異なる場合は、別名を設定します。

```ini
[ALIASES]
ntoskrnl_vuln.exe=ntkrnlmp.exe
```

### Qt Creator で GDB を使用する場合

Qt Creator が GDB の出力方法を変更するため、一時ファイルではなく生の出力を使用します。

```ini
[GENERAL]
use_tmp_logging_file=false
```

### PID やメモリーマップを取得できない環境

シリアル接続の組み込み機器や QEMU 上の raw ファームウェアなど、
`/proc/<pid>/maps` を取得できない場合は初期コンテキストを指定できます。

```ini
[INIT]
context = {
      "pid": 200,
      "mappings": [ [0x400000, 0x7A81158, 0x7681158, "asav941-200.qcow2|lina"] ]
  }
```

各マッピングは `mem_base`、`mem_end`、`mem_size`、`mem_name` の順です。

### Ghidra で自動リベースを無効にする

動的に生成されたコードなどを raw アドレスで扱う場合は、次の Ghidra 専用設定を使用します。

```ini
[GENERAL]
use_raw_addr=true
```

## インストール

### IDA

#### 前提条件と導入

IDA 9.2 以降が必要です。それ以前の IDA では、`ida9.2` タグより前のリビジョンを
使用してください。`ext_ida` 内の `SyncPlugin.py` と `retsync` ディレクトリーを、
次のような IDA のプラグインディレクトリーへコピーします。

* `C:\Program Files\IDA Pro 9.2\plugins`
* `%APPDATA%\Hex-Rays\IDA Pro\plugins`
* `~/.idapro/plugins`

IDB を開き、`Alt+Shift+S` または **Edit → Plugins → ret-sync** から起動します。

#### トラブルシューティング

`retsync/rsconfig.py` の次の値でログを調整できます。

```python
LOG_LEVEL = logging.INFO
LOG_TO_FILE_ENABLE = False
```

`LOG_LEVEL` を `logging.DEBUG` にすると詳細ログを出力します。
`LOG_TO_FILE_ENABLE` を `True` にすると、`broker.py` と `dispatcher.py` の例外を
`%TMP%` 内の `retsync.%s.err` 形式のファイルへ記録します。

### Ghidra

#### Ghidra 拡張のビルド

ビルド済みの `ext_ghidra/dist/ghidra_*_retsync.zip` を使うか、対象の Ghidra
に合わせて以下の手順でビルドします。拡張 ZIP はビルドに使った Ghidra の
バージョン専用です（例：`ghidra_11.4.2_PUBLIC_20250929_retsync.zip`）。

1. 対象バージョンの Ghidra をインストールまたは展開します。
2. Ghidra が必要とする JDK を用意し、必要なら `JAVA_HOME` を設定します。
3. `$GHIDRA_DIR/Ghidra/application.properties` の
   `application.gradle.version` を確認し、そこに指定されたバージョンの Gradle を用意します。
   ディストリビューションの古い Gradle を無条件に使うのではなく、対象 Ghidra の指定を優先してください。
4. リポジトリのルートから次を実行します。

```bash
cd ext_ghidra
export GHIDRA_INSTALL_DIR=/opt/ghidra
gradle clean buildExtension
```

`GHIDRA_INSTALL_DIR` は Ghidra のルートディレクトリー（その直下に `support` と
`Ghidra` がある場所）への絶対パスです。環境変数の代わりに Gradle
プロジェクトプロパティを使うこともできます。

```bash
cd ext_ghidra
gradle clean buildExtension -PGHIDRA_INSTALL_DIR=/opt/ghidra
```

ビルドに成功すると、インストール可能な ZIP が `ext_ghidra/dist` に生成されます。
正確なファイル名には対象 Ghidra のバージョンとリリース日が含まれます。

よくあるビルドエラー：

* `GHIDRA_INSTALL_DIR is not defined!`：環境変数または `-P` オプションを設定してください。
* Gradle の API/クラス互換性エラー：`application.gradle.version` と実行中の
  `gradle --version` が合っているか確認してください。
* Java のバージョンエラー：対象 Ghidra のリリースノートに記載された JDK を使用してください。

#### Ghidra 拡張の導入

1. Ghidra の Project Manager で **File → Install Extensions...** を開きます。
2. `+` を押し、`ext_ghidra/dist/ghidra_*_retsync.zip` を選択して確定します。
3. Ghidra を再起動し、CodeBrowser でモジュールを開きます。
4. 新しいプラグインの検出ダイアログで設定を開き、`RetSyncPlugin` を有効にします。
5. ツールバーまたは `Alt+S`（有効化）、`Alt+Shift+S`（無効化）、`Alt+R`（再起動）を使います。

状態ウィンドウは **Window → RetSyncPlugin** から表示できます。

### Binary Ninja

Binary Ninja 対応は実験的です。解析データベースをバックアップしてから使用してください。
Binary Ninja 2.2 以降と Python 3 が必要です（Python 2 は非対応）。

`ext_bn` の内容を `%APPDATA%\Binary Ninja\plugins` などのプラグインディレクトリーへ
コピーし、Binary Ninja を再起動します。現時点では Plugin Manager から配布していません。

### WinDbg

#### ビルドと導入

`ext_windbg` の Visual Studio ソリューションを使用します。Visual Studio 2017 と
2026 で動作確認済みで、その間のバージョンも動作する見込みです。ビルドすると
`x64\release\sync.dll` が生成されます。

WinDbg Classic では、アーキテクチャに合う拡張ディレクトリーへコピーします。

```text
C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\winext\sync.dll
```

WinDbg Preview では、PATH から検索される場所（例：
`C:\Users\user\AppData\Local\Microsoft\WindowsApps\sync.dll`）に置きます。

対象を開いて `.load sync` を実行し、`!sync` で同期を開始します。Win32 エラー 2 は
DLL の配置、エラー 193 は x86/x64 の不一致を確認してください。Preview で両方を同じ
ディレクトリーに置く場合は、x86 版を `sync32.dll` に改名して `.load sync32` とできます。

### GDB

`ext_gdb/sync.py` を任意の場所へコピーし、GDB から読み込みます。

```text
(gdb) source sync.py
(gdb) sync
```

自動で読み込む場合は、GDB の auto-load script または初期化ファイルを使用してください。

### LLDB

LLDB 対応は実験的です。次のコマンドで読み込みます。`~/.lldbinit` に追加することもできます。

```text
(lldb) command script import sync
(lldb) process launch -s
(lldb) sync
```

### OllyDbg と x64dbg

いずれも実験的対応です。OllyDbg/OllyDbg2 は付属の Visual Studio ソリューションで
ビルドするか、ビルド済み DLL を使い、各製品のプラグインディレクトリーへコピーします。

x64dbg は [testplugin](https://github.com/x64dbg/testplugin) を基にしています。必要に応じて
x64dbg リリースに付属する `pluginsdk` を `ext_x64dbg\x64dbg_sync` へコピーしてビルドし、
生成された `.dp32` または `.dp64` を x64dbg のプラグインディレクトリーへ置きます。

## 使い方

### 共通のデバッガーコマンド

WinDbg ではコマンドの先頭に `!` が必要です（例：GDB の `sync` は WinDbg では
`!sync`）。GDB では `!` を付けません。

| コマンド | 説明 |
|---|---|
| `synchelp` | 利用可能なコマンドと簡単な説明を表示 |
| `sync` / `syncoff` | 同期を開始／停止 |
| `cmt [-a address] <string>` | 現在位置または指定アドレスへコメントを追加 |
| `rcmt [-a address]` | コメントを削除 |
| `fcmt [-a address] <string>` | 現在位置を含む関数へコメントを追加 |
| `raddr <expression>` | 式を評価してリベースしたアドレスをコメントとして追加 |
| `rln <expression>` | 指定アドレスのシンボルを逆アセンブラーから取得 |
| `lbl [-a address] <string>` | ラベルを追加 |
| `cmd <string>` | デバッガーコマンドを実行し、出力をコメントとして追加 |
| `bc [on\|off\|set 0xBBGGRR]` | 経路の色付けを有効化／無効化／色指定 |
| `idblist` | dispatcher に接続された IDB クライアントを表示 |
| `syncmodauto <on\|off>` | モジュール名による解析画面の自動切り替えを変更 |
| `idbn <n>` | n 番目の IDB をアクティブに設定 |
| `jmpto <expression>` | 式を評価し、対応モジュールのリベース済み位置へ移動 |
| `jmpraw <expression>` | リベースや IDB 切り替えをせず raw アドレスへ移動 |
| `translate <base> <addr> <mod>` | モジュール名とオフセットを基準にアドレスをリベース |

`cmt`、`rcmt`、`fcmt` の `-a`/`--address` には、命令の有効な 16 進アドレスを指定します。
引数解析を明示的に終了するには `--` を使用します。

#### WinDbg 固有コマンド

| コマンド | 説明 |
|---|---|
| `curmod` | 現在の命令位置に対応するモジュール情報を表示 |
| `modlist` | IDB の切り替えに適した DML 形式のモジュール一覧 |
| `idb <module name>` | 指定モジュールをアクティブ IDB に設定 |
| `modmap <base> <size> <name>` | 合成モジュールをデバッガー内部の一覧へ追加 |
| `modunmap <base>` | 合成モジュールを削除 |
| `modcheck [md5]` | 現在のモジュールと IDB のファイルが一致するか確認 |
| `bpcmds [save\|load]` | ブレークポイントコマンドを IDB へ保存／復元 |
| `ks` | `kv` を DML のクリック可能なアドレス付きで表示 |

引数なしの `bc` は現在の命令だけを色付けします。これはコードトレーサーではなく、
手動ステップした経路をグラフ上で見やすくする機能です。

#### GDB 固有コマンド

| コマンド | 説明 |
|---|---|
| `bbt` | 逆アセンブラーのシンボルを利用した読みやすいバックトレース |
| `patch` | 実行中のコンテキストを基に逆アセンブラーのバイト列をパッチ |
| `bx` | シンボルを逆アセンブラーで解決して GDB の `x` 相当の表示を実行 |
| `cc` | 逆アセンブラーのカーソル位置まで実行 |

### IDA の操作

`Overwrite idb name` は dispatcher へ登録する IDB 名を変更します。実行ファイルと DLL
の名前が衝突する場合などに使用します。同期中に変更した場合は `Restart` を押して再登録します。

グローバルショートカット：

* `Alt+Shift+S`：ret-sync を起動
* `Ctrl+Shift+S`：同期全体を切り替え
* `Ctrl+H`：Hex-Rays の同期を切り替え

デバッガー操作：

* `F2` / `F3`：通常／一回限りのブレークポイント
* `Ctrl+F2` / `Ctrl+F3`：通常／一回限りのハードウェアブレークポイント
* `Alt+F2`：現在のアドレスをデバッガー側へ変換
* `Alt+F5`：続行、`Ctrl+Alt+F5`：実行（GDB のみ）
* `F10` / `F11`：ステップ／トレース

### Ghidra の操作

`RetSyncPlugin` ウィンドウはドラッグ＆ドロップで CodeBrowser に組み込めます。複数の
モジュールを同じ CodeBrowser で開くと、それらの間を同期できます。

![Ghidra の RetSyncPlugin](img/ghidra.png)

グローバルショートカット：

* `Alt+S`：同期を有効化
* `Alt+Shift+S`：同期を無効化
* `Alt+R`：同期を再起動
* `Alt+Shift+R`：設定を再読み込み

デバッガー操作：

* `F2` / `Alt+F3`：通常／一回限りのブレークポイント
* `Ctrl+F2` / `Ctrl+F3`：通常／一回限りのハードウェアブレークポイント
* `Alt+F2`：現在のアドレスを変換
* `F5`：続行、`Alt+F5`：実行（GDB のみ）
* `F10` / `F11`：ステップ／トレース

### Binary Ninja の操作

`Alt+S` で同期を有効化し、`Alt+Shift+S` で無効化します。ブレークポイントやステップの
ショートカットは、おおむね IDA と同じです。

### OllyDbg と x64dbg の操作

OllyDbg 1.10 は `Alt+S` / `Alt+U`、OllyDbg2 は `Ctrl+S` / `Ctrl+U` で同期を
有効化／無効化します。OllyDbg2 ではグラフ同期、コメント、ラベルのみ実装されています。

x64dbg は Plugins メニューまたは `!sync` / `!syncoff` で同期を切り替えます。
`!synchelp` で使用可能なコマンドを確認できます。

## Python ライブラリー

完全なデバッグ環境がない場合や独自ツールから利用する場合は、`ext_lib/sync.py` を使って
位置同期とシンボル解決を行えます。次はイベント列を順番に同期する例です。

```python
from sync import Sync

HOST = "127.0.0.1"
MAPPINGS = [
    [0x555555400000, 0x555555402000, 0x2000, "/bin/tempfile"],
]
EVENTS = [
    [0x555555400e74, "malloc"],
    [0x555555400eb3, "open"],
    [0x555555400ee8, "exit"],
]

synctool = Sync(HOST, MAPPINGS)
for offset, name in EVENTS:
    synctool.invoke(offset)
    print("0x%08x - %s" % (offset, name))
    input("次のイベントへ進むには Enter を押してください")
```

## 拡張例

ret-sync はデバッガー以外のツールとも統合できます。過去には次の統合例があります。

* [Tetrane REVEN](http://blog.tetrane.com/2015/02/reven-in-your-toolkit.html)
* [EFI DXE Emulator](https://github.com/assafcarlsbad/efi_dxe_emulator)
* [Combining static and dynamic binary analysis - ret-sync](https://www.synacktiv.com/ressources/bieresecu1_ret-sync_en.pdf)

## 既知の問題と制限

* Python 2.7/3.7、IDA 7.7（Windows/Linux/macOS）、Ghidra 10.1.1、
  Binary Ninja 3.0.3225-dev、GDB 8.1.0、LLDB 310.2.37 で動作確認されています。
  現在の対応条件については、各インストール節も確認してください。
* 通信相手の認証も通信の暗号化も行いません。信頼できるネットワーク内で使用してください。
* 自己書き換えコードは対象外です。
* GDB の `return` では停止イベントが呼ばれない場合があり、マルチスレッドデバッグでは
  シグナル処理に問題があります。
* WinDbg では、継続コマンド付きのブレークポイントでも IDA 側が通知を受け、イベント数が
  多いと遅くなる場合があります。その場合は一時的に同期を無効化してください。
* Ghidra のデコンパイラーウィジェットでは、ショートカットが期待どおり動作しない場合があります。
* IDA の大きなグラフは再描画が遅く、Linux ではショートカットが競合する場合があります。
* Logitech Updater が既定ポート `9100` を使うことがあります。その場合は `.sync` で
  別のポートを指定してください。

```ini
[INTERFACE]
host=127.0.0.1
port=9234
```

## ライセンス

**ret-sync** は GNU General Public License バージョン 3、またはそれ以降の
バージョンの条件で再配布・変更できるフリーソフトウェアです。詳細は `COPYING` を参照してください。

Binary Ninja プラグインは MIT License で提供されます。

## 謝辞

本プロジェクトへの助言、フィードバック、バグ報告、コード提供を行ったすべての方々、
ならびに qb-sync と各対応ツールの開発者・コミュニティに感謝します。
