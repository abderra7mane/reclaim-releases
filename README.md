# Reclaim

A macOS disk cleaner for developers: finds space that can be given back safely and cleans only what you choose.

This repo hosts Reclaim's releases: the installers, the command-line tool, the rules schema and the files Reclaim checks for updates.

Reclaim scans for package manager caches, browser caches, build output of projects you no longer work on, Docker leftovers, Xcode and Android data, files left behind by uninstalled apps, apps you never open and more. Each item shows its size, why it is listed, how risky removing it is and exactly what cleaning will do. Nothing is deleted until you tick it and confirm.

It comes as a desktop app and a command-line tool (`reclaim`) built on the same engine. macOS 13 or later, Apple Silicon and Intel.

## Safety model

- **You choose.** Only `safe` items are ticked by default. A confirmation lists every item, spells out each `review` item with its paths, and shows every command that will run.
- **Risk levels.**
  - `safe`: regenerated automatically, nothing is lost (caches, build output of stale projects, logs of uninstalled apps).
  - `caution`: costs time or bandwidth to get back (Docker images, simulators, toolchains, old IDE settings).
  - `review`: may be your own data or hard to get back (app leftovers, Docker volumes, VMs, installers, old Claude Code transcripts). Never ticked by default.
- **The tool's own command first.** When a tool has a cleanup command (`npm cache clean`, `docker builder prune`, `brew cleanup`, `go clean -modcache`, …), Reclaim runs it. Otherwise it deletes the files permanently. There is no Trash step.
- **Running apps are left alone.** An item whose app or daemon is running shows "Quit Google Chrome to clean" (or a hint such as "Run `gradle --stop` first") and is skipped. Reclaim never quits or signals apps; it only unloads the launch agent of an app whose files it removes, with `launchctl bootout`.
- **Build output must be git-ignored.** A folder in a project counts as build output only when its name is a known one (`node_modules`, `target`, `dist`, `.next`, …) **and** git ignores it. Projects with no commit and no edit for 90 days are ticked; active ones are listed unticked.
- **Hard guardrails** that no rule or flag can relax:
  - Only paths inside your home folder (and the per-user `X` and `C/<app>` folders under `/private/var/folders`) are deleted, and only inside the folders the rule declared. The one exception is uninstalling an app you ticked: its `.app` bundle in `/Applications` or `~/Applications` (at most three folders deep; apps owned by root or from the App Store are left alone).
  - Never these folders themselves: `~`, `~/Documents`, `~/Desktop`, `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Public`, `~/Library` and its `Application Support`, `Caches`, `Containers`, `Group Containers` and `Preferences` folders, and your project roots; never anything inside a `.git` folder, iCloud Drive, `~/Library/CloudStorage`, Dropbox, Google Drive, `~/Library/Keychains`, `~/.ssh`, `~/.gnupg` or the per-user temp folder `T`.
  - Symlinks are never followed; an item path that is a symlink is refused. Walks don't cross into other volumes.
  - Right before deleting, each path's device, inode and modification time must match the scan; otherwise the item is skipped as "changed since scan".
  - Commands are run as argument lists with the binary resolved on the `PATH`, never through a shell.
- **Personal data is information only.** Mail downloads, iOS backups and Messages attachments are shown with their size and "Show in Finder", never selectable.
- **Every run is logged** to `~/Library/Logs/reclaim/runs.jsonl`, dry runs included.
- **Real gain is measured.** Reclaim reads the free space of the Data volume before and after cleaning and shows it next to the estimate. Sizes marked `~` share storage with other files (APFS clones, pnpm hardlinks), so the real gain can be smaller; Docker's disk image shrinks later.

No admin rights, no background work, no network access except the update check you start in Settings.

## Install

### The app

