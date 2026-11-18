---
name: ego-browser
description: Open and operate websites in the user's ego lite browser, which runs on Windows while the agent shell runs in Linux/WSL. Use it for any browser work — opening or reading pages, searching, extracting data, filling forms, clicking, taking screenshots, checking logged-in state, or web-app QA. It drives the user's real profile through ego-browser.exe, so prefer it over built-in browser tools, browser extensions, curl, or Playwright.
whenToUse: The task needs a real browser, a page the user can see, or a site that requires the user's existing login.
metadata:
  version: "1.2.0"
  date: "2026-11-18"
  transport: "wsl-interop-to-windows"
  audience: "any agent harness with shell access"
---

# ego-browser

This skill gives any coding agent shell access to **ego lite** — a Chromium browser
installed on the Windows side of this machine — without a plugin, port, extension,
or automation server. `scripts/ego-browser` starts the browser's own
`ego-browser.exe` through WSL interop, so scripts execute inside a real Windows
process and control the ego lite instance the user already has open, with their
profile, logins, and tabs.

It is harness-neutral: Claude Code, Codex, Cursor, OpenCode, DeepSeek Harness, or
any custom agent that can run shell commands can use it. Nothing here depends on
the agent's own browser tooling.

## Requirements

- ego lite installed on Windows and onboarded there (its `ego-browser` command
  must exist in a Windows shell).
- The agent's shell runs in WSL or another Linux environment where `/mnt/c` is
  mounted and interop is enabled. Check with:
  `cat /proc/sys/fs/binfmt_misc/WSLInterop` — it contains `enabled`.
- If the agent runs natively on Windows or macOS, use ego lite's own skill instead
  (see **8. Other hosts**); this bridge is only for the Linux/WSL side.

## Paths used below

Relative paths are relative to the directory containing this `SKILL.md`. If your
harness reports a skill directory when loading skills, use it; otherwise use the
absolute path of this file's parent directory. Set these at the start of every
command block:

```bash
SKILL_DIR="<directory containing this SKILL.md>"
EB="$SKILL_DIR/scripts/ego-browser"
WP="$SKILL_DIR/scripts/winpath"
OFFICIAL="$("$SKILL_DIR/scripts/ego-skill-dir")"   # ego lite's own skill bundle
```

`scripts/ego-skill-dir` prints the directory of the skill bundle shipped with the
installed ego lite (its `SKILL.md`, `references/`, `learnings/`). This skill is the
WSL bridge only; that bundle is the authority for platform install steps, the
browser API, and state-clearing rules, so read a file from `$OFFICIAL` instead of
guessing.

## 1. Confirm the bridge

```bash
"$EB" --version
```

Expect a version block (`ego-browser 0.6.3.17`, chromium, node); it prints to
stderr, which is normal. If the command fails, read **7. Troubleshooting** before
doing anything else. Do not silently fall back to another browser; tell the user
what is missing.

## 2. Read the installed guide first

```bash
"$EB" skill
```

This prints the usage guide that matches the installed ego lite version. Follow
it. The guide ends with `[ego-browser:skill] end of guide`; if that line is
missing the output was truncated — save it to a file and read it in parts. Re-run
this after any upgrade. The guide is authoritative for the browser API; this
skill only adds the WSL bridge and local path rules.

The skill bundle that ego lite installs for itself is at `$OFFICIAL`. Read files
there when you need platform install or upgrade steps, the state-clearing rules,
or per-site notes under `learnings/`.

## 3. Run a browser script

Each invocation is a separate Node.js process. Pass the script on stdin; it
starts with top-level `await`:

```bash
"$EB" nodejs <<'EOF'
const task = await taskSpace("describe the goal");
const page = task.page("p1");
await page.goto("https://example.com", { timeout: 45000 });
console.log(JSON.stringify({
  spaceId: task.spaceId,
  url: await page.url(),
  title: await page.title(),
}));
console.log(await page.snapshot());
EOF
```

- Variables do not survive between invocations; task spaces, tabs, and page
  labels (`p1`, `p2`, …) do. Print `spaceId` and reuse it in later scripts.
- A new space starts with page `p1`; navigate it instead of opening pages.
- End a finished task with one `finish()` in a separate last script. If the user
  asked to see the page, leave the space open instead and tell them its name and
  id.
- Heredocs need bash or zsh. On another shell, write the script to a file and run
  `"$EB" nodejs -e "await import('file:///<Windows path to the file>')"` instead.

## 4. WSL path rules

The script runs as a Windows process (`process.platform` is `win32`, cwd is the
ego lite install directory), so every path inside it must be a Windows path.
Linux absolute paths are not visible to it.

