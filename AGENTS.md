# Aki (あき) Implementation Spec

This repository does not contain the implementation (Swift sources, tests, or build scripts).
This file is a specification that lets a coding agent implement Aki from scratch on a local machine and reproduce the behavior of the author's local implementation.
User-facing documentation is in README.md.

## Rules for agents

- **Do not commit implementation files**: the repository tracks only `AGENTS.md`, `README.md`, and `LICENSE`.
  With git, add the generated files to `.git/info/exclude` so they stay untracked.
  With jj, limit auto-tracked files to these three (the value is parsed as a TOML string, so it needs an extra layer of quoting).

  ```bash
  jj config set --repo snapshot.auto-track "'root-file:\"AGENTS.md\" | root-file:\"README.md\" | root-file:\"LICENSE\"'"
  ```
- **This file is the source of truth**: when you change the implementation's behavior, update this file in the same change.
- **Tests touch only temporary directories**: neither unit tests nor manual checks scan the whole home directory from the shell.
  It takes minutes, and macOS privacy protection (TCC) shows the terminal one permission dialog after another.
- **Ask the user before installing or launching**: the first launch from `~/Applications` or a similar location registers a login item, so confirm with the user before running `make install` or `make run`, or launching the app.

## What to build

A macOS app that shows free storage space in the menu bar.
Clicking it opens a popover with a rough breakdown of used space and the large folders and files under the home directory.
Folders can be drilled into level by level and revealed in Finder.
The app is read-only and has no delete feature.

- **App name**: Aki (あき)
- **Bundle ID**: `dev.aki.Aki`
- **Supported OS**: macOS 14 or later (required for `@Observable`)
- **Language and tools**: Swift 6 (swift-tools-version 6.0), SwiftPM, SwiftUI, Swift Testing. No external dependencies. Do not create an Xcode project.
- **UI text**: all in Japanese. Use the Japanese strings in this spec verbatim and do not translate them. English in parentheses after a string is only a gloss for readers and does not appear in the UI.

## File layout

```
Package.swift
Sources/AkiCore/SizeNode.swift     SizeNode, ScanResult, ScanProgress
Sources/AkiCore/Scanner.swift      directory traversal
Sources/AkiCore/Volume.swift       VolumeStats, Breakdown, formatBytes
Sources/Aki/main.swift             entry point (including the hidden CLI)
Sources/Aki/App.swift              MenuBarExtra
Sources/Aki/AppModel.swift         state, scanning, cache, login item
Sources/Aki/PopoverView.swift      popover UI
Sources/Aki/Snapshot.swift         development only: renders the popover to a PNG
Tests/AkiCoreTests/ScannerTests.swift
Tests/AkiCoreTests/BreakdownTests.swift
scripts/build-app.sh               assembles the .app and ad-hoc signs it
Makefile
.gitignore                         .build/ build/ .swiftpm/ (local only; do not commit)
```

`AkiCore` is the library under test and has no UI.
`Aki` is the executable and depends on `AkiCore`.

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

Because `Aki` is a separate module, make every type, property, initializer, and function in AkiCore `public`, except those described below as internal or private.
A public struct's memberwise initializer is still internal, so write an explicit `public init` for each struct.
The imports for each file are:

- **SizeNode.swift**: `Foundation`, `os` (`OSAllocatedUnfairLock`)
- **Scanner.swift**: `Darwin`, `Foundation`, `os`
- **Volume.swift**: `Darwin`, `Foundation`
- **main.swift**: `Foundation`, `AkiCore`
- **App.swift**: `SwiftUI`, `AkiCore`
- **AppModel.swift**: `Foundation`, `Observation`, `os` (`Logger`), `ServiceManagement`, `AkiCore`
- **PopoverView.swift**: `AppKit`, `ServiceManagement` (`openSystemSettingsLoginItems()`), `SwiftUI`, `AkiCore`
- **Snapshot.swift**: `AppKit`, `AkiCore`, `SwiftUI`

`public enum Scanner` has the same name as `Foundation.Scanner`, but in Swift 6.4 the type declared in the current module takes precedence, so it can be referenced without the `AkiCore.` prefix.

### Data types (SizeNode.swift)

- **`SizeNode`**: a struct conforming to `Codable, Sendable, Identifiable, Hashable`.
  Its properties are `name: String`, `path: String` (an absolute path), `size: Int64`, `isDirectory: Bool`, and `children: [SizeNode]`, all `var`.
  `id` returns `path`.
  `children` is sorted by size in descending order.
  The initializer is `init(name:path:size:isDirectory:children: = [])`.
