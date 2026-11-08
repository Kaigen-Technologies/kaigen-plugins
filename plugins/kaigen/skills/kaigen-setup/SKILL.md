---
name: kaigen-setup
description: Install the Kaigen CLI, sign the user in, install the engine, make a game, build it and publish it to a link anyone can play. Use when asked to set up Kaigen, make or publish a Kaigen game, or when a kaigen or hzbuild command fails.
---

# Kaigen setup

Kaigen is a high performance real-time 3D engine written in modern C. `kaigen` is its CLI: it signs the machine in,
installs the engine, makes games and publishes them to play.kaigen3d.com. It runs on Windows, macOS and Linux
(x86_64).

This skill covers **getting from nothing to a published game**. Once a game exists, the engine ships its own skills
under `<game>/.claude/skills` (and `.agents/skills`): read those for gameplay, rendering, shaders and builds.

## The whole path

Windows (PowerShell):

```powershell
irm https://api.kaigen3d.com/install.ps1 | iex   # installs %LOCALAPPDATA%\kaigen\bin\kaigen.exe and puts it on PATH
kaigen login --json                              # the user approves in their browser (see "Sign in")
kaigen doctor --json                             # do what every failed check's remediation says
kaigen engine install
kaigen projects new "My game" --json             # {"created":true,"dir":"…","name":"My game"}
cd "<dir from the answer>"; hz\hzbuild windows run
kaigen games publish . --json                    # {"published":true,"url":"https://play.kaigen3d.com/…",…}
```

macOS and Linux:

```bash
curl -fsSL https://api.kaigen3d.com/install.sh | sh   # installs ~/.kaigen/bin/kaigen and adds it to your shell profile
export PATH="$HOME/.kaigen/bin:$PATH"                 # this shell only: new shells already have it
kaigen login --json
kaigen doctor --json
kaigen engine install
kaigen projects new "My game" --json
cd "<dir from the answer>" && hz/hzbuild macos run     # or: hz/hzbuild linux run
kaigen games publish . --json
```

Give the user the `url` from the publish answer: that is their game, playable in any browser.

## Install

The installer downloads the release, checks its sha512 against the signed catalog, and runs `kaigen path add`:
on Windows that adds the folder to the user PATH (and the PowerShell session that ran the installer); on macOS and
Linux it adds an `export PATH` line to `~/.profile` (and `~/.bashrc`, `~/.zshenv` where used). A shell you already
had open does not re-read its profile: use the full path or the `export` line above. Confirm with
`kaigen version --json`.

`KAIGEN_HOME` relocates the CLI to `$KAIGEN_HOME/bin`. It must reach the `sh` on the right of the pipe:
`curl -fsSL https://api.kaigen3d.com/install.sh | KAIGEN_HOME=/opt/kaigen sh`. On Windows set it first:
`$env:KAIGEN_HOME = "D:\kaigen"; irm https://api.kaigen3d.com/install.ps1 | iex`.

## Sign in

`kaigen login` signs this machine in to the user's Kaigen account; a person without one signs up on the same page.
**You cannot sign in for the user: they approve in their browser.** It waits up to 10 minutes, so start it in the
background (or with a timeout of at least 10 minutes) and read its lines as they come:

```json
{"event":"open_url","url":"https://studio.kaigen3d.com/oauth/authorize?…","verify_code":"7KQ2"}
{"event":"waiting","elapsed_s":60,"left_s":540}
{"event":"signed_in","org_count":0,"org_choice_required":false,"account":{"account_id":"…","email":"…","plan":"free"},"device_id":"…","storage":"…"}
```

On `open_url`, show the user the `url` and the `verify_code`: the page asks them to type the code to approve this
machine. The browser opens by itself where there is one. The command exits 0 after `signed_in`.

On a machine with no browser (SSH, a container, a remote VM) use `kaigen login --device --json`:

