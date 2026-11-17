# Aki（あき）実装仕様

このリポジトリには実装（Swift のソース、テスト、ビルドスクリプト）を含めない。
このファイルは、コーディングエージェントが手元で Aki を一から実装し、作者の手元にある実装と同じ挙動を再現するための仕様である。
利用者向けの説明は README.md にある。

## エージェントの作業ルール

- **実装ファイルはコミットしない**：リポジトリで追跡するのは `AGENTS.md`、`README.md`、`LICENSE` だけである。
  生成したファイルは、git なら `.git/info/exclude` に追加して追跡対象から外す。
  jj なら、自動で追跡するファイルをこの 3 つに絞る（値は TOML の文字列として渡すので、外側を二重に引用する）。

  ```bash
  jj config set --repo snapshot.auto-track "'root-file:\"AGENTS.md\" | root-file:\"README.md\" | root-file:\"LICENSE\"'"
  ```
- **仕様の正はこのファイル**：実装の挙動を変えたら、このファイルも同じ変更で更新する。
- **テストは一時ディレクトリだけを触る**：ユニットテストでも手動確認でも、シェルからホームディレクトリ全体をスキャンしない。
  数分かかるうえ、ターミナルに対してプライバシー保護（TCC）の許可ダイアログが次々に出る。
- **インストールと起動はユーザーの了承を得てから行う**：`~/Applications` などから初めて起動するとログイン項目に登録されるので、`make install` や `make run`、アプリの起動はユーザーに確認してから行う。

## 何を作るか

macOS のメニューバーにストレージの空き容量を表示するアプリを作る。
クリックするとポップオーバーが開き、使用量の大まかな内訳と、ホーム以下の大きいフォルダやファイルが表示される。
フォルダは階層をたどって掘り下げられ、Finder で表示できる。
閲覧専用で、削除機能は持たない。

- **アプリ名**：Aki（あき）
- **Bundle ID**：`dev.aki.Aki`
- **対応 OS**：macOS 14 以降（`@Observable` を使うため）
- **言語とツール**：Swift 6（swift-tools-version 6.0）、SwiftPM、SwiftUI、Swift Testing。外部依存は使わない。Xcode プロジェクトは作らない。
- **UI 文言**：すべて日本語。以下に書く文字列をそのまま使う。

## ファイル構成

```
Package.swift
Sources/AkiCore/SizeNode.swift     SizeNode, ScanResult, ScanProgress
Sources/AkiCore/Scanner.swift      ディレクトリ走査
Sources/AkiCore/Volume.swift       VolumeStats, Breakdown, formatBytes
Sources/Aki/main.swift             エントリポイント（隠し CLI を含む）
Sources/Aki/App.swift              MenuBarExtra
Sources/Aki/AppModel.swift         状態、スキャン、キャッシュ、ログイン項目
Sources/Aki/PopoverView.swift      ポップオーバーの UI
Sources/Aki/Snapshot.swift         開発用：ポップオーバーを PNG に書き出す
Tests/AkiCoreTests/ScannerTests.swift
Tests/AkiCoreTests/BreakdownTests.swift
scripts/build-app.sh               .app を組み立てて ad-hoc 署名する
Makefile
.gitignore                         .build/ build/ .swiftpm/（ローカルのみ。コミットしない）
```

`AkiCore` はテスト対象のライブラリで、UI を持たない。
`Aki` は実行ファイルで、`AkiCore` に依存する。

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "Aki",
    platforms: [.macOS(.v14)],
    targets: [
        .target(name: "AkiCore"),
        .executableTarget(name: "Aki", dependencies: ["AkiCore"]),
        .testTarget(name: "AkiCoreTests", dependencies: ["AkiCore"]),
    ]
)
```

## AkiCore

`Aki` は別モジュールなので、AkiCore の型、プロパティ、イニシャライザ、関数は、以下で internal や private と書いたもの以外すべて `public` にする。
public な struct でもメンバーごとのイニシャライザは internal になるので、struct のイニシャライザは `public init` を明示的に書く。
各ファイルの import は次のとおり。

- **SizeNode.swift**：`Foundation`、`os`（`OSAllocatedUnfairLock`）
- **Scanner.swift**：`Darwin`、`Foundation`、`os`
- **Volume.swift**：`Darwin`、`Foundation`
- **main.swift**：`Foundation`、`AkiCore`
- **App.swift**：`SwiftUI`、`AkiCore`
- **AppModel.swift**：`Foundation`、`Observation`、`os`（`Logger`）、`ServiceManagement`、`AkiCore`
- **PopoverView.swift**：`AppKit`、`ServiceManagement`（`openSystemSettingsLoginItems()`）、`SwiftUI`、`AkiCore`
- **Snapshot.swift**：`AppKit`、`AkiCore`、`SwiftUI`

`public enum Scanner` は `Foundation.Scanner` と同じ名前だが、Swift 6.4 では自モジュールの型が優先されるため、`AkiCore.` を付けずに参照できる。

### データ型（SizeNode.swift）

- **`SizeNode`**：`Codable, Sendable, Identifiable, Hashable` の struct。
  プロパティは `name: String`、`path: String`（絶対パス）、`size: Int64`、`isDirectory: Bool`、`children: [SizeNode]` で、すべて `var` にする。
  `id` は `path` を返す。
  `children` はサイズの降順に並べる。
  イニシャライザは `init(name:path:size:isDirectory:children: = [])`。
- **`ScanResult`**：`Codable, Sendable` の struct。
  `root: SizeNode`、`unreadableCount: Int`（権限不足などで中を読めなかったディレクトリ数）、`scannedAt: Date` を持ち、すべて `var` にする。
  イニシャライザは `public init(root:unreadableCount:scannedAt:)`。
- **`ScanProgress`**：`final class` で `Sendable` に準拠させる。
  内部状態（`bytes: Int64`、`currentPath: String`）を、private な struct にまとめて `OSAllocatedUnfairLock(initialState:)` で守る。
  イニシャライザは引数なしの `public init()`。
  読み取り用に、ロックを取って値を返す計算プロパティ `public var bytes: Int64` と `public var currentPath: String` を公開する。
  書き込み用の `func add(_ bytes: Int64)`（0 なら何もしない）と `func setPath(_:)` は internal にする。
  スキャンのスレッドから書き、UI のスレッドから読む。

### 走査（Scanner.swift）

`public enum Scanner` に次の API を置く。

```swift
public static func disableDatalessMaterialization()
public static func scan(_ path: String, threshold: Int64 = 50_000_000,
                        progress: ScanProgress? = nil) async -> ScanResult