- **`ScanResult`**: a struct conforming to `Codable, Sendable`.
  It has `root: SizeNode`, `unreadableCount: Int` (the number of directories whose contents could not be read, for example because of missing permissions), and `scannedAt: Date`, all `var`.
  The initializer is `public init(root:unreadableCount:scannedAt:)`.
- **`ScanProgress`**: a `final class` conforming to `Sendable`.
  Group its internal state (`bytes: Int64`, `currentPath: String`) into a private struct and protect it with `OSAllocatedUnfairLock(initialState:)`.
  The initializer is a parameterless `public init()`.
  For reading, expose the computed properties `public var bytes: Int64` and `public var currentPath: String`, which take the lock and return the value.
  The writers `func add(_ bytes: Int64)` (does nothing for 0) and `func setPath(_:)` are internal.
  The scan threads write to it and the UI thread reads from it.

### Traversal (Scanner.swift)

Put the following API on `public enum Scanner`.

```swift
public static func disableDatalessMaterialization()
public static func scan(_ path: String, threshold: Int64 = 50_000_000,
                        progress: ScanProgress? = nil) async -> ScanResult
public static func scanSync(_ path: String, threshold: Int64 = 50_000_000,
                            progress: ScanProgress? = nil) -> ScanResult
```

`disableDatalessMaterialization()` sets a process-wide policy so that reading files that have not been downloaded (dataless files, for example in iCloud Drive) does not trigger a download.
Call it once, right after launch.

```swift
_ = setiopolicy_np(IOPOL_TYPE_VFS_MATERIALIZE_DATALESS_FILES,
                   IOPOL_SCOPE_PROCESS,
                   IOPOL_MATERIALIZE_DATALESS_FILES_OFF)
```

`scan` runs `scanSync` on `DispatchQueue.global(qos: .utility)` and bridges it to async with `withCheckedContinuation`.
`scanSync` creates a shared context, traverses the tree, and returns `ScanResult(root:unreadableCount:scannedAt: Date())`.

#### Counting sizes

- Count sizes by allocated blocks (`Int64(st.st_blocks) * 512`), the same basis as `du`.
- For regular files (`S_IFREG`) with `st_nlink > 1`, record the `(st_dev, st_ino)` pair in a shared `Set` and count every occurrence after the first as 0 (hard-link deduplication).
  The `Set` is accessed concurrently, so protect it with `OSAllocatedUnfairLock`.
- Count everything that is not a directory (regular files, symbolic links, and so on) as a file. Do not follow symbolic links.
- Also count unreadable directories in the shared context, under a lock.
- The shared context is a `private final class Context: Sendable`.
  It holds `threshold`, `progress`, an `OSAllocatedUnfairLock<Set<InodeKey>>` (`InodeKey` is a Hashable struct with `dev: Int32` and `ino: UInt64`), and an `OSAllocatedUnfairLock<Int>` (the unreadable count).
  One is created per `scanSync` call, so hard-link deduplication applies only within a single scan.
- `SF_DATALESS` is imported as Int32 and `st_flags` as UInt32, so compare with `st.st_flags & UInt32(SF_DATALESS)`.
- Join paths with `(path as NSString).appendingPathComponent(_:)` and extract names with `lastPathComponent`.

#### Pruning by threshold

Keep only folders and files at or above `threshold` (default 50,000,000 bytes) as `SizeNode`s.
Smaller items do not become nodes; only their sizes are added to the parent.
So every node's `size` is the total of everything under it and can be larger than the sum of its `children`'s sizes.
The scan's starting point (root) is always returned as a node, regardless of its size.

#### Traversal steps

Enumerate the upper levels directly and in parallel, and walk the deeper levels with `fts`, one thread per subtree.
The parallelization depth is `splitDepth = 4`.

1. **Starting point**: if `lstat` fails, return a directory node with `size: 0` (not counted as unreadable).
   If the starting point is not a directory, return a single node for that file and add its size to progress.
   If it is a directory, take its `st_dev` and proceed to `scanDir(path, dev, depth: splitDepth)`.
