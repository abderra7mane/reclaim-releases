# Writing rules

Everything Reclaim finds comes from rules: entries in TOML files that say where to look, how risky removing it is and how to clean it. The built-in rules are compiled into Reclaim; `reclaim rules show ID` prints one. Your own rules go in `~/.config/reclaim/rules/*.toml` (or `$XDG_CONFIG_HOME/reclaim/rules`), loaded after the built-ins in file-name order. Settings → Rules folder → Open creates and opens that folder.

A rule file can add rules, replace a built-in rule (same `id`) or disable one.

## A first rule

```toml
#:schema https://github.com/abderra7mane/reclaim-releases/releases/latest/download/rules.schema.json
schema = 1

[[rule]]
id = "my-tool-cache"
name = "My Tool cache"
category = "package-caches"
description = "Downloads My Tool keeps. It downloads them again when needed."
risk = "safe"
detector = "paths"

[rule.owner]
apps = ["com.example.MyTool"]

[rule.paths]
macos = ["~/Library/Caches/my-tool", "~/.mytool/cache"]
```

This lists one item, "My Tool cache", with the size of both folders, ticked by default because it is `safe`. Cleaning deletes both folders, unless My Tool is running: then the item shows "Quit My Tool to clean" and is skipped.

Check it, then see what it finds (both are read-only):

```sh
reclaim rules validate ~/.config/reclaim/rules/my-tools.toml
reclaim scan --rule my-tool-cache
reclaim clean --rule my-tool-cache --dry-run
```