public static func scanSync(_ path: String, threshold: Int64 = 50_000_000,
                            progress: ScanProgress? = nil) -> ScanResult
```

`disableDatalessMaterialization()` は、iCloud Drive などで未ダウンロードのファイル（dataless）を読んだときにダウンロードが走らないよう、プロセス全体のポリシーを設定する。
起動直後に一度だけ呼ぶ。

```swift
_ = setiopolicy_np(IOPOL_TYPE_VFS_MATERIALIZE_DATALESS_FILES,
                   IOPOL_SCOPE_PROCESS,
                   IOPOL_MATERIALIZE_DATALESS_FILES_OFF)
```

`scan` は `DispatchQueue.global(qos: .utility)` で `scanSync` を実行し、`withCheckedContinuation` で async に橋渡しする。
`scanSync` は共有コンテキストを作って走査し、`ScanResult(root:unreadableCount:scannedAt: Date())` を返す。

#### サイズの数え方

- サイズは割り当て済みのブロック数で数える（`Int64(st.st_blocks) * 512`）。`du` と同じ基準である。
- 通常ファイル（`S_IFREG`）で `st_nlink > 1` のものは、`(st_dev, st_ino)` の組を共有の `Set` に記録し、2 回目以降は 0 として数える（ハードリンクの重複排除）。
  この `Set` は並列に触られるので `OSAllocatedUnfairLock` で守る。
- ディレクトリ以外（通常ファイル、シンボリックリンクなど）はすべてファイルとして数える。シンボリックリンクはたどらない。
- 読めないディレクトリの数も共有コンテキストにロック付きで数える。
- 共有コンテキストは `private final class Context: Sendable` にする。
  `threshold`、`progress`、`OSAllocatedUnfairLock<Set<InodeKey>>`（`InodeKey` は `dev: Int32` と `ino: UInt64` を持つ Hashable な struct）、`OSAllocatedUnfairLock<Int>`（unreadable の数）を持つ。
  `scanSync` の呼び出しごとに 1 つ作るので、ハードリンクの重複排除は 1 回のスキャンの中でだけ効く。
- `SF_DATALESS` は Int32 として、`st_flags` は UInt32 として取り込まれるので、`st.st_flags & UInt32(SF_DATALESS)` で比べる。
- パスの連結には `(path as NSString).appendingPathComponent(_:)` を、名前の取り出しには `lastPathComponent` を使う。

#### 閾値による枝刈り

`threshold`（既定 50,000,000 バイト）以上のフォルダとファイルだけを `SizeNode` として残す。
それ未満の項目はノードにせず、サイズだけを親に合算する。
したがって、どのノードでも `size` は配下の全項目の合計であり、`children` のサイズの和より大きくなりうる。
スキャンの起点（root）は大きさにかかわらず常にノードとして返す。

#### 走査の手順

上位の階層は自前で列挙して並列化し、深い階層は `fts` で 1 スレッドずつたどる。
並列化する深さは `splitDepth = 4` とする。

1. **起点**：`lstat` に失敗したら `size: 0` のディレクトリノードを返す（unreadable には数えない）。
   起点がディレクトリでなければ、そのファイル 1 つのノードを返し、そのサイズを progress に加える。
   ディレクトリなら、起点の `st_dev` を持って `scanDir(path, dev, depth: splitDepth)` に進む。
2. **`scanDir(path, dev, depth)`**：`FileManager.default.contentsOfDirectory(atPath:)` で直下を列挙する。
   列挙に失敗したら unreadable を 1 増やし、`size: 0` のディレクトリノードを返す。
   各エントリを `lstat` し（失敗したエントリは無視する）、次のように振り分ける。
   - ディレクトリのうち、`st_dev` が起点と異なるもの（別ボリュームのマウントポイント）と、`st_flags` に `SF_DATALESS` が立っているもの（未ダウンロード）は、たどらずに捨てる。
   - それ以外のディレクトリは、後で並列に処理する一覧に入れる。
   - ファイルはサイズを数えて合計し、閾値以上なら `isDirectory: false` のノードにする。

   ファイルの合計は列挙が終わった時点でまとめて progress に加える。
   子ディレクトリは `DispatchQueue.concurrentPerform` で並列に処理する。
   `depth > 1` なら `scanDir(child, dev, depth - 1)` を、そうでなければ `walk(child)` を呼ぶ。
   結果は添字ごとにロック付きの配列 `[SizeNode?]` に入れ、`nil`（`walk` が `fts_open` に失敗した場合）は捨てる。
   自分のサイズは「ファイルの合計＋子ディレクトリのサイズの合計」とする。
   `children` は「閾値以上のファイルのノード＋閾値以上のディレクトリのノード」をサイズの降順に並べたものにする。
3. **`walk(path) -> SizeNode?`**：まず `progress.setPath(path)` を呼ぶ。
   `fts_open` を `FTS_PHYSICAL | FTS_XDEV | FTS_NOCHDIR` で開き、失敗したら `nil` を返す。
   `fts_open` には NULL で終わる C 文字列の配列を渡す。

   ```swift
   let cPath = strdup(path)
   defer { free(cPath) }
   var argv: [UnsafeMutablePointer<CChar>?] = [cPath, nil]
   guard let fts = fts_open(&argv, FTS_PHYSICAL | FTS_XDEV | FTS_NOCHDIR, nil) else { return nil }
   defer { fts_close(fts) }
   ```

   `fts_info` は UInt16、`fts_level` は Int16 で取り込まれるので、`Int32(ent.pointee.fts_info)` を `FTS_*` 定数と比べ、`Int(ent.pointee.fts_level)` を level として使う。
   ディレクトリごとに `Frame { level: Int, size: Int64 = 0, children: [SizeNode] = [] }` をスタックに積み、後行順で組み立てる。
   `fts_read` が返す `fts_info` ごとの処理は次のとおり。

   | `fts_info` | 処理 |
   | --- | --- |
   | `FTS_D` | `fts_level` を持つ Frame を push する。`fts_statp` の `st_flags` に `SF_DATALESS` が立っていれば `fts_set(fts, ent, FTS_SKIP)` で中に入らない（直後に同じエントリの `FTS_DP` が来る） |
   | `FTS_DP` | スタック先頭の level が `fts_level` と一致するときだけ pop する |
   | `FTS_DNR`, `FTS_ERR` | 読めないディレクトリでは `FTS_D` の後、`FTS_DP` の代わりにこれが来る。スタック先頭の level が一致するときだけ、unreadable を 1 増やして pop する |
   | `FTS_F`, `FTS_SL`, `FTS_SLNONE`, `FTS_DEFAULT` | スタックが空でなければ、サイズを数えて先頭 Frame の size に足す。閾値以上ならファイルノードを先頭 Frame の children に足す |
   | それ以外（`FTS_NS`, `FTS_DC` など） | 数えない |

   pop では、先頭 Frame を取り出して次のように扱う。
   - スタックが空になった（起点の）Frame は、閾値にかかわらずノードにして結果とする。
   - 閾値以上の Frame はノードにし（children はサイズ降順）、親 Frame の size と children に足す。
   - 閾値未満の Frame はノードにせず、size だけを親 Frame に足す。

   ノードの `name` は `fts_path` の最後の要素、`path` は `fts_path` にする。
   progress へは 64,000,000 バイトたまるごとに加え、閾値以上の Frame を閉じたときと走査の終了時にも、たまった分を加える。
   これにより、スキャン完了時に `progress.bytes` は起点のサイズと一致する。

### 容量と内訳（Volume.swift）

- **`VolumeStats`**：`Sendable, Equatable` の struct。
  `total`、`available`、`system`（いずれも `Int64` の `var`）と、計算プロパティ `used: Int64 { max(0, total - available) }` を持つ。
  イニシャライザは `public init(total:available:system:)`。
- **`VolumeStats.current() -> VolumeStats?`**：ホームディレクトリの URL の `resourceValues` から `volumeTotalCapacity` と `volumeAvailableCapacityForImportantUsage` を取る。
  後者が取れなければ `volumeAvailableCapacity` を使う。
  `total` が取れなければ `nil` を返す。
  `available` は Finder の「使用可能」と同じ基準で、パージ可能な領域を含む。
- **`system`**：`/`、`/System/Volumes/VM`、`/System/Volumes/Preboot` の各ボリュームの使用量の合計である。
  APFS では `statfs` の `f_blocks` と `f_bfree` がコンテナ全体の値になるので、`statfs` は使わない（使うとシステムが数 TB と表示される）。
  `df` と同じく `getattrlist` の `ATTR_VOL_SPACEUSED` を使う。
  返り値のバッファは、先頭 4 バイトが全体長で、その後に使用量の `off_t` が続く。

```swift
private static func usedBytes(of path: String) -> Int64 {
    var list = attrlist()
    list.bitmapcount = u_short(ATTR_BIT_MAP_COUNT)
    list.volattr = attrgroup_t(ATTR_VOL_INFO) | attrgroup_t(ATTR_VOL_SPACEUSED)
    var buf = [UInt8](repeating: 0, count: 16)
    let r = buf.withUnsafeMutableBytes { getattrlist(path, &list, $0.baseAddress, $0.count, 0) }
    guard r == 0 else { return 0 }
    return buf.withUnsafeBytes { $0.loadUnaligned(fromByteOffset: 4, as: Int64.self) }
}
```

- **`Breakdown`**：`Sendable, Equatable` の struct で、`apps`、`home`、`system`、`other`、`free`（いずれも `Int64` の `var`）を持つ。
  イニシャライザは `public init(apps:home:system:other:free:)`。
  `static func make(volume: VolumeStats, home: Int64?, apps: Int64?) -> Breakdown` は、`nil` を 0 とみなし、`other = max(0, used - home - apps - system)`、`free = available` とする。
  スキャン中や権限不足で合計が使用量を超えても、その他は 0 未満にならない。
- **`public func formatBytes(_ bytes: Int64) -> String`**：`ByteCountFormatter.string(fromByteCount:countStyle: .file)`（10 進）。

## Aki（アプリ）

### エントリポイント（main.swift）

`AkiApp` には `@main` を付けない。
`main.swift` があるターゲットで `@main` を付けるとコンパイルできない。
`main.swift` は次の順に処理する。

1. `Scanner.disableDatalessMaterialization()` を呼ぶ。
2. 引数に `--scan <path>` があれば、UI を出さずに `scanSync`（`~` を展開、threshold は既定の 50MB）を実行して次の形式で出力し、`exit(0)` する。
   `--scan` の後に引数がなければ、このオプションは無視する（`--snapshot` も同様）。
   所要時間は `Date` の差で測る。
   1 行目は `total_kb=<size/1024>\t<formatBytes>\tunreadable=<数>\telapsed=<秒、小数 1 桁>s` で、続けて上位 10 件の子を `  <formatBytes>\t<name>` の形で 1 行ずつ出す。
   `du -skx` との突き合わせに使う。
3. 引数に `--snapshot <out.png> [ホームからの相対パス]` があれば、`renderSnapshot(to:drill:)` を呼んで `exit(0)` する。
4. それ以外は `AkiApp.main()` を呼ぶ。

### メニューバー（App.swift）

`struct AkiApp: App` が `@State private var model = AppModel()` を持つ。
シーンは `MenuBarExtra { PopoverView(model: model) } label: { ... }` に `.menuBarExtraStyle(.window)` を付ける。
ラベルは `HStack(spacing: 4)` で、SF Symbol と `Text(model.freeText)` を並べる。
SF Symbol は通常 `internaldrive`、空きが 10% 未満なら `exclamationmark.triangle.fill` にする。

### 状態とスキャン（AppModel.swift）

`@MainActor @Observable final class AppModel` にする。
ログは、ファイルの先頭に置いた `private let log = Logger(subsystem: "dev.aki.Aki", category: "app")` に出す。

- **読み取り専用の状態**（`private(set) var`）：`volume: VolumeStats?`、`home: ScanResult?`、`apps: ScanResult?`、`isScanning`、`progressBytes: Int64`、`progressPath: String`、`launchAtLogin`、`loginItemNeedsApproval`。
- **計算プロパティ**：
  - `breakdown`：`volume` があれば `Breakdown.make(volume:home: home?.root.size, apps: apps?.root.size)`。
  - `freeText`：`formatBytes(volume.available)`、`volume` がなければ `"—"`。
  - `isLow`：`Double(available) / Double(total) < 0.1`（total が 0 以下なら false）。Int64 のまま割ると 0 になるので Double で計算する。
  - `unreadableCount`：home と apps の unreadableCount の和。
- **`init(autoStart: Bool = true)`**：キャッシュを読み込み、`refreshVolume()` とログイン項目の状態の更新を行う。
  `autoStart` が false ならここで終える（スナップショット用）。
  true なら続けて次を行う。
  1. 初回のインストール済み起動ならログイン項目に登録する（後述）。
  2. 60 秒ごとに `refreshVolume()` を呼ぶ `Task` を起動する（`[weak self]`、`while !Task.isCancelled` のループ。Task はどこにも保持しない）。
  3. `home` がない、または `home.scannedAt` が 1 時間（3600 秒）より古ければ `startScan()` を呼ぶ。
- **`refreshVolume()`**：`volume = VolumeStats.current()`。
- **`startScan()`**：実行中なら何もしない（二重起動しない）。
  `isScanning = true` にし、progress の表示を 0 と空文字に戻し、`ScanProgress` を作って `Task` を起動する。
  その中で、0.5 秒ごとに progress を読んで `progressBytes` と `progressPath` に写すポーリング用の `Task` を起動する。
  `async let` でホーム（`NSHomeDirectory()`、progress あり）と `/Applications`（progress なし）を同時にスキャンし、両方を待つ。
  終わったらポーリングを cancel し、`home` と `apps` を更新し、`isScanning = false` にして、`refreshVolume()` とキャッシュの保存を行う。
- **ログイン項目**：`SMAppService.mainApp` を使う。
  - `setLaunchAtLogin(_ on: Bool)`：`register()` か `unregister()` を呼び、失敗はログに出す。その後で状態を更新する。
  - 状態の更新：`status == .enabled` を `launchAtLogin` に、`status == .requiresApproval` を `loginItemNeedsApproval` に入れる。
  - 初回のインストール済み起動：`Bundle.main.bundlePath` が `/Applications/` か `NSHomeDirectory() + "/Applications/"` で始まり、UserDefaults の `loginItemConfigured` が false のときだけ `setLaunchAtLogin(true)` を呼び、`loginItemConfigured` を true にする。
    `build/` から試しに起動した版は登録しない。
    2 回目以降はチェックボックスの設定に従う。
- **キャッシュ**：`struct Cache: Codable { var home: ScanResult?; var apps: ScanResult? }` を、ユーザーの Caches ディレクトリの `dev.aki.Aki/scan.json` に `JSONEncoder` の既定設定で保存する。
  親ディレクトリを作ってから `.atomic` で書く。
  読み込みと書き込みの失敗は、どちらも黙って無視する（`try?`）。

### ポップオーバー（PopoverView.swift）

`struct PopoverView: View` は `model: AppModel` と、ドリルダウンの経路 `@State private var stack: [SizeNode]` を持つ。
イニシャライザは `init(model:initialStack: [SizeNode] = [])`。
表示中のノード `current` は `stack.last ?? model.home?.root` である。

使う定数は次のとおり。

- 1 階層に表示する行数の上限：20
- 行の高さ：30pt
- サイズ列の幅：80pt
- リストの最大の高さ：330pt
- フルディスクアクセスの設定画面の URL：`x-apple.systempreferences:com.apple.preference.security?Privacy_AllFiles`

全体は `VStack(alignment: .leading, spacing: 12)` で、上から「容量と内訳」、`Divider()`、「大きいフォルダ」、`Divider()`、「フッター」を並べる。
`.padding(14)` と `.frame(width: 380)` を付ける。
`onAppear` で `model.refreshVolume()` を呼ぶ。
`onChange(of: model.home?.scannedAt) { stack = [] }`（引数なしのクロージャの形。引数 1 つの形は macOS 14 で非推奨）で `stack` を空に戻す。

#### 容量と内訳

`volume` と `breakdown` があるときは、`VStack(alignment: .leading, spacing: 8)` で次を並べる。
ないときは `Text("容量を取得できませんでした")`（secondary）だけを出す。

1. `HStack(alignment: .firstTextBaseline)`：
   左に `"空き \(formatBytes(available))"` を `.title2.bold()` で出す（`isLow` なら赤、それ以外は primary）。
   `Spacer()` を挟み、右に `"\(formatBytes(total)) 中 \(使用率)% 使用"` を `.caption` の secondary で出す。
   使用率は `Int((Double(used) / Double(max(total, 1)) * 100).rounded())`。Int64 のまま割ると常に 0% になるので、必ず Double で計算する。
2. 使用率バー（高さ 10）。
3. 凡例：`LazyVGrid`（`GridItem(.flexible(), spacing: 16)` と `GridItem(.flexible())` の 2 列、leading 揃え、行間 4）。
   各項目は `HStack(spacing: 6)` に、8pt の円、ラベル、`Spacer()`、値（`monospacedDigit`、secondary）を並べ、`.font(.callout)` にする。
   値が `nil` なら `"…"` を出す。

内訳の項目は次の順に並べる。
アプリ、ホーム、その他の値は、ホームのスキャン結果がまだないとき `nil` にする。

| ラベル | 値 | 色 |
| --- | --- | --- |
| アプリ | `apps` | `.blue` |
| ホーム | `home` | `.orange` |
| システム | `system` | `.gray` |
| その他 | `other` | `.purple` |
| 空き | `free` | `Color.primary.opacity(0.12)` |

使用率バーは `parts: [(Double, Color)]`（各項目の値を Double にしたものと色。`nil` は 0）を受け取る。
`GeometryReader` の中の `HStack(spacing: 1)` に、各項目の色の `Rectangle` を幅 `max(0, geo.size.width * 値 / 合計 - 1)` で並べ、全体を `Capsule` で切り抜く。
合計は値の和で、最小 1 とする。

#### 大きいフォルダ

`VStack(alignment: .leading, spacing: 6)` にする。

見出しの行は `HStack(spacing: 6)` で、次を並べる。
- `stack` が空でなければ、`chevron.left` の戻るボタン（`.borderless`、help は「戻る」）。押すと `stack` の末尾を取り除く。
- 見出しの文字列（`.headline`、1 行、先頭を省略）：`stack` が空なら「ホームの大きいフォルダ」、そうでなければ `current.path` を `~` 表記にしたもの。
- `Spacer()`。
- `current` があれば、`folder` アイコンのボタン（`.borderless`、help は「Finderで表示」）。押すと `current` を Finder で表示する。

`current` があれば、`children` の先頭 20 件を `NodeRow` で並べる。
`current.size` から表示した 20 件のサイズの和を引いた残りが正なら、末尾に `RestRow` を足す。
行は `ScrollView` の中の `VStack(spacing: 0)` に入れ、`ScrollView` の高さは `min(max(行数, 1) * 30, 330)` にする。
`current` がなく、スキャン中なら「初回スキャン中です。数分かかることがあります。」を、スキャン中でなければ「まだスキャンしていません」を、`.callout` の secondary で出す。

**`NodeRow`**（`node`、`parentSize`、`onOpen` を受け取る）は `HStack(spacing: 8)` に次を並べる。
- アイコン（幅 16）：フォルダなら `folder.fill`、ファイルなら `doc.fill`。色は `node.isDirectory ? Color.accentColor : Color.secondary`。
- `VStack(alignment: .leading, spacing: 3)` に名前（1 行、中央を省略）と割合バー（高さ 3、`size / max(parentSize, 1)`）。`.frame(maxWidth: .infinity, alignment: .leading)` を付ける。
- サイズ（`monospacedDigit`、secondary、幅 80、右揃え）。
- `chevron.right`（`.caption`、tertiary）。開けない行では opacity 0 にして幅だけ残す。

行は横の padding 6、高さ 30 にする。
ホバー中は角丸 6 の `Color.primary.opacity(0.08)` を背景に敷き、`contentShape(Rectangle())` で行全体をクリック可能にする。
開ける行（フォルダで、`children` が空でないもの）はクリックで `onOpen`（`stack` に push）を呼び、それ以外はクリックで Finder に表示する。
help は、開ける行なら「クリックで中を表示」、それ以外は「クリックで Finder に表示」。
右クリックメニューには「Finderで表示」と「パスをコピー」を置く。
「パスをコピー」は `NSPasteboard.general` を `clearContents()` してから `setString(path, forType: .string)` する。

**`RestRow`**（`size` と `parentSize` を受け取る）は `NodeRow` と同じレイアウトで、アイコンは `ellipsis.circle`（secondary）、名前は「その他（小さい項目）」（secondary）にする。
シェブロンは opacity 0 で置く。
ホバー表示とクリック動作は持たない。

割合バーは `fraction: Double` を受け取り、`GeometryReader` の中の `ZStack(alignment: .leading)` に、背景の `Capsule`（`Color.primary.opacity(0.08)`）と、幅 `全幅 * 割合（0〜1 に丸める）` の `Capsule`（`Color.accentColor.opacity(0.7)`）を重ねる。

パスの表示では、ホームディレクトリで始まるパスの先頭を `~` に置き換える。
Finder での表示には `NSWorkspace.shared.activateFileViewerSelecting([URL(fileURLWithPath: path)])` を使う。

#### フッター

`VStack(alignment: .leading, spacing: 8)` に、上から次を並べる。

1. スキャン中なら、`HStack(spacing: 6)` に `ProgressView().controlSize(.small)` と `"スキャン中… \(formatBytes(progressBytes))"`（`monospacedDigit`）を並べて `.caption` にする。
   その下に `progressPath` を `~` 表記で出す（`.caption2`、secondary、1 行、中央を省略）。
   スキャン中でなく `home` があれば、`"最終スキャン: \(scannedAt.formatted(date: .abbreviated, time: .shortened))"`（`.caption`、secondary）を出す。
2. `unreadableCount > 0` なら、`HStack(spacing: 6)` に `lock.fill`（secondary）、`"読めないフォルダ \(数) 件（その分小さく表示）"`、`Spacer()`、リンク形式のボタン「フルディスクアクセス…」を並べて `.caption` にする。
   ボタンは上記の URL を `NSWorkspace.shared.open` で開く。
3. `loginItemNeedsApproval` なら、`HStack(spacing: 6)` に `exclamationmark.circle`（orange）、「ログイン時の起動にはシステム設定での許可が必要です」、`Spacer()`、リンク形式のボタン「開く」を並べて `.caption` にする。
   ボタンは `SMAppService.openSystemSettingsLoginItems()` を呼ぶ。
4. 最下段の `HStack`：
   - 「再スキャン」ボタン（スキャン中は無効）。
   - `.checkbox` スタイルの Toggle「ログイン時に起動」（`.caption`）。get は `launchAtLogin`、set は `setLaunchAtLogin(_:)`。
   - `Spacer()`。
   - 「終了」ボタン（`NSApp.terminate(nil)`）。

### 開発用のスナップショット（Snapshot.swift）

`@MainActor func renderSnapshot(to path: String, drill: String? = nil)` は、キャッシュ済みの結果でポップオーバーを描画し、PNG に書き出す。
メニューバーをスクリーンショットで確認できない環境での見た目の確認に使う。

1. `_ = NSApplication.shared` で初期化し、`NSApp.setActivationPolicy(.prohibited)` にする。
2. `AppModel(autoStart: false)` を作る。
3. `drill`（例：`Library/Developer`）があれば、`home.root` から `/` で区切った名前を順に `children` から探して `stack` を作る。見つからなければそこで止める。
4. `PopoverView(model:initialStack:).background(.background)` を `NSHostingView` に入れ、`fittingSize` の大きさにして、borderless の `NSWindow` の `contentView` にする。
5. `layoutSubtreeIfNeeded()` の後、RunLoop を 0.5 秒回す。
6. `bitmapImageRepForCachingDisplay(in:)` と `cacheDisplay(in:to:)` で描画し、PNG で書き出す。

## ビルドとパッケージング

`scripts/build-app.sh`（bash、`set -euo pipefail`、リポジトリのルートに `cd`）は次を行う。

1. `swift build -c release` を実行し、`swift build -c release --show-bin-path` で得たディレクトリの `Aki` を取る。
2. `build/Aki.app` を消して作り直し、`Contents/MacOS/Aki` に実行ファイルを、`Contents/Info.plist` に下の内容を置く。`Contents/Resources` も作る。
3. `codesign --force --sign - build/Aki.app` で ad-hoc 署名する。
   リンカがすでに ad-hoc 署名しているので、`replacing existing signature` と出るのは正常である。

スクリプトには実行権限を付ける（`chmod +x scripts/build-app.sh`）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key><string>dev.aki.Aki</string>
    <key>CFBundleName</key><string>Aki</string>
    <key>CFBundleDisplayName</key><string>Aki</string>
    <key>CFBundleExecutable</key><string>Aki</string>
    <key>CFBundlePackageType</key><string>APPL</string>
    <key>CFBundleShortVersionString</key><string>0.1.0</string>
    <key>CFBundleVersion</key><string>1</string>
    <key>LSMinimumSystemVersion</key><string>14.0</string>
    <key>LSUIElement</key><true/>
    <key>NSHighResolutionCapable</key><true/>
</dict>
</plist>
```