2. **`scanDir(path, dev, depth)`**: list the direct entries with `FileManager.default.contentsOfDirectory(atPath:)`.
   If listing fails, increment unreadable by 1 and return a directory node with `size: 0`.
   `lstat` each entry (ignore entries for which it fails) and sort them as follows.
   - Discard, without descending into them, directories whose `st_dev` differs from the starting point's (mount points of other volumes) and directories with `SF_DATALESS` set in `st_flags` (not downloaded).
   - Put the other directories in a list to be processed in parallel later.
   - Count and sum the sizes of files; a file at or above the threshold becomes a node with `isDirectory: false`.

   Add the file total to progress all at once when listing finishes.
   Process child directories in parallel with `DispatchQueue.concurrentPerform`.
   If `depth > 1`, call `scanDir(child, dev, depth - 1)`; otherwise, call `walk(child)`.
   Store the results by index in a lock-protected `[SizeNode?]` array, and drop `nil` (returned when `walk` fails in `fts_open`).
   The directory's own size is the file total plus the sum of the child directories' sizes.
   `children` is the file nodes at or above the threshold plus the directory nodes at or above the threshold, sorted by size in descending order.
3. **`walk(path) -> SizeNode?`**: first call `progress.setPath(path)`.
   Open the tree with `fts_open` using `FTS_PHYSICAL | FTS_XDEV | FTS_NOCHDIR`, and return `nil` if it fails.
   Pass `fts_open` a NULL-terminated array of C strings.

   ```swift
   let cPath = strdup(path)
   defer { free(cPath) }
   var argv: [UnsafeMutablePointer<CChar>?] = [cPath, nil]
   guard let fts = fts_open(&argv, FTS_PHYSICAL | FTS_XDEV | FTS_NOCHDIR, nil) else { return nil }
   defer { fts_close(fts) }
   ```

   `fts_info` is imported as UInt16 and `fts_level` as Int16, so compare `Int32(ent.pointee.fts_info)` with the `FTS_*` constants and use `Int(ent.pointee.fts_level)` as the level.
   Push a `Frame { level: Int, size: Int64 = 0, children: [SizeNode] = [] }` onto a stack for each directory and assemble the tree in post-order.
   Handle each `fts_info` returned by `fts_read` as follows.

   | `fts_info` | Action |
   | --- | --- |
   | `FTS_D` | Push a Frame with `fts_level`. If `SF_DATALESS` is set in the `st_flags` of `fts_statp`, call `fts_set(fts, ent, FTS_SKIP)` so the directory is not entered (an `FTS_DP` for the same entry follows immediately) |
   | `FTS_DP` | Pop only when the level at the top of the stack equals `fts_level` |
   | `FTS_DNR`, `FTS_ERR` | For an unreadable directory, one of these comes after `FTS_D` instead of `FTS_DP`. Only when the level at the top of the stack matches, increment unreadable by 1 and pop |
   | `FTS_F`, `FTS_SL`, `FTS_SLNONE`, `FTS_DEFAULT` | If the stack is not empty, count the size and add it to the top Frame's size. If it is at or above the threshold, add a file node to the top Frame's children |
   | Anything else (`FTS_NS`, `FTS_DC`, etc.) | Not counted |

   On pop, take the top Frame and handle it as follows.
   - The Frame whose removal leaves the stack empty (the starting point) becomes a node and the result, regardless of the threshold.
   - A Frame at or above the threshold becomes a node (with children sorted by size in descending order) and is added to the parent Frame's size and children.
   - A Frame below the threshold does not become a node; only its size is added to the parent Frame.

   A node's `name` is the last component of `fts_path`, and its `path` is `fts_path`.
   Add to progress each time 64,000,000 bytes accumulate, and also flush the accumulated amount when closing a Frame at or above the threshold and at the end of the walk.
   This makes `progress.bytes` equal to the starting point's size when the scan finishes.

### Capacity and breakdown (Volume.swift)

- **`VolumeStats`**: a struct conforming to `Sendable, Equatable`.
  It has `total`, `available`, and `system` (all `Int64` `var`s) and the computed property `used: Int64 { max(0, total - available) }`.
  The initializer is `public init(total:available:system:)`.
- **`VolumeStats.current() -> VolumeStats?`**: read `volumeTotalCapacity` and `volumeAvailableCapacityForImportantUsage` from the `resourceValues` of the home directory URL.
  If the latter is unavailable, use `volumeAvailableCapacity`.
  Return `nil` if `total` is unavailable.
  `available` uses the same basis as Finder's "Available" and includes purgeable space.
