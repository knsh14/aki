# Aki (あき)

A macOS app that shows free storage space in the menu bar. Click it to see a rough breakdown of used space and to drill down through the large folders and files under your home directory (it is read-only and has no delete feature).

- Menu bar: free space (updated every 60 seconds; a warning icon appears when less than 10% is free)
- Popover: a usage bar with a breakdown (apps, home, system, other, free), and the large items in your home directory (50 MB or larger, top 20 per level)
  - Click a folder to open it, or a file to show it in Finder. Right-click to copy the path
- Launch at login: on the first launch from `/Applications` or `~/Applications`, the app registers itself as a login item. You can toggle this with the checkbox in the popover (copies launched from `build/` are not registered)
- Scan results are cached in `~/Library/Caches/dev.aki.Aki/scan.json`. On launch, the app rescans if the last scan is more than an hour old
- The UI is in Japanese

## How to implement

This repository does not contain the implementation (Swift sources, tests, or build scripts).
It contains only [AGENTS.md](AGENTS.md), the spec for reproducing Aki, this README, and [LICENSE](LICENSE).
Have a coding agent read AGENTS.md and implement the app on your own machine.

1. Clone the repository and start a coding agent (Claude Code, Codex, etc.) in that directory.
2. Ask it to "Implement Aki as specified in AGENTS.md, and make the tests and the build pass."
3. When it is done, install the app with the steps in "Build and run" below.

Do not commit the files the agent generates.
With git, add them to `.git/info/exclude` so they stay untracked.
To change the spec, update AGENTS.md and commit it.

## Build and run

Use these after implementing the app.
Requires macOS 14 or later and Xcode (Swift 6).

```bash
make app       # build build/Aki.app (ad-hoc signed)
make run       # build and launch
make install   # copy to ~/Applications
make test      # run the unit tests
```

## Permissions

- On the first scan, macOS asks for access to Desktop, Documents, Downloads, and data from other apps.
- Folders such as `~/Library/Mail` cannot be read without Full Disk Access, so their sizes are shown smaller. Open the settings pane with the "フルディスクアクセス…" (Full Disk Access…) button in the popover and allow Aki.
- Because the app is ad-hoc signed, **rebuilding it changes the app's identity, and you have to grant the permissions again**. Once you have run `make install`, avoid rebuilding.

## What the numbers mean

- Free: the same basis as Finder's "Available" (includes purgeable space). It can be larger than what `df` shows.
- System: the used space of the read-only system volume, VM (swap), and Preboot.
- Other: used space minus apps, home, and system (/Library, /private/var, snapshots, etc.). It is shown as 0 if it would be negative.
- Sizes are counted by allocated blocks (the same as `du`). Hard links are counted once, and folders that have not been downloaded from iCloud or similar services are not entered.

## Development

```bash
build/Aki.app/Contents/MacOS/Aki --scan ~/some/dir   # print scan results without the UI
build/Aki.app/Contents/MacOS/Aki --snapshot out.png Library   # render the popover from the cache to a PNG (drilled into ~/Library)
```

## License

MIT License. See [LICENSE](LICENSE).