The `#:schema` comment lets editors with [Taplo](https://taplo.tamasfe.dev) (e.g. the "Even Better TOML" VS Code extension) complete and check fields as you type. The schema is attached to every release as [`rules.schema.json`](https://github.com/abderra7mane/reclaim-releases/releases/latest/download/rules.schema.json).

## Fields

| Field | Required | Meaning |
| --- | --- | --- |
| `id` | yes | Unique, kebab-case (`a-z`, `0-9`, `-`). A user rule with a built-in's id replaces it. |
| `name` | yes | Shown as the item title (for `item = "group"`). |
| `category` | yes | `package-caches`, `browsers`, `ide`, `xcode`, `android`, `rust`, `docker`, `projects`, `logs`, `app-leftovers`, `unused-apps`, `large-items`, `claude-code`, `system`, `personal-data` or `other-large-folders`. Items in `personal-data` and `other-large-folders` are information only. |
| `description` | yes | Why the item is listed and what removing it costs. Shown as the item's reason. |
| `risk` | yes | `safe` (regenerated automatically, nothing lost), `caution` (costs time or bandwidth to get back) or `review` (may be the user's own data). |
| `detector` | yes | How items are found; see [Detectors](#detectors). |
| `requires` | no | Binaries that must be found on the `PATH`, e.g. `["npm"]`. The rule is skipped silently when one is missing. |
| `approximate` | no | `true` shows sizes with `~`, for files that share storage with others (APFS clones, hardlinks). |
| `preselect` | no | Overrides whether items are ticked by default. Only `safe` rules are ticked by default, and only `safe` rules may set `preselect = true`; `false` unticks a `safe` rule. |
| `platforms` | no | `["macos"]`, `["linux"]`, `["windows"]`. Defaults to every platform that has paths. |
| `[rule.owner]` | no | Apps or processes that must not run while the files are cleaned. |
| `[rule.vars]` | no | Values usable as `${name}` in paths. |
| `[rule.paths]` | for `paths` and `tool` | What to measure and delete. |
| `[rule.clean]` | for `tool` | The tool's own cleanup command. With a command, cleaning runs it instead of deleting the paths (for `paths` rules too). |
| `[rule.params]` | no | Detector-specific settings. |

Unknown keys are errors, so a typo never goes unnoticed.

### `[rule.paths]`

```toml
[rule.paths]
macos = ["~/Library/Caches/Foo", "~/.foo/versions/*"]
exclude = ["**/keep-me"]   # globs removed from the matches
item = "each"              # group (default): one item for all matches; each: one item per match
min_age_days = 90          # only matches last modified more than 90 days ago
keep_newest = 1            # per folder, keep the newest match (highest version number in the name, then newest)
```

Paths are globs (`*`, `**`, `{a,b}`, `[abc]`) and must start at an allowed root:

- `~/` or `${HOME}/`: your home folder.
- `${DARWIN_USER_CACHE_DIR}/`: the per-user cache folder (`/private/var/folders/…/C`).
- `${DARWIN_USER_DIR}/X/…` or `${DARWIN_USER_DIR}/C/<app-id>…`: only these two parts of the per-user folder. The temp folder `T` is never allowed.
- A rule variable, e.g. `${cache}/_cacache`.

They must name something inside the root (not `~` itself), can't contain `.` or `..`, and can never match a protected folder: `~/Documents`, `~/Desktop`, `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music`, `~/Public`, `~/Library` and its `Application Support`, `Caches`, `Containers`, `Group Containers` and `Preferences` folders themselves, or anything inside a `.git` folder, iCloud Drive, `~/Library/CloudStorage`, `~/Dropbox`, `~/Google Drive`, `~/Library/Keychains`, `~/.ssh` or `~/.gnupg`. These are checked when the rule loads and again right before deleting; your project roots themselves are refused right before deleting.

Matches are never followed through symlinks. With `item = "each"`, each item is titled by its path relative to the pattern's fixed part (e.g. `android-36/google_apis/arm64-v8a`) and its id is `<rule-id>:<path>`.

### `[rule.vars]`

```toml
[rule.vars]
cache = ["npm", "config", "get", "cache"]                    # a command's trimmed output
cargo = { env = "CARGO_HOME", default = "~/.cargo" }         # an environment variable, or the default
deno = { command = ["deno", "info", "--json"], json = "denoDir" } # a string field of a command's JSON output
```

Commands run without a shell, from your home folder. If one fails, the rule is skipped with a warning. A variable can only start a path, and its value must be an absolute path. As with every path, Reclaim deletes only inside your home folder.

### `[rule.clean]`

```toml
[rule.clean]
command = ["npm", "cache", "clean", "--force"]
fallback = "delete"   # when the binary can't be found: delete (the paths) or none (hide the rule)
timeout_secs = 600
```

`command` is an argument list. The first element must be a bare binary name, resolved on the `PATH` (Reclaim also looks in `/opt/homebrew/bin`, `~/.cargo/bin`, `~/.bun/bin`, `~/.local/bin`, `~/go/bin`, `~/.deno/bin` and your login shell's `PATH`); shells (`sh`, `bash`, `zsh`, …), `env` and `sudo` are refused. With `item = "each"`, `{name}` in the command is replaced by the match's file name, e.g. `["tart", "delete", "{name}"]`.

The paths are still measured for the size, and the command's output goes into the run log. A command that exits with a non-zero code fails the item.

### `[rule.owner]`

```toml
[rule.owner]
apps = ["com.google.Chrome"]      # bundle IDs of apps
processes = ["GradleDaemon"]      # process names
hint = "Run `gradle --stop` first" # shown instead of "Quit X to clean"
```

`processes` match a process's executable name, the base name of one of its arguments, or a Java main class ending (so `GradleDaemon` matches `org.gradle.launcher.daemon.bootstrap.GradleDaemon`); `*` works as a wildcard. Owners are checked at scan time and again right before cleaning.

## Detectors

| Detector | What it does |
| --- | --- |
| `paths` | Measures `[rule.paths]` and deletes the matches, or runs `[rule.clean] command` when it has one. |
| `tool` | Like `paths`, but requires `[rule.clean] command`. |
| `project-artifacts` | Build output inside git repos under the configured project roots. `[rule.params] names` lists folder names (globs); a folder counts only when git ignores it. |
| `app-leftovers` | Files of uninstalled apps in `~/Library` (`[rule.params] part = "data"` or `"logs"`). |
| `apps` | Installed apps with last use and size (the Unused apps view). |
| `large-folders` | Folders over the threshold that no other item covers. |
| `plugin:<name>` | A detector written in Rust for one special case. |

Built-in plugins: `code-sign-clones`, `jetbrains-versions`, `user-caches`, `simulators`, `swift-toolchains`, `android-avds`, `rustup`, `docker`, `colima`, `ollama`, `claude-transcripts`, `claude-cli-versions`, `agent-worktrees`. They are used by the built-in rules (see `reclaim rules show <id>` for their settings); a new plugin needs Rust code in `crates/reclaim-core/src/detect/plugins/`.

`paths` and `tool` rules can set `[rule.params] info_only = true` to show their items with their size and "Show in Finder" only, never selectable.

## When two rules find the same path

Every path belongs to one item. A deeper path wins over a folder that contains it: with a rule for `~/Library/Caches/pnpm`, the generic "Other caches" rule doesn't count that folder again. For the same path, the pattern with more fixed (non-wildcard) parts wins, then the rule loaded first. Reclaim's own folders are never part of any item.

If another rule finds a path inside a folder your rule names, your rule loses that whole folder (it isn't split around the other item). Name the subfolders you mean instead, e.g. `~/.cargo/registry` rather than `~/.cargo`.

App leftovers win over a rule that names the same folder: a folder named after the bundle ID of an app that isn't installed (e.g. `~/Library/Caches/com.example.MyTool`) shows up under App leftovers instead of your rule. Test such a rule with the app installed.

## Replacing and disabling built-in rules

Copy a built-in rule (`reclaim rules show <id>` prints it as TOML) into your rules folder and change it; a rule with the same `id` replaces the built-in entirely. To turn one off:

```toml
schema = 1

[[rule]]
id = "aerial-videos"
disabled = true
```

or add its id to `[rules] disabled` in `~/.config/reclaim/config.toml`. `reclaim rules list` shows each rule's source.

## Errors

`reclaim rules validate FILE...` prints every problem with the file, the rule id and the field, and exits with `1`:

```
error: my-tools.toml: rule `my-tool-cache`: `preselect`: only `safe` rules can be pre-ticked, this rule is `caution`
error: my-tools.toml: rule `my-tool-cache`: `paths.macos[1]`: must start with `~/`, `${HOME}/`, `${DARWIN_USER_DIR}/`, `${DARWIN_USER_CACHE_DIR}/` or a rule variable
error: my-tools.toml: rule `my-tool-cache`: `clean.command`: `sh` is not allowed: commands run without a shell
```

When the app or the CLI loads an invalid user rule, it skips that rule with a warning and keeps the others.