- **`system`**: the sum of the used space of the volumes `/`, `/System/Volumes/VM`, and `/System/Volumes/Preboot`.
  On APFS, the `f_blocks` and `f_bfree` of `statfs` describe the whole container, so do not use `statfs` (System would show up as several TB).
  Like `df`, use `ATTR_VOL_SPACEUSED` from `getattrlist`.
  The returned buffer starts with a 4-byte total length, followed by the used space as an `off_t`.

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

- **`Breakdown`**: a struct conforming to `Sendable, Equatable` with `apps`, `home`, `system`, `other`, and `free` (all `Int64` `var`s).
  The initializer is `public init(apps:home:system:other:free:)`.
  `static func make(volume: VolumeStats, home: Int64?, apps: Int64?) -> Breakdown` treats `nil` as 0 and sets `other = max(0, used - home - apps - system)` and `free = available`.
  Even when the sum exceeds the used space (during a scan or because of missing permissions), Other never goes below 0.
- **`public func formatBytes(_ bytes: Int64) -> String`**: `ByteCountFormatter.string(fromByteCount:countStyle: .file)` (decimal units).

## Aki (the app)

### Entry point (main.swift)

Do not mark `AkiApp` with `@main`.
A target that has `main.swift` does not compile if `@main` is used.
`main.swift` does the following, in order.

1. Call `Scanner.disableDatalessMaterialization()`.
2. If the arguments contain `--scan <path>`, run `scanSync` without the UI (expanding `~`, with the default 50MB threshold), print the result in the format below, and `exit(0)`.
   If no argument follows `--scan`, ignore the option (the same applies to `--snapshot`).
   Measure the elapsed time as a difference of `Date`s.
   The first line is `total_kb=<size/1024>\t<formatBytes>\tunreadable=<count>\telapsed=<seconds, one decimal place>s`, followed by the top 10 children, one per line, as `  <formatBytes>\t<name>`.
   This output is used for comparison with `du -skx`.
3. If the arguments contain `--snapshot <out.png> [path relative to home]`, call `renderSnapshot(to:drill:)` and `exit(0)`.
4. Otherwise, call `AkiApp.main()`.

### Menu bar (App.swift)

`struct AkiApp: App` holds `@State private var model = AppModel()`.
The scene is `MenuBarExtra { PopoverView(model: model) } label: { ... }` with `.menuBarExtraStyle(.window)`.
The label is an `HStack(spacing: 4)` with an SF Symbol and `Text(model.freeText)`.
The SF Symbol is normally `internaldrive`, and `exclamationmark.triangle.fill` when less than 10% is free.

### State and scanning (AppModel.swift)

Make it `@MainActor @Observable final class AppModel`.
Log to `private let log = Logger(subsystem: "dev.aki.Aki", category: "app")`, declared at the top of the file.

- **Read-only state** (`private(set) var`): `volume: VolumeStats?`, `home: ScanResult?`, `apps: ScanResult?`, `isScanning`, `progressBytes: Int64`, `progressPath: String`, `launchAtLogin`, `loginItemNeedsApproval`.
- **Computed properties**:
  - `breakdown`: if `volume` exists, `Breakdown.make(volume:home: home?.root.size, apps: apps?.root.size)`.
  - `freeText`: `formatBytes(volume.available)`, or `"—"` if there is no `volume`.
  - `isLow`: `Double(available) / Double(total) < 0.1` (false if total is 0 or less). Dividing as Int64 yields 0, so compute in Double.
  - `unreadableCount`: the sum of the unreadableCount of home and apps.
- **`init(autoStart: Bool = true)`**: load the cache, call `refreshVolume()`, and update the login item state.
  If `autoStart` is false, stop here (used for snapshots).
  If it is true, continue as follows.
  1. On the first launch of an installed copy, register the login item (see below).
  2. Start a `Task` that calls `refreshVolume()` every 60 seconds (`[weak self]`, a `while !Task.isCancelled` loop; the Task is not stored anywhere).
  3. If there is no `home`, or `home.scannedAt` is more than one hour (3600 seconds) old, call `startScan()`.
- **`refreshVolume()`**: `volume = VolumeStats.current()`.
- **`startScan()`**: do nothing if a scan is already running (no double start).
  Set `isScanning = true`, reset the progress display to 0 and an empty string, create a `ScanProgress`, and start a `Task`.
  Inside it, start a polling `Task` that reads the progress every 0.5 seconds and copies it to `progressBytes` and `progressPath`.
  Scan the home directory (`NSHomeDirectory()`, with progress) and `/Applications` (without progress) concurrently with `async let`, and await both.
  When both finish, cancel the poller, update `home` and `apps`, set `isScanning = false`, then call `refreshVolume()` and save the cache.
