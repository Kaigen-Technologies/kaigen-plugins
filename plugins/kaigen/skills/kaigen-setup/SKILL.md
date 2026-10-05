---
name: kaigen-setup
description: Install the Kaigen 3D engine CLI, activate a license key, and create or repair a Kaigen project. Use when asked to set up Kaigen, install the engine, activate a KGEN key, start a new Kaigen game/app, or when a kaigen or hzbuild command fails.
---

# Kaigen setup

Kaigen is a high performance 3D realtime engine written in modern C. `kaigen` is its CLI: it installs the engine, activates
a license, and scaffolds projects. macOS and Windows only — there is no Linux
build.

This skill covers **getting a machine working**. Once a project exists, the
engine ships its own skills under `<project>/.claude/skills` — read those for
gameplay, rendering, shaders and build details.

## The whole setup

```bash
curl -fsSL https://api.kaigen3d.com/install.sh | sh   # installs ~/.kaigen/bin/kaigen
export PATH="$HOME/.kaigen/bin:$PATH"
kaigen activate KGEN-XXXX-XXXX-XXXX --accept-terms    # the user's key; see "Terms" below
kaigen doctor                                          # confirms everything
kaigen engine install                                  # the latest engine
kaigen new mygame --template starting-template
cd mygame && hz/hzbuild macos run                      # or: windows run
```

`kaigen new` and `kaigen templates` use the newest engine installed on this
machine and never ask the server which one; with none installed they stop with
`no engine installed — run: kaigen engine install`. `kaigen engine install` with
no version fetches the latest. `--engine <version>` (installed first if needed)
and a project's pin still win.

On Windows the equivalent is:

```powershell
irm https://api.kaigen3d.com/install.ps1 | iex   # installs %LOCALAPPDATA%\kaigen\bin
[Environment]::SetEnvironmentVariable('PATH', [Environment]::GetEnvironmentVariable('PATH','User') + ';' + "$env:LOCALAPPDATA\kaigen\bin", 'User')
kaigen activate KGEN-XXXX-XXXX-XXXX --accept-terms
kaigen doctor
kaigen engine install
kaigen new mygame --template starting-template
cd mygame; hz\hzbuild windows run
```

Use `SetEnvironmentVariable`, never `setx`: `setx` truncates at 1024 characters
and writes the expanded user+system PATH into the user PATH.

**You cannot obtain a license key.** It comes from the user or from Kaigen. If
there is no key, stop and ask for one — do not guess, and do not try to work
around activation.

## Start by running doctor

`kaigen doctor` checks the license, installed engines, the current project and
the toolchain. Every failure prints the exact command that fixes it. When
anything is wrong, run `doctor` first and do what it says rather than
diagnosing yourself.

```bash
kaigen doctor --json
```

```json
{"checks":[{"area":"license","status":"fail","detail":"this machine is not activated",
            "remediation":"kaigen activate <your KGEN key>"}],
 "problems":1,"ok":false}
```

Read `remediation` and run it. That field is always a literal command.

## Machine-readable mode

`--json` is supported by exactly these commands, and nowhere else:
`version`, `doctor`, `engine list`, `activate`. It may go anywhere on the
line. In JSON mode stdout carries exactly one JSON document and nothing else,
so it is safe to pipe into a parser.

**Every other command prints human log lines** — `[T0: INFO 0.0s] file.c:123: …`
plus, on build commands, a timings block. Never parse those; use the exit code.

Success shapes:

```json
// version
{"version":"X.Y.Z","platform":"macos","activate_url":"https://api.kaigen3d.com/v1/activate"}
// doctor
{"checks":[{"area":"license","status":"ok|fail","detail":"…","remediation":"…"}],
 "problems":0,"ok":true}
// engine list
{"engines":[{"version":"…","id":"…","edition":"binary|source","platform":"macos",
             "size_bytes":167849039}],"engines_dir":"…"}
// activate
{"activated":true,"licensee":"…","license_id":"…","expires":"YYYY-MM-DD","license_dir":"…","terms_version":"…"}
```

Failure is always one shape:

```json
{"error":{"code":"key_unknown","message":"key unknown or revoked",
          "remediation":"check the key, or request a new one"}}
```