Download the DMG for your Mac from the [latest release](https://github.com/abderra7mane/reclaim-releases/releases/latest): `_aarch64` for Apple Silicon, `_x86_64` for Intel, `_universal` for both.

Reclaim is signed but not notarized, so macOS blocks the first launch:

1. Open the DMG and drag Reclaim to Applications.
2. Open Reclaim. When macOS says it can't check it for malicious software, click Done.
3. Open System Settings → Privacy & Security, scroll to Security and click **Open Anyway** next to Reclaim.

Updates install from Settings → **Check for Updates** and keep Full Disk Access.

### The command-line tool

- **From the app:** Settings → Command-line tool → **Install** links `~/.local/bin/reclaim` to the copy inside the app, so it updates with the app.
- **Standalone:** download `reclaim_<version>_aarch64.tar.gz` or `_x86_64.tar.gz` from the release, unpack it and put `reclaim` on your `PATH`. If macOS blocks it, run `xattr -d com.apple.quarantine reclaim` once.

`~/.local/bin` must be on your `PATH`.

## Full Disk Access

Without Full Disk Access, macOS hides parts of your home folder: other apps' containers, the Trash, Safari, Mail and some browser profiles. Reclaim works without it and marks those items "Needs Full Disk Access".

- **App:** the first launch explains it and opens System Settings → Privacy & Security → Full Disk Access. Turn Reclaim on there. A banner reminds you while it's missing.
- **Command line:** grant Full Disk Access to your terminal app (Terminal, iTerm, Ghostty, …). `reclaim` prints a hint naming it when access is missing.

## Using the app

Click **Scan**. Results stream in per category; the last scan is shown instantly on the next launch ("Scanned 2 hours ago · Rescan"). Expand an item to see its paths, the reason, the exact action and facts such as last commit or last used date. Tick items, click **Clean N Items…**, check the confirmation and confirm. The result shows the estimated and the real gain, and anything skipped or failed with the reason.

Besides the categories, the sidebar has two views:

- **Unused apps:** every installed app with last used date, version, app and data size. Apps not opened for 180 days are flagged. Ticking one uninstalls it with its data (`brew uninstall --cask --zap` for Homebrew casks). Apps owned by root or from the App Store show "Needs admin".
- **Other large folders:** folders of 1 GB or more that no rule covers, read only, with "Show in Finder".

Settings has the theme (Native, Console, Studio), light or dark (System, Light, Dark), project folders, thresholds, the rules and logs folders, Full Disk Access, the command-line tool and updates.

Keyboard: ⌘R rescan, ⌘. stop, ⌘↩ clean / confirm, Esc cancel.

## Command line

```
reclaim scan   [--category ID]... [--rule ID]... [--json]
reclaim clean  [--category ID]... [--rule ID]... [--item ITEM_ID]... [--safe] [--dry-run] [--yes] [--json]
reclaim apps   [--unused] [--json]
reclaim rules  list [--json] | show ID | validate FILE...
reclaim log    [--last N] [--json]
```

- `reclaim scan` lists every item grouped by category: size (`~` when approximate), risk, ✔ when ticked by default, and status such as "Quit Google Chrome to clean". The footer shows the Data volume's free space, the total reclaimable and the total ticked. It never changes anything. An unfiltered scan also refreshes the app's cached scan.
- `reclaim clean` scans the selected rules again, then cleans:
  - by default, the items ticked by default;
  - `--safe`: only those whose risk is `safe`;
  - `--rule` / `--category`: every cleanable item of those, any risk (for unused apps, only the unused ones);
  - `--item`: exactly these items (ids from `reclaim scan --json` or `reclaim apps --json`).

  It prints the summary and asks `Delete N items (X GB)? [y/N]` unless `--yes`. `--dry-run` runs every check but deletes nothing and runs no cleanup command. Without a terminal, `clean` needs `--yes` or `--dry-run`.
- `reclaim apps` lists installed apps, oldest last use first; `--unused` only the flagged ones.
- `reclaim rules list | show ID | validate FILE...` shows the active rules (built-in and yours) and checks rule files. See [RULES.md](RULES.md).
- `reclaim log` shows past runs: date, source (`cli`, `app` or `dry run`), items, estimated and real gain, skipped and failed items.

Exit codes: `0` success, `1` error (bad arguments, invalid rules), `2` cancelled at the prompt, `3` finished with some items skipped or failed.

Examples:

```sh
reclaim scan --category package-caches
reclaim clean --dry-run                      # what the default selection would do
reclaim clean --rule npm-cache --rule bun-cache --yes
reclaim clean --item 'project-build-output:~/Projects/old-app' --yes
reclaim apps --unused
reclaim log --last 3
```

Categories: `package-caches`, `browsers`, `ide`, `xcode`, `android`, `rust`, `docker`, `projects`, `logs`, `app-leftovers`, `unused-apps`, `large-items`, `claude-code`, `system`, `personal-data`, `other-large-folders`. `reclaim rules list` shows the rule ids.

## JSON output

All `--json` output is pretty-printed JSON on stdout. Sizes are bytes (allocated on disk); times are seconds since the Unix epoch.

### `reclaim scan --json`

```jsonc
{
  "scanned_at": 1792680000,
  "duration_ms": 31842,
  "volume": { "total": 494384795648, "free": 104639434752 }, // Data volume; null if unreadable
  "full_disk_access": "granted",                           // granted | missing | unknown
  "items": [ /* Item, see below */ ],
  "warnings": [{ "rule_id": "docker-build-cache", "message": "Docker isn't running: start Docker to scan" }]
}
```

An item:

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | Stable id for `clean --item`: the rule id, or `<rule-id>:<key>` for rules with several items, where the key is the path (`~/…` inside the home folder, absolute otherwise) or a name such as a bundle ID or a toolchain. |
| `rule_id`, `rule_name` | string | The rule that found it. |
| `category` | string | One of the categories above. |
| `title` | string | Display name. |
| `paths` | array | `{ path, size, approximate?, file?, identity? }`; `identity` is `{ dev, ino, mtime_ns }` at scan time. |
| `size` | number | Total bytes. |
| `approximate` | bool | Shown with `~`: the files share storage with others. |
| `risk` | string | `safe`, `caution` or `review`. |
| `preselected` | bool | Ticked by default. |
| `reason` | string | Why it is listed and what removing it costs. |
| `action` | object | `{ "kind": "delete" }`, `{ "kind": "command", "argv": [...] }` or `{ "kind": "none" }` (information only). |
| `prepare` | array of argv | Optional. Commands run first, e.g. `launchctl bootout` for a loaded launch agent. |
| `follow_up` | argv | Optional. A command run after a successful action, e.g. `git worktree prune`. |
| `owned_by` | object | Optional. `{ apps?, processes?, hint? }` that must not run while it is cleaned. |
| `owner` | object | Optional. `{ name, label }` of the owner running at scan time, e.g. `"label": "Quit Google Chrome to clean"`. |
| `status` | string | `ok`, `owner-running`, `needs-fda`, `needs-admin` or `info-only`. Only `ok` items can be cleaned. |
| `warnings` | array of string | Optional, e.g. a size that is a lower bound because a subfolder was unreadable. |
| `facts` | object | Optional string map, e.g. `stale`, `last_commit`, `last_edit` (projects), `bundle_id`, `version`, `last_used`, `unused`, `app_path`, `uninstall`, `homebrew_cask` (apps), `uncommitted_changes`, `branch` (worktrees). |

Optional fields are left out when empty.

### `reclaim clean --json` and `reclaim log --json`

`clean --json` prints the run record (the summary goes to stderr), or `null` when nothing was selected; `log --json` prints an array of them, newest first. They are the lines of `~/Library/Logs/reclaim/runs.jsonl`. Optional fields are left out when empty:

```jsonc
{
  "run_id": "1792680000-4242-0",
  "started_at": 1792680000,
  "finished_at": 1792680012,
  "dry_run": false,
  "source": "cli",              // cli | app
  "free_before": 104639434752,  // null when unreadable
  "free_after": 105102000000,
  "real_gain": 462565248,       // can be negative
  "items": [
    {
      "item_id": "npm-cache",
      "rule_id": "npm-cache",
      "title": "npm cache",
      "paths": ["/Users/me/.npm/_cacache"],
      "command": ["npm", "cache", "clean", "--force"], // also prepare / follow_up when present
      "estimate": 428847104,
      "outcome": "command-ok",  // deleted | command-ok | skipped | failed
      "freed_estimate": 428847104,
      // skipped items have "reason": { "code": "owner-running", "label": "…" } (see below), failed ones "error"
      "command_output": "…"     // truncated to 8 KB
    }
  ]
}
```

Skip reasons (`reason.code`): `not-selectable` (`status`), `nothing-to-clean`, `unknown-rule`, `owner-running` (`label`), `changed-since-scan` (`path`), `tool-missing` (`binary`), `refused` (`message`).

### `reclaim apps --json`

An array of `{ item_id, name, bundle_id, version, path, last_used, unused, app_size, data_size, total, status, label, uninstall, homebrew_cask }`. `last_used` is `null` when unknown (never counted as unused); `uninstall` is `delete`, `homebrew`, `needs-admin` or `toolbox` (apps installed by JetBrains Toolbox are uninstalled there).

### `reclaim rules list --json`

The active rules with their full definitions, as in [RULES.md](RULES.md).

## Config

`~/.config/reclaim/config.toml` (or `$XDG_CONFIG_HOME/reclaim/config.toml`), shared by the app and the CLI and created on the first scan:

```toml
[projects]
roots = ["~/Projects", "~/Workspace"] # folders searched for git repos; detected on first run
stale_days = 90                       # no commit and no edit for this long → build output ticked

[apps]
unused_days = 180                     # apps not opened for this long are flagged

[large_folders]
min_gb = 1                            # threshold of "Other large folders"

[leftovers]
allow = []                            # extra bundle IDs that are never app leftovers

[rules]
disabled = []                         # rule ids to skip

[app]
theme = "native"                      # native | console | studio (the app only)
appearance = "system"                 # system | light | dark (the app only)
```

Unknown keys and invalid values are reported as warnings and fall back to the defaults. On first run, the project roots are the folders directly in your home folder that contain a git repo within three levels (except `Library`, `Documents`, `Downloads`, other system folders, hidden and cloud folders). Edit them in Settings or in the file.

## Where Reclaim keeps its files

| Path | What |
| --- | --- |
| `~/.config/reclaim/config.toml` | Config |
| `~/.config/reclaim/rules/*.toml` | Your rules ([RULES.md](RULES.md)) |
| `~/Library/Logs/reclaim/runs.jsonl` | Run log |
| `~/Library/Application Support/com.abdou.reclaim/last-scan.json` | Last full scan |

Reclaim never cleans its own folders.

## Updating from 0.1.0

Reclaim 0.1.0 looks for updates in a repo that is no longer public, so it doesn't find new versions. Download the latest DMG from this repo once and replace the app in Applications; your config and rules stay. Updates install from Settings after that.

## License

Free to use; all rights reserved. See [LICENSE](LICENSE).