- **Login item**: use `SMAppService.mainApp`.
  - `setLaunchAtLogin(_ on: Bool)`: call `register()` or `unregister()` and log any failure. Then update the state.
  - Updating the state: store `status == .enabled` in `launchAtLogin` and `status == .requiresApproval` in `loginItemNeedsApproval`.
  - First launch of an installed copy: only when `Bundle.main.bundlePath` starts with `/Applications/` or `NSHomeDirectory() + "/Applications/"` and the UserDefaults key `loginItemConfigured` is false, call `setLaunchAtLogin(true)` and set `loginItemConfigured` to true.
    A copy launched from `build/` for testing is not registered.
    After the first launch, the checkbox setting decides.
- **Cache**: save `struct Cache: Codable { var home: ScanResult?; var apps: ScanResult? }` to `dev.aki.Aki/scan.json` in the user's Caches directory, using `JSONEncoder` with its default settings.
  Create the parent directory first, then write with `.atomic`.
  Silently ignore both read and write failures (`try?`).

### Popover (PopoverView.swift)

`struct PopoverView: View` holds `model: AppModel` and the drill-down path `@State private var stack: [SizeNode]`.
The initializer is `init(model:initialStack: [SizeNode] = [])`.
The node being shown, `current`, is `stack.last ?? model.home?.root`.

The constants are:

- Maximum number of rows shown per level: 20
- Row height: 30pt
- Size column width: 80pt
- Maximum list height: 330pt
- URL of the Full Disk Access settings pane: `x-apple.systempreferences:com.apple.preference.security?Privacy_AllFiles`

The whole view is a `VStack(alignment: .leading, spacing: 12)` containing, from top to bottom, the capacity and breakdown section, `Divider()`, the large folders section, `Divider()`, and the footer.
Apply `.padding(14)` and `.frame(width: 380)`.
Call `model.refreshVolume()` in `onAppear`.
Reset `stack` to empty with `onChange(of: model.home?.scannedAt) { stack = [] }` (the zero-parameter closure form; the one-parameter form is deprecated in macOS 14).

#### Capacity and breakdown

When `volume` and `breakdown` exist, lay out the following in a `VStack(alignment: .leading, spacing: 8)`.
Otherwise, show only `Text("容量を取得できませんでした")` (Could not get the capacity; secondary).

1. `HStack(alignment: .firstTextBaseline)`:
   On the left, `"空き \(formatBytes(available))"` (Free …) in `.title2.bold()` (red if `isLow`, otherwise primary).
   Then a `Spacer()`, and on the right, `"\(formatBytes(total)) 中 \(percent)% 使用"` (percent used of total) in `.caption`, secondary.
   `percent` is `Int((Double(used) / Double(max(total, 1)) * 100).rounded())`. Dividing as Int64 always gives 0%, so always compute in Double.
2. The usage bar (height 10).
3. The legend: a `LazyVGrid` (two columns, `GridItem(.flexible(), spacing: 16)` and `GridItem(.flexible())`, leading alignment, row spacing 4).
   Each item is an `HStack(spacing: 6)` with an 8pt circle, the label, a `Spacer()`, and the value (`monospacedDigit`, secondary), in `.font(.callout)`.
   If the value is `nil`, show `"…"`.

The breakdown items, in this order, are below.
The values of アプリ, ホーム, and その他 are `nil` while there is no home scan result yet.

| Label | Meaning | Value | Color |
| --- | --- | --- | --- |
| アプリ | Apps | `apps` | `.blue` |
| ホーム | Home | `home` | `.orange` |
| システム | System | `system` | `.gray` |
| その他 | Other | `other` | `.purple` |
| 空き | Free | `free` | `Color.primary.opacity(0.12)` |