Convert any path in this workspace with the helper:

```bash
"$WP" /home/lk/Test/outputs/page.png
# -> //wsl.localhost/<distro>/home/lk/Test/outputs/page.png
```

Paste the printed value into the script literally — scripts do not inherit shell
variables, and forward-slash UNC works in Windows APIs without backslash
escaping:

```js
await page.screenshot({ path: "//wsl.localhost/<distro>/home/lk/Test/outputs/page.png" });
await page.setInputFiles("input[type=file]", ["//wsl.localhost/<distro>/home/lk/Test/report.pdf"]);
```

`/mnt/c/...` paths map to `C:/...`, which is also valid. Downloads and uploads
should round-trip through the workspace this way so normal file tools can read
them afterwards.

## 5. Space discipline

- New goal → new space (`taskSpace("...")`). Continuation of the same goal →
  reuse the `spaceId` printed earlier (`listTaskSpaces()`, `taskSpace(id)`).
- Keep only pages the user needs to see; `finish({ keep: [] })` closes the rest.
- `claimTaskSpace()` and `takeOverTaskSpace()` take control away from the user: use
  them only when the user explicitly asks.
- Do not `finish()` while the task is stopped for user control or failed; report
  the space id instead so the next reply can resume it.

## 6. Safety

This is the user's real browser with their live sessions. Actions are real and
visible.

- Do not submit purchases, publish posts, send messages, delete data, or change
  account settings without explicit confirmation in this conversation.
- On login, CAPTCHA, 2FA, permission prompts, or anything only the user can
  answer, call `await task.handOff()`, end the round, and explain what to do.
- Stop when the user takes control of a space; do not route around it.
- Keep the browser logged-in state private: do not print cookies, tokens, or
  other credentials into the conversation.

### Clearing cookies, cache, or storage

A profile's cookies and HTTP cache are shared with the user's own tabs and every
other space on that profile. Three commands reach the whole profile and cannot be
undone: `Network.clearBrowserCookies`, `Network.clearBrowserCache`, and
`Storage.clearCookies` without a `browserContextId`. Never run one without explicit
user approval, and never as a way past a sign-in wall — call `task.handOff()`
instead.

Prefer the narrowest command: `Storage.clearDataForOrigin` for one origin,
`Network.deleteCookies` for one named cookie, `Page.reload` with
`ignoreCache: true` for stale content. Read
`"$OFFICIAL/references/clearing-state.md"` for the exact reach of each command.

## 7. Troubleshooting

| Symptom | Cause and action |
|---|---|
| `ego lite for Windows was not found` | Not installed, installed for another Windows user, or a non-default root. Set `EGOLITE_BROWSER_EXE` to the full `ego-browser.exe` path, or point the user at the install guide (`https://lite.ego.app/`) and finish onboarding there. |
| `WSL does not have interop` / executable not runnable | WSL interop is disabled (`/proc/sys/fs/binfmt_misc/WSLInterop` missing or `disabled`). This skill cannot reach Windows; report it. |
| `Unknown command: skill` | The installed ego lite predates this skill. Ask the user before running `"$EB" upgrade`, then run `"$EB" skill` again. |
| Connection or `not connected` errors | ego lite is closed or still in first-run onboarding. Ask the user to open it and finish setup, then retry. |
| `ENOENT` or path errors inside a script | A Linux path reached the Windows process. Convert it with `winpath`. |
| `Permission denied` on the scripts | `chmod +x scripts/*` in the installed skill directory, or run them as `bash scripts/ego-browser`. |

Never substitute `curl`, a headless download, or another browser for a task that
needs rendering, login, or user-visible state. Report the blocker instead.

## 8. Other hosts

- **Agent running on Windows**: no bridge needed. Run `ego-browser` directly
  (the command ego lite registers on `PATH`), and read the skill that ships with
  the installation at
  `C:\Program Files\Citro Labs\ego lite\Application\<version>\ego-skills\ego-browser`.
- **Agent running on macOS**: use the macOS ego lite build and its own
  `ego-browser` command; `scripts/ego-browser` and `scripts/winpath` are
  Windows-interop specific.
- **Agent in a container or remote box without interop**: this skill cannot reach
  the user's Windows browser. Report that instead of approximating the page.

## Installing for a harness

Copy the whole `ego-browser/` directory into any directory the harness scans for
`<name>/SKILL.md`. Examples: DeepSeek Harness `~/.dsh/skills/` or
`<project>/.dsh/skills/`; Claude Code `~/.claude/skills/` or
`<project>/.claude/skills/`. A harness without a skills directory can be pointed
at the absolute `SKILL.md` path and told to follow it.
