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
kaigen activate KGEN-XXXX-XXXX-XXXX                   # the user's license key
kaigen doctor                                          # confirms everything
kaigen new mygame --template cubes
cd mygame && hz/hzbuild macos run                      # or: windows run
```

`kaigen new` installs the engine itself if the machine has none, so there is no
separate install step. `kaigen engine install` exists for pinning a specific
version or pre-warming a machine — `doctor` will report a missing engine as a
problem, but creating a project fixes it.

On Windows use `irm https://api.kaigen3d.com/install.ps1 | iex` instead of the
curl line.

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
{"activated":true,"licensee":"…","license_id":"…","expires":"YYYY-MM-DD","license_dir":"…"}
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
| `kaigen activate <KGEN key>` | licenses this machine (once per machine) |
| `kaigen doctor` | check everything, print fixes |
| `kaigen engine install [version]` | install an engine; no version = the project's pin, else latest |
| `kaigen engine list` | what is installed |
| `kaigen new <name>` | create a project, installing the engine if needed |
| `kaigen restore` | rebuild `./hz` in a freshly cloned project from its pin |

`kaigen new` flags: `--template <name>` (default `cubes`), `--engine <version>`,
`--embed` (copy the engine into the project instead of linking it), `--swift`
(macOS: nest the engine project under `app/` for a native SwiftUI host),
`--no-git`. **`new` initialises a git repo and makes an initial commit unless
you pass `--no-git`.**

Templates: `cubes`, `empty`, `platformer`, `obby`, `lasertag`, `procgen_candy`.

`kaigen activate` also accepts `--key <KGEN key>` instead of the positional, and
`kaigen engine activate --key …` is the same command under its older name.

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
`curl`. `KAIGEN_HOME` only affects where the CLI itself lives; the license and
the engines always follow `HOME`.

The binary identifies itself as `hzbuild` in its own usage text and log lines —
`kaigen` and `hzbuild` are the same program. Where the two disagree, this
document and `kaigen --help` describe the `kaigen` spelling.

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

Inside a project, build with `hz/hzbuild`, not `kaigen`. Same binary, but
`hz/hzbuild` is the one pinned to that project's engine.

## Troubleshooting

| symptom | cause | fix |
|---|---|---|
| `no valid hzlicense.hzt` | machine not activated | `kaigen activate <key>` |
| `restore: no hzproject.hzt` | wrong directory | `cd` to the project root |
| `./hz is missing` | fresh clone | `kaigen restore` |
| `no native compiler` | toolchain absent | `hz/hzbuild install --accept-license` |
| `no metal compiler` (macOS) | Command Line Tools only | install full Xcode, then `sudo xcode-select -s /Applications/Xcode.app` |
| activation exits 2 | key unknown, revoked, expired, or at its machine cap | ask the user; you cannot fix this |
| `command not found: kaigen` | not on PATH | `export PATH="$HOME/.kaigen/bin:$PATH"` |

## Rules

- Never edit files under the engine install (`~/Library/Application Support/hz/engines/…`
  on macOS, `%LOCALAPPDATA%\hz\engines\…` on Windows). It is a read-only drop
  shared by every project.
- Never commit the `hz` symlink or `hzlicense.hzt`.
- Do not hand-edit `hzproject.hzt`'s `engine_version` — use `kaigen restore` or
  `kaigen new --engine <version>`.
- If a build fails after switching engine versions, delete `cooked/` and rebuild;
  cooked assets go stale when the engine's structs change.