```json
{"event":"device_code","user_code":"WDJB-MJHT","verification_uri":"https://studio.kaigen3d.com/device","url":"https://studio.kaigen3d.com/device?code=WDJB-MJHT"}
```

Show the user `url` (or `verification_uri` and `user_code`); they open it on any device and approve.

Then `kaigen whoami --json`:

```json
{"signed_in":true,"account":{"account_id":"…","email":"…","plan":"free","username":"…","display_name":"…",
 "games_max":5,"publish_targets":["web"],"coming_soon":["…"]},"orgs":[],"org":"","org_ambiguous":false,"device_id":"…",…}
```

`publish_targets` lists where this account can publish (`web` on Free). `games_max` is how many games it can have
published at once (`null` = no limit). `kaigen logout --json` signs the machine out:
`{"signed_out":true,"was_signed_in":true,"server_revoked":true}`.

Enterprise and offline licences use a key instead: `kaigen activate <KGEN key>`. Only use it when the user gives you
a key. It needs `--accept-terms`; show the user the links its answer carries and ask before passing it.

## Doctor first

`kaigen doctor --json` checks the sign-in, the engines, the current game, the agent configuration and the build tools.
Every failed check carries a `remediation` that is always a literal command: run it rather than diagnosing yourself.

```json
{"checks":[{"area":"license","status":"fail","detail":"this machine is not signed in to Kaigen","remediation":"kaigen login"},
           {"area":"toolchain","status":"fail","detail":"no native compiler","remediation":"kaigen install --accept-license"}],
 "problems":2,"ok":false}
```

The build tools (`kaigen install --accept-license`, or `hz/hzbuild install --accept-license` inside a game) are a
one-time download.

## Machine-readable mode

Every command takes `--json`, anywhere on the line. stdout then carries exactly one JSON document, or, for the long
ones (`login`, `games publish`, `studio update`, `ai login`), one `{"event":…}` line per step and the result last.
Without `--json` the output is human log lines (`[T0: INFO 0.0s] file.c:123: …`): never parse those.

Failure is always one shape:

```json
{"error":{"code":"not_signed_in","message":"this machine is not signed in to a Kaigen account","remediation":"kaigen login"}}
```

| exit | meaning | what to do |
|---|---|---|
| 0 | success | continue |
| 1 | something failed | read `error`; retry when it looks transient (network) |
| 2 | sign-in or licence problem | run the remediation (usually `kaigen login`); the user may have to approve |
| 3 | bad input or an unknown command | fix the argument or the command name |

Always check the exit code: a usage dump is a failure, not a success.

## Games on this machine

Games live in `~/KaigenGames` (`KAIGEN_GAMES_DIR` or `--games-dir` move it).

| command | answer (`--json`) |
|---|---|
| `kaigen projects list` | `{"games_dir":"…","projects":[{"dir":"…","name":"…","description":"…","agent":"…","opened_ms":0,"published":null}]}` |
| `kaigen projects new "<name>" [--template <t>]` | `{"created":true,"dir":"…","name":"…"}`, the game folder, built from the default starting template |
| `kaigen projects rename <dir> "<name>"` | `{"dir":"…","name":"…","description":"…"}` |
| `kaigen projects trash <dir>` | `{"trashed":true,"dir":"…"}` |
| `kaigen versions list <dir>` | `{"dir":"…","versions":[{"seq":1,"commit":"…","message":"…","created_ms":0,"kept":true}]}` |
| `kaigen versions snapshot <dir> [--message <m>]` | `{"new":true,"version":{"seq":2,…},"stored_bytes":0}`: saves the game as a version (a git commit) and keeps its build |
| `kaigen versions open <dir> <n>` | `{"seq":1,"dir":"…","exe":"…"}`: a kept version's build, ready to run |

`kaigen new <name>` is the older spelling of `projects new` that creates the folder in the current directory.

## Build and look