The usage bar takes `parts: [(Double, Color)]` (each item's value as a Double and its color; `nil` counts as 0).
Inside a `GeometryReader`, place a `Rectangle` in each item's color in an `HStack(spacing: 1)`, with width `max(0, geo.size.width * value / sum - 1)`, and clip the whole bar with a `Capsule`.
`sum` is the sum of the values, with a minimum of 1.

#### Large folders

Use a `VStack(alignment: .leading, spacing: 6)`.

The header row is an `HStack(spacing: 6)` containing:
- If `stack` is not empty, a `chevron.left` back button (`.borderless`, help `戻る` (Back)). Pressing it removes the last element of `stack`.
- The heading text (`.headline`, one line, truncated at the head): `ホームの大きいフォルダ` (Large folders in Home) if `stack` is empty, otherwise `current.path` in `~` notation.
- `Spacer()`.
- If `current` exists, a `folder` icon button (`.borderless`, help `Finderで表示` (Show in Finder)). Pressing it reveals `current` in Finder.

If `current` exists, list the first 20 of its `children` as `NodeRow`s.
If `current.size` minus the sum of the sizes of the 20 rows shown is positive, append a `RestRow`.
Put the rows in a `VStack(spacing: 0)` inside a `ScrollView`, and set the `ScrollView`'s height to `min(max(row count, 1) * 30, 330)`.
If there is no `current`, show `初回スキャン中です。数分かかることがあります。` (Running the first scan. It can take a few minutes.) while scanning, or `まだスキャンしていません` (Not scanned yet) otherwise, in `.callout`, secondary.

**`NodeRow`** (takes `node`, `parentSize`, and `onOpen`) is an `HStack(spacing: 8)` containing:
- The icon (width 16): `folder.fill` for a folder, `doc.fill` for a file. The color is `node.isDirectory ? Color.accentColor : Color.secondary`.
- A `VStack(alignment: .leading, spacing: 3)` with the name (one line, truncated in the middle) and a proportion bar (height 3, `size / max(parentSize, 1)`), with `.frame(maxWidth: .infinity, alignment: .leading)`.
- The size (`monospacedDigit`, secondary, width 80, right-aligned).
- `chevron.right` (`.caption`, tertiary). On rows that cannot be opened, set its opacity to 0 so that only its width remains.

Rows have a horizontal padding of 6 and a height of 30.
While a row is hovered, draw `Color.primary.opacity(0.08)` with a corner radius of 6 as its background, and make the whole row clickable with `contentShape(Rectangle())`.
Clicking an openable row (a folder whose `children` is not empty) calls `onOpen` (which pushes it onto `stack`); clicking any other row reveals it in Finder.
The help text is `クリックで中を表示` (Click to show contents) for openable rows and `クリックで Finder に表示` (Click to show in Finder) for the others.
The context menu has `Finderで表示` (Show in Finder) and `パスをコピー` (Copy path).
`パスをコピー` calls `clearContents()` on `NSPasteboard.general` and then `setString(path, forType: .string)`.

**`RestRow`** (takes `size` and `parentSize`) has the same layout as `NodeRow`, with the icon `ellipsis.circle` (secondary) and the name `その他（小さい項目）` (Other (small items); secondary).
Place the chevron with opacity 0.
It has no hover highlight and no click action.

The proportion bar takes `fraction: Double` and, inside a `GeometryReader`, layers in a `ZStack(alignment: .leading)` a background `Capsule` (`Color.primary.opacity(0.08)`) and a `Capsule` (`Color.accentColor.opacity(0.7)`) whose width is the full width times the fraction (clamped to 0...1).

When displaying a path, replace a leading home directory with `~`.
To reveal an item in Finder, use `NSWorkspace.shared.activateFileViewerSelecting([URL(fileURLWithPath: path)])`.

#### Footer

A `VStack(alignment: .leading, spacing: 8)` containing, from top to bottom:

1. While scanning, an `HStack(spacing: 6)` with `ProgressView().controlSize(.small)` and `"スキャン中… \(formatBytes(progressBytes))"` (Scanning…; `monospacedDigit`), in `.caption`.
   Below it, `progressPath` in `~` notation (`.caption2`, secondary, one line, truncated in the middle).
   When not scanning and `home` exists, show `"最終スキャン: \(scannedAt.formatted(date: .abbreviated, time: .shortened))"` (Last scan; `.caption`, secondary).
2. If `unreadableCount > 0`, an `HStack(spacing: 6)` with `lock.fill` (secondary), `"読めないフォルダ \(count) 件（その分小さく表示）"` (N unreadable folders; sizes are shown smaller by that amount), `Spacer()`, and a link-style button `フルディスクアクセス…` (Full Disk Access…), in `.caption`.
   The button opens the URL above with `NSWorkspace.shared.open`.
3. If `loginItemNeedsApproval`, an `HStack(spacing: 6)` with `exclamationmark.circle` (orange), `ログイン時の起動にはシステム設定での許可が必要です` (Launch at login needs approval in System Settings), `Spacer()`, and a link-style button `開く` (Open), in `.caption`.
   The button calls `SMAppService.openSystemSettingsLoginItems()`.
4. The bottom `HStack`:
   - A `再スキャン` (Rescan) button (disabled while scanning).
   - A `.checkbox`-style Toggle `ログイン時に起動` (Launch at login; `.caption`). Its getter is `launchAtLogin` and its setter is `setLaunchAtLogin(_:)`.
   - `Spacer()`.
   - A `終了` (Quit) button (`NSApp.terminate(nil)`).

### Development snapshot (Snapshot.swift)

`@MainActor func renderSnapshot(to path: String, drill: String? = nil)` renders the popover from the cached results and writes it to a PNG.
It is for checking the look in environments where the menu bar cannot be captured in a screenshot.

1. Initialize with `_ = NSApplication.shared` and call `NSApp.setActivationPolicy(.prohibited)`.
2. Create `AppModel(autoStart: false)`.
3. If `drill` is given (for example `Library/Developer`), split it on `/` and build `stack` by looking up each name in turn among the `children`, starting from `home.root`. Stop at the first name that is not found.
4. Put `PopoverView(model:initialStack:).background(.background)` in an `NSHostingView`, size it to its `fittingSize`, and set it as the `contentView` of a borderless `NSWindow`.
5. After `layoutSubtreeIfNeeded()`, run the RunLoop for 0.5 seconds.
6. Render with `bitmapImageRepForCachingDisplay(in:)` and `cacheDisplay(in:to:)`, and write the result as a PNG.

## Build and packaging

`scripts/build-app.sh` (bash, `set -euo pipefail`, `cd` to the repository root) does the following.

1. Run `swift build -c release` and take `Aki` from the directory printed by `swift build -c release --show-bin-path`.
2. Delete and recreate `build/Aki.app`, put the executable at `Contents/MacOS/Aki` and the content below at `Contents/Info.plist`, and also create `Contents/Resources`.
3. Ad-hoc sign the bundle with `codesign --force --sign - build/Aki.app`.
   The linker has already ad-hoc signed the binary, so `replacing existing signature` in the output is expected.

Make the script executable (`chmod +x scripts/build-app.sh`).

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

`LSUIElement` is true so that no icon appears in the Dock.

The `Makefile` targets are:

- **`app`**: `./scripts/build-app.sh`
- **`test`**: `swift test`
- **`run`**: `app`, then `open build/Aki.app`
- **`install`**: `app`, then create `~/Applications`, delete any existing `~/Applications/Aki.app`, and run `cp -R build/Aki.app ~/Applications/` (no sudo needed)
- **`clean`**: `rm -rf .build build`

## Tests

Write the tests with Swift Testing (`import Testing`, `@testable import AkiCore`).
Each test creates an `aki-test-<UUID>` directory under `FileManager.default.temporaryDirectory` and touches only that directory.
Fill files with `0xAB` when writing them (zero-filled files may be treated as sparse, so their block counts can differ from what the test expects).
During cleanup, restore the permissions of directories set to `chmod 000` to `0o755` before deleting them.
Below, MB means 1,000,000 bytes.
Every test that does not state a threshold scans with `threshold: 0` (with the default of 50MB, no child nodes remain).

ScannerTests.swift:

- **`sumsFileSizes`**: scan `a/f1`, `b/f2`, and `top.txt` (1MB each) with threshold 0. The total is at least 3MB and less than 3.5MB, the children's names are {a, b, top.txt}, `a` is a directory, `top.txt` is a file, `a` is at least 1MB, and unreadable is 0.
- **`childrenSortedDescending`**: with `small/f` (1MB), `big/f` (3MB), and `mid/f` (2MB), the children are ordered big, mid, small.
- **`prunesBelowThresholdButKeepsSize`**: scan `a/f1` (1MB) and `big/deep/f` (12MB) with threshold 10MB. The only child is big, and the total is at least 13MB. `big/deep` remains, and `big/deep/f` remains as a file node with the correct absolute path.
- **`dedupesHardLinks`**: with `a/f` (1MB) and its hard link `b/g`, the total is at least 1MB and less than 1.5MB.
- **`unreadableDirectoriesAreCountedAndTreeStaysIntact`**: create `ok/f`, `locked/f`, `deep/inner/x` (2MB), `deep/inner/locked/f`, and `deep/after/y` (1MB unless stated otherwise), and `chmod 000` both `locked` and `deep/inner/locked`. unreadable is 2, `ok` is at least 1MB, `deep` is at least 3MB and less than 3.5MB, the children of `deep` are {inner, after}, `inner` is at least 2MB, and the total is less than 4.5MB.
- **`unreadableRootDoesNotCrash`**: create `x/f` (1MB), `chmod 000` `x`, and scan with `x` as the starting point. unreadable is 1 and the total is 0.
- **`progressReachesTotal`**: scan `a/f` (1MB) and `b/c/g` (2MB) with progress. `progress.bytes == root.size`.

BreakdownTests.swift:

- **`otherIsRemainder`**: with total 1000, available 100, system 50, home 500, and apps 100, the result equals `Breakdown(apps: 100, home: 500, system: 50, other: 250, free: 100)`.
- **`otherIsClampedToZero`**: with total 1000, available 400, system 50, home 700, and apps 100, other is 0 and free is 400.
- **`currentVolumeSystemIsPlausible`**: on the real machine, `VolumeStats.current()` gives total > 0, 0 < available ≤ total, system > 0, and system < used / 2. This checks that container-wide values are not being picked up, and it reads the actual disk.
- **`missingScanCountsAsZero`**: with total 1000, available 100, system 50, and home and apps nil, home and apps are 0 and other is 850.

Because these test trees are shallow, they exercise only the parallel enumeration above `splitDepth` and not the `fts` path (in particular, not the handling of `FTS_DNR` and `FTS_ERR`).
The `fts` path is checked by the comparison with `du` in the next section.

## Verification

1. `swift test` passes in full (11 tests).
2. `make app` succeeds with no warnings and produces `build/Aki.app`.
3. The `total_kb` printed by `build/Aki.app/Contents/MacOS/Aki --scan <dir>` matches `du -skx <dir>`.
   Directories 4 levels below the starting point (`<dir>/a/b/c/d`) and deeper are walked with `fts`, so check with a tree in a temporary directory that has files and a `chmod 000` directory below that depth.
   Include one hard link, and also check that unreadable equals the number of directories set to `chmod 000`.
   Check with a folder that does not trigger permission dialogs (a temporary directory, `~/Library/Caches`, and so on), not with the whole home directory.
   For reference, in the author's environment, scanning `~/go` took about 10–13 seconds (`du` took about 31 seconds), and scanning the whole home directory took about 4 minutes.
4. Once the app has been launched and a cache exists, `Aki --snapshot out.png` or `Aki --snapshot out.png Library` renders the popover to a PNG for checking its look.
   Launch the app only with the user's consent.

## Permissions and ad-hoc signing

- The first scan shows dialogs asking for access to Desktop, Documents, Downloads, and data from other apps.
- Parts of `~/Library/Mail` and `~/Library/Containers` cannot be read without Full Disk Access. Sizes appear smaller by the unreadable amount, and the footer shows the count. The user grants Full Disk Access themselves.
- With ad-hoc signing, the app's identity (cdhash) changes on every rebuild, so the permissions have to be granted again.

## Limitations kept as current behavior

The following are intentionally left unfixed.
Reproduce them as they are.

- **The blocks of directories themselves are not counted**: `du` also counts a directory's `st_blocks`, but Aki counts only non-directories.
  On APFS, directories have 0 blocks, so the totals match, but on HFS+, SMB, and other file systems the totals can differ.
- **`total_kb` is truncated**: it is floored with `size / 1024`, whereas `du -k` rounds 512-byte blocks up.
  APFS allocates in 4 KiB units, so the two match.
- **Mount points inside `walk`**: `FTS_XDEV` keeps the walk from entering them, but `FTS_D` and `FTS_DP` still arrive, so with threshold 0 they become nodes of size 0.
  At the `scanDir` levels, mount points are discarded, so the handling is inconsistent, but with the default threshold they do not remain as nodes.
- **Hard-link deduplication is per scan**: the home directory and `/Applications` are scanned separately, so hard links spanning both are counted twice in the breakdown.

## Implementation notes

- Under Swift 6's strict concurrency checking, capturing a `var` in the `concurrentPerform` closure triggers a warning. Copy the list of child directories into a `let` before passing it.
- Using `[weak self]` in `AppModel`'s polling `Task` conflicts with the outer `Task`'s capture and triggers a warning. Use `self` directly in the polling `Task` (it is cancelled when the scan finishes).
- To view `os_log` output, call `/usr/bin/log` by its full path. In zsh, the built-in `log` command takes precedence.