`LSUIElement` を true にして、Dock にアイコンを出さない。

`Makefile` のターゲットは次のとおり。

- **`app`**：`./scripts/build-app.sh`
- **`test`**：`swift test`
- **`run`**：`app` の後に `open build/Aki.app`
- **`install`**：`app` の後に `~/Applications` を作り、既存の `~/Applications/Aki.app` を消してから `cp -R build/Aki.app ~/Applications/`（sudo は不要）
- **`clean`**：`rm -rf .build build`

## テスト

Swift Testing（`import Testing`、`@testable import AkiCore`）で書く。
テストは `FileManager.default.temporaryDirectory` の下に `aki-test-<UUID>` のディレクトリを作り、そこだけを触る。
ファイルは `0xAB` で埋めて書く（ゼロ埋めだと疎ファイルとして扱われ、ブロック数が期待とずれうる）。
後片付けでは、`chmod 000` にしたディレクトリの権限を `0o755` に戻してから削除する。
以下、MB は 1,000,000 バイトとする。
threshold を書いていないテストは、すべて `threshold: 0` でスキャンする（既定の 50MB では子ノードが残らない）。

ScannerTests.swift：

- **`sumsFileSizes`**：`a/f1`、`b/f2`、`top.txt`（各 1MB）を threshold 0 でスキャンする。合計が 3MB 以上 3.5MB 未満、子の名前が {a, b, top.txt}、`a` はディレクトリ、`top.txt` はファイル、`a` は 1MB 以上、unreadable は 0。
- **`childrenSortedDescending`**：`small/f`（1MB）、`big/f`（3MB）、`mid/f`（2MB）で、子の並びが big、mid、small。
- **`prunesBelowThresholdButKeepsSize`**：`a/f1`（1MB）と `big/deep/f`（12MB）を threshold 10MB でスキャンする。子は big だけで、合計は 13MB 以上。`big/deep` が残り、`big/deep/f` がファイルノードとして正しい絶対パスで残る。
- **`dedupesHardLinks`**：`a/f`（1MB）と、そのハードリンク `b/g` で、合計が 1MB 以上 1.5MB 未満。
- **`unreadableDirectoriesAreCountedAndTreeStaysIntact`**：`ok/f`、`locked/f`、`deep/inner/x`（2MB）、`deep/inner/locked/f`、`deep/after/y`（指定のないものは 1MB）を作り、`locked` と `deep/inner/locked` を `chmod 000` にする。unreadable は 2、`ok` は 1MB 以上、`deep` は 3MB 以上 3.5MB 未満、`deep` の子は {inner, after}、`inner` は 2MB 以上、合計は 4.5MB 未満。
- **`unreadableRootDoesNotCrash`**：`x/f`（1MB）を作って `x` を `chmod 000` にし、`x` を起点にスキャンすると、unreadable が 1、合計が 0。
- **`progressReachesTotal`**：`a/f`（1MB）と `b/c/g`（2MB）を progress 付きでスキャンし、`progress.bytes == root.size`。