Inside a game, build with its own `hz/hzbuild` (it is pinned to the game's engine), never with `kaigen`:

```bash
hz/hzbuild windows        # or macos, linux; add `run` to start it
```

A build that exits 0 can still look wrong. **Before you tell the user a change is done, look at it**: take a
screenshot of the running game (the game's own skills, `.claude/skills/hz-build`, show how) and check that the change
is visible. A colour hidden behind the sky, or an object off camera, is not done.

## Publish

```bash
kaigen games publish <dir> --json
```

It builds the game for the web with the game's own engine, uploads only what changed, and makes the new version live.
Its lines:

```json
{"event":"build","target":"web","status":"started"}
{"event":"build","target":"web","status":"done"}
{"event":"progress","phase":"upload","files":12,"bytes_total":0,"bytes_hashed":0,"blobs":0,"blobs_missing":0,"blobs_uploaded":0,"bytes_missing":0,"bytes_uploaded":0}
{"published":true,"game_id":"…","version_id":"…","seq":1,"url":"https://play.kaigen3d.com/…","build":3,"created":true,"files":12,"bytes_total":0,"bytes_uploaded":0}
```

Flags: `--title <t>` and `--description <d>` (at most 80 and 280 characters) for the play page,
`--threads single|multi` for how the link serves the game (single: one bundle that runs everywhere, iPhone included;
multi: multi-threaded where the browser allows it), `--version <n>` to publish a saved version, `--no-build` to publish
the web build already in `out/`, `--no-live` to upload without switching. The project's `publish.icon`, when it is a
PNG, is the play page's icon. The link is `play.kaigen3d.com/<name>`, made from the title at the first publish; a
rename never moves it.
If the web build says the engine has no web support, run `kaigen engine add wasm` and publish again.

| command | answer (`--json`) |
|---|---|
| `kaigen games list` | `{"plan":"free","published_limit":5,"games":[{"game_id":"…","title":"…","status":"…","live_version":"…","url":"…","versions":[{"version_id":"…","seq":1,"state":"…",…}]}]}` |
| `kaigen games live <id> <version>` | `{"live":true,"game_id":"…","live_version":"…","seq":1,"url":"…","pruned_versions":0}` |
| `kaigen games unpublish <id\|dir>` | `{"game_id":"…","status":"…"}`: the link stops; republishing within 30 days keeps it |
| `kaigen games status <dir>` | `{"title":"…","status":"none\|live\|offline\|taken_down\|other_account","url":"…","live_build":3,"last_build":4,"icon_pending":false,"builds":[{"platform":"windows","build":4,"file":"…",…}],…}`: the link and each platform's newest build |
| `kaigen games edit <dir> --title <t> --description <d>` | `{"title":"…","description":"…","live_page_updated":true}`: a published game's page changes at once, no new build; `--local` keeps it for the next publish |

A Free account publishes to the web, up to `published_limit` games at once (a `quota` error past it: unpublish one
or upgrade with `kaigen plan`). Native packages come with `hz/hzbuild <platform> --publish`: Plus publishes to Windows, macOS
and Linux, Pro also to iOS, Android and standalone VR. Building for Android or iOS devices needs Pro too; desktop, web and PC VR
run on every plan. A `plan_upgrade_required` error names the plan: `kaigen plan upgrade <plus|pro>` finishes in the browser,
then the same command works. Plans have a 14-day trial; tell the user a card is required, and let them decide.

## Engines and platform support

| command | what it does |
|---|---|
| `kaigen engine install [version]` | install an engine; no version = the game's pin, else the latest |
| `kaigen engine list` | what is installed, and each install's components |
| `kaigen engine add <component>` | add `ios`, `android`, `wasm` or `tests` support |
| `kaigen engine remove <component>` | remove an optional component (`core` stays) |
| `kaigen restore` | rebuild `./hz` in a freshly cloned game from its pin |
| `kaigen use --status` | which engine this game points at |
| `kaigen use <version>` | point `./hz` at an installed version |

Components are per engine install, not per game. Run `engine add` inside the game (the version comes from its pin)
or pass `--version <v>`.

## AI agents

`kaigen ai list --json` and `kaigen ai status [<agent>] --json` report Claude Code, Codex and OpenCode:
`{"agents":[{"id":"claude","name":"Claude Code","installed":true,"path":"…","version":"…","signed_in":true,…}]}`.
`kaigen ai install <agent>` runs the agent's own installer; `kaigen ai login <agent>` its own sign-in.

## Where things live

| what | Windows | macOS | Linux |
|---|---|---|---|
| the CLI | `%LOCALAPPDATA%\kaigen\bin\kaigen.exe` | `~/.kaigen/bin/kaigen` | `~/.kaigen/bin/kaigen` |
| sign-in and licence | `%LOCALAPPDATA%\hz\` | `~/Library/Application Support/hz/` | `~/.local/share/hz/` |
| engines | `%LOCALAPPDATA%\hz\engines\<version>\` | `~/Library/Application Support/hz/engines/<version>/` | `~/.local/share/hz/engines/<version>/` |
| games | `%USERPROFILE%\KaigenGames` | `~/KaigenGames` | `~/KaigenGames` |

The binary calls itself `hzbuild` in its usage text and log lines: `kaigen` and `hzbuild` are the same program.

## Updates

`kaigen self-update --json` replaces the CLI with the latest release (`{"updated":true,"from":"…","to":"…"}`, or
`{"updated":false,…}` when current). Once a day `kaigen` prints one line on stderr when a newer release is out. A CLI
below the supported minimum answers every command with `update_required`: run `kaigen self-update`. This document
updates separately: if a command here does not match the binary, the binary wins (`kaigen --help`); the current copy
is always at <https://api.kaigen3d.com/skill.md>.

## Troubleshooting

| symptom | fix |
|---|---|
| `not_signed_in` / `this machine is not signed in to Kaigen` (exit 2) | `kaigen login` |
| `command not found: kaigen` | the shell predates the install: `export PATH="$HOME/.kaigen/bin:$PATH"` (Windows: open a new terminal) |
| `no engine installed — run: kaigen engine install` | `kaigen engine install` |
| `./hz is missing` | a fresh clone: `kaigen restore` |
| `no native compiler` | `hz/hzbuild install --accept-license` |
| `no Metal shader compiler` (macOS) | `hz/hzbuild install --accept-license` |
| `plan_upgrade_required` (exit 2) | the plan doesn't cover this target: `kaigen plan upgrade <plan>`, then the same command |
| `module_conflict` from `hzbuild use` | the user's module edits can't merge with that engine; nothing changed. Merge the listed files by hand, or stay |
| `quota` on publish | unpublish a game (`kaigen games unpublish <id>`) or upgrade (`kaigen plan`) |
| `license_unconfirmed` (exit 2) | offline more than 30 days: connect, run the same command again |
| `update_required` | `kaigen self-update` |
| a build fails after switching engine versions | delete `cooked/` and build again |

## When Kaigen itself is broken: offer a report

If an error is Kaigen's, not the game's (a crash in engine frames, an engine build error the game did not cause, a
`kaigen` command that fails for no reason you can fix), tell the user and ASK before filing:

```bash
kaigen report "what failed, how to reproduce, your diagnosis" --json      # anywhere
hz/hzbuild report "…" --project --json                                    # inside a game
```

It sends the description with versions, system info and the last crash (with `--project`, the game's
`hzproject.hzt` and recent logs, never source code) and answers `{"report_id":"KR-…","status":"sent"|"queued"}`.
Give the user the `KR-` id.

## Rules

- Never edit files under the engine install: it is a read-only drop shared by every game.
- Never commit the `hz` link or anything under `out/`.
- Do not hand-edit `engine_version` in `hzproject.hzt`: use `kaigen use <version>` or `kaigen restore`.