`code` is one of: `missing_key`, `malformed_key`, `key_unknown`,
`activation_limit`, `license_expired`, `activation_failed`.

Exit codes:

| code | meaning | what to do |
|---|---|---|
| 0 | success | continue |
| 1 | something failed | read the error, retry if it looks transient |
| 2 | license problem | the user must fix or supply a key — ask them |
| 3 | bad input, or an unknown command | fix the argument or the command name |

Exit 2 is never something you can solve by retrying. Exit 3 also covers an
unrecognised subcommand, so **always check the exit code** — a usage dump is a
failure, not a success.

`doctor` follows the same table: exit 2 when the license is the problem, exit 1
when something else is, 0 when clean. None of `doctor`'s failures are transient,
so never retry it — act on `remediation` instead.

`curl … install.sh | sh` exiting 0 means the binary was downloaded and verified,
**not** that it is on your PATH. Export the PATH line it prints (it does not
edit your shell profile) and confirm with `kaigen version` before continuing.

## Commands

| command | what it does |
|---|---|
| `kaigen version` | CLI version and the backend it targets |
| `kaigen activate <KGEN key> --accept-terms` | licenses this machine (once per machine), accepting Kaigen's terms |
| `kaigen doctor` | check everything, print fixes |
| `kaigen self-update` | update this `kaigen` to the latest release |
| `kaigen engine install [version]` | install an engine; no version = the project's pin, else latest |
| `kaigen engine list` | what is installed, and which components each install has |
| `kaigen engine add <component>` | add optional platform support (`ios`, `wasm`, `tests`) |
| `kaigen engine remove <component>` | remove an optional component |
| `kaigen templates` | the templates available for the newest installed engine (or the project's pin), and which are downloaded |
| `kaigen templates remove <name>` | delete a downloaded template (it downloads again on the next `new`) |
| `kaigen new <name>` | create a project with the newest installed engine |
| `kaigen restore` | rebuild `./hz` in a freshly cloned project from its pin |
| `kaigen use --status` | what engine this project points at, and whether it is a drop or a source tree |
| `kaigen use <version>` | point `./hz` at an installed version (restores the pin) |
| `kaigen use source [<path>]` | point `./hz` at a local engine checkout — **source licences only** |

`kaigen new` flags: `--template <name>` (default `cubes`), `--engine <version>`,
`--embed` (copy the engine into the project instead of linking it), `--swift`
(macOS: nest the engine project under `app/` for a native SwiftUI host),
`--no-git`. **`new` initialises a git repo and makes an initial commit unless
you pass `--no-git`.**

Templates are downloaded on demand for the engine version in use: `new` fetches a
missing one itself (it needs the network and an activated license the first
time). `kaigen templates` lists what exists; an unknown `--template` name prints
the same list.

### Updates

`kaigen self-update` downloads the latest release, checks it against the signed
catalog, and replaces the binary in place (`--json`: `{"updated":true,"from":…,
"to":…}`, or `{"updated":false,"version":…,"latest":…}` when already current).
Once a day, at the end of a command, `kaigen` checks for a newer release and
prints one line on stderr when there is one (never in `--json` mode). When the
installed version is below the minimum Kaigen supports, every command except
`version`, `help` and `self-update` fails with `update_required` (exit 1,
remediation `kaigen self-update`). Codes: `update_check_failed` and
`download_failed` (network, retry), `needs_manual` (the binary's folder is not
writable: run the remediation, a reinstall), `engine_copy` (exit 3: this is a
project's `hz/hzbuild`, which updates with its engine; use the installed
`kaigen`). `doctor` reports it as the `cli` check.

`kaigen activate` also accepts `--key <KGEN key>` instead of the positional, and
`kaigen engine activate --key …` is the same command under its older name.

### Terms

Activating accepts Kaigen's Terms of Service, Privacy Policy and Closed Beta
Agreement. Without `--accept-terms`, `kaigen activate` changes nothing and
answers with the three links (`--json`: `{"error":{"code":"terms_required",
"remediation":"kaigen activate <key> --accept-terms","terms_version":…,
"terms_url":…,"privacy_url":…,"beta_url":…}}`, exit 3).

**Never pass `--accept-terms` on the user's behalf without asking.** Show them
the three links from that answer, and run the remediation only after they say
they agree. `terms_changed` (exit 3) means new terms were published while
activating: show the new links and ask again. `terms_unavailable` (exit 1) is a
network failure: retry.

## Platform support is a separate download

An engine install starts with only the `core` component, which builds for the
host desktop platform. **iOS and web are optional components and are NOT
installed by default** — if a build for one of them fails on a fresh machine,
this is why, and it is a download, not a code problem.

```bash
kaigen engine add ios      # iOS support
kaigen engine add wasm     # web/browser support
kaigen engine add tests    # the engine test suite
kaigen engine remove wasm  # frees the disk again
```

Run these from inside the project and the version comes from its pin. Outside a
project, pass `--version <v>` or it cannot tell which install you mean.

`kaigen engine list` shows what each install has:

```
engine: 0.5.8.006-internal (binary, macos, 120.48 MB) — components: core
engine: 0.5.8.006-internal (binary, macos, 132.98 MB) — components: core+ios
```

Rules the CLI enforces, so do not fight them:

- `core` is required — `add core` and `remove core` both fail by design.
- An unknown name fails with `no <name> component published for <version>`;
  the real names are `core`, `ios`, `wasm`, `tests`.
- Components are per engine *install*, not per project. Adding `ios` to a
  version benefits every project pinned to it.

## Pointing a project at an engine source tree

**Source licences only.** A binary-edition machine has no engine source, so
`use source` there leaves the project unbuildable. If unsure, ask the user which
licence they hold rather than guessing.

```bash
kaigen use source /path/to/kaigen-engine
kaigen use --status
kaigen use 0.6.0            # back to an installed version
```

- `use source` **removes `engine_version` from `hzproject.hzt`** — the engine is
  then whatever the checkout is at, and `kaigen restore` no longer applies. Do
  not re-add the pin by hand; `kaigen use <version>` restores it.
- `use --status` prints the mode and an identity string containing the engine's
  git sha. Use it before assuming which engine a build came from.
- Switching between a source tree and a drop changes the engine identity and
  forces a full rebuild. That is expected — do not go hunting for a cause.

## Where things live

| what | macOS | Windows |
|---|---|---|
| the CLI | `~/.kaigen/bin/kaigen` | `%LOCALAPPDATA%\kaigen\bin\kaigen.exe` |
| license | `~/Library/Application Support/hz/hzlicense.hzt` | `%LOCALAPPDATA%\hz\hzlicense.hzt` |
| engines | `~/Library/Application Support/hz/engines/<version>/` | `%LOCALAPPDATA%\hz\engines\<version>\` |

`KAIGEN_HOME` relocates the CLI (it installs to `$KAIGEN_HOME/bin`) — useful in
CI and containers. It has to reach the `sh` on the right-hand side of the pipe,
not `curl`, so put it there:

```bash
curl -fsSL https://api.kaigen3d.com/install.sh | KAIGEN_HOME=/opt/kaigen sh
```

`KAIGEN_HOME=… curl … | sh` silently does nothing — it sets the variable for
`curl`. On Windows there is no pipe-side to get wrong; set it in the session
before running the installer:

```powershell
$env:KAIGEN_HOME = "D:\kaigen"; irm https://api.kaigen3d.com/install.ps1 | iex
```

`KAIGEN_HOME` only affects where the CLI itself lives; the license and the
engines always follow `HOME` (macOS) / `%LOCALAPPDATA%` (Windows).

The binary identifies itself as `hzbuild` in its own usage text and log lines —
`kaigen` and `hzbuild` are the same program. Where the two disagree, this
document and `kaigen --help` describe the `kaigen` spelling.

## Keeping this current

`kaigen self-update` updates the CLI; this document updates separately. If a
command here does not match what the binary does, the binary wins — check
`kaigen --help`, and tell the user their copy of this skill is stale.

- Claude Code plugin: `/plugin marketplace update`, then `/plugin install kaigen@kaigen`
- Codex plugin: `codex plugin marketplace upgrade`
- skills CLI: `npx skills add https://api.kaigen3d.com/skill.md -g`
- the current version is always at <https://api.kaigen3d.com/skill.md>

## Inside a project

A project pins its engine version in `hzproject.hzt` and reaches it through a
`hz` symlink, which is machine-local and therefore **not** committed. After a
fresh `git clone`:

```bash
cd <project>
kaigen restore          # recreates ./hz, installing the pinned engine if missing
hz/hzbuild install --accept-license   # first machine only: downloads the toolchain
hz/hzbuild macos run
```

Building for iOS or web needs its engine component first — `kaigen engine add
ios` / `kaigen engine add wasm`. See "Platform support is a separate download".

Inside a project, build with `hz/hzbuild`, not `kaigen`. Same binary, but
`hz/hzbuild` is the one pinned to that project's engine.

## Troubleshooting

| symptom | cause | fix |
|---|---|---|
| `no valid hzlicense.hzt` | machine not activated | `kaigen activate <key>` |
| `no engine installed — run: kaigen engine install` | `new` / `templates` found no engine on this machine | `kaigen engine install` |
| `restore: no hzproject.hzt` | wrong directory | `cd` to the project root |
| `./hz is missing` | fresh clone | `kaigen restore` |
| `./hz` points at the wrong engine | switched versions or trees | `kaigen use --status`, then `kaigen use <version>` or `kaigen use source <path>` |
| `no native compiler` | toolchain absent | `hz/hzbuild install --accept-license` |
| `no metal compiler` (macOS) | Command Line Tools only | install full Xcode, then `sudo xcode-select -s /Applications/Xcode.app` |
| activation exits 2 | key unknown, revoked, expired, or at its machine cap | ask the user; you cannot fix this |
| `command not found: kaigen` | not on PATH | `export PATH="$HOME/.kaigen/bin:$PATH"` |
| `update_required` | this kaigen is below the supported minimum | `kaigen self-update` |
| `license_missing` (exit 2) | no license on this machine: build/run/publish need one | ask the user for their key; `kaigen activate <key>` |
| `license_invalid` (exit 2) | the license file is damaged or was modified | `kaigen activate <key>` again |
| `license_expired` (exit 2) | the license is past its date | the user renews it with Kaigen, then `kaigen activate <key>` |
| `license_unconfirmed` (exit 2) | offline more than 30 days: build/run/publish stop | connect to the internet, run the same command again |
| `license_revoked` / `license_inactive` (exit 2) | the license was revoked or deactivated | ask the user; they need a valid key (`kaigen activate <key>`) |
| `license: not confirmed … within N day(s)` warning | nearing the 30-day offline limit | connect to the internet once; any kaigen command confirms it |

## When Kaigen itself is broken: offer a report

If an error is Kaigen's, not the project's (a crash in engine frames, a failure inside
the engine, an engine build error the project did not cause, a `kaigen` command that
fails for no reason you can fix) and you cannot get past it, tell the user and ASK
before filing — never file on your own:

```bash
kaigen report "what failed, how to reproduce, your diagnosis" --json            # anywhere
hz/hzbuild report "…" --project --json                                          # inside a project
```

It sends the Kaigen team the description plus versions, system info, the last crash and
(with `--project`) the project's `hzproject.hzt` and recent session logs — never source
code; home-folder paths are replaced by `~` — and prints `{"report_id":"KR-7F3A-91C2",…,"status":"sent"|"queued"}`. Give the
user the `KR-` id; `queued` means offline and it goes out automatically. Write the
diagnosis as facts (the error, the frame, what you tried); never put secrets or the
user's private data in it.

## Rules

- Never edit files under the engine install (`~/Library/Application Support/hz/engines/…`
  on macOS, `%LOCALAPPDATA%\hz\engines\…` on Windows). It is a read-only drop
  shared by every project.
- Never commit the `hz` symlink or `hzlicense.hzt`.
- Do not hand-edit `hzproject.hzt`'s `engine_version` — use `kaigen restore` or
  `kaigen new --engine <version>`.
- If a build fails after switching engine versions, delete `cooked/` and rebuild;
  cooked assets go stale when the engine's structs change.