BreakdownTests.swift：

- **`otherIsRemainder`**：total 1000、available 100、system 50、home 500、apps 100 で、`Breakdown(apps: 100, home: 500, system: 50, other: 250, free: 100)` と等しい。
- **`otherIsClampedToZero`**：total 1000、available 400、system 50、home 700、apps 100 で、other が 0、free が 400。
- **`currentVolumeSystemIsPlausible`**：実機の `VolumeStats.current()` で、total > 0、0 < available ≦ total、system > 0、system < used / 2。コンテナ全体の値を拾っていないことの確認で、実際のディスクを読む。
- **`missingScanCountsAsZero`**：total 1000、available 100、system 50、home と apps が nil で、home と apps が 0、other が 850。

これらのテストは木が浅いため、`splitDepth` より上の並列の列挙だけを通り、`fts` の経路（特に `FTS_DNR` と `FTS_ERR` の処理）は通らない。
`fts` の経路は、次の節の `du` との突き合わせで確認する。

## 動作確認

1. `swift test` がすべて通る（11 件）。
2. `make app` が警告なく通り、`build/Aki.app` ができる。
3. `build/Aki.app/Contents/MacOS/Aki --scan <dir>` の `total_kb` と `du -skx <dir>` の値が一致する。
   起点から 4 階層下のディレクトリ（`<dir>/a/b/c/d`）から先は `fts` でたどるので、それより深い位置にファイルと `chmod 000` のディレクトリを置いた木を一時ディレクトリに作って確かめる。
   ハードリンクを 1 つ入れ、unreadable が `chmod 000` にしたディレクトリの数と一致することも確かめる。
   ホーム全体ではなく、権限ダイアログの出ないフォルダ（一時ディレクトリや `~/Library/Caches` など）で確かめる。
   参考までに、作者の環境では `~/go` の集計が約 10〜13 秒（`du` は約 31 秒）、ホーム全体のスキャンが約 4 分だった。
4. アプリを一度起動してキャッシュができていれば、`Aki --snapshot out.png` や `Aki --snapshot out.png Library` でポップオーバーの見た目を PNG で確かめられる。
   起動はユーザーの了承を得てから行う。

## 権限と ad-hoc 署名

- 初回のスキャンでは、デスクトップ、書類、ダウンロードや「他のアプリのデータ」へのアクセス許可を求めるダイアログが出る。
- `~/Library/Mail` や `~/Library/Containers` の一部はフルディスクアクセスがないと読めない。読めない分はサイズが小さく表示され、フッターに件数が出る。フルディスクアクセスはユーザーが自分で許可する。
- ad-hoc 署名では再ビルドのたびにアプリの識別情報（cdhash）が変わるので、許可をやり直す必要がある。

## 現状の挙動として残している制限

次の点は意図的に直していない。
再現するときも、この挙動のまま実装する。

- **ディレクトリ自体のブロックは数えない**：`du` はディレクトリの `st_blocks` も数えるが、Aki はディレクトリ以外だけを数える。
  APFS ではディレクトリのブロック数が 0 なので一致するが、HFS+ や SMB などでは合計がずれうる。
- **`total_kb` は切り捨て**：`size / 1024` で切り捨てるが、`du -k` は 512 バイト単位のブロックを切り上げる。
  APFS は 4 KiB 単位で割り当てるので一致する。
- **`walk` の中のマウントポイント**：`FTS_XDEV` で中には入らないが、`FTS_D` と `FTS_DP` は来るので、threshold 0 ではサイズ 0 のノードになる。
  `scanDir` の階層ではマウントポイントを捨てるので扱いが揃っていないが、既定の閾値ではノードに残らない。
- **ハードリンクの重複排除はスキャンごと**：ホームと `/Applications` は別々にスキャンするので、両方にまたがるハードリンクは内訳で二重に数えられる。

## 実装時の注意

- Swift 6 の厳格な並行性チェックで、`concurrentPerform` のクロージャが `var` をキャプチャすると警告になる。子ディレクトリの一覧は `let` に写してから渡す。
- `AppModel` のポーリング用の `Task` で `[weak self]` を使うと、外側の `Task` のキャプチャと食い違って警告になる。ポーリングの `Task` では `self` をそのまま使う（スキャン終了時に cancel する）。
- `os_log` を見るときは `/usr/bin/log` をフルパスで呼ぶ。zsh では組み込みの `log` コマンドが優先される。
