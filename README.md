# AI-aware Konsole

Konsole patch that highlights which **tab is running an AI agent** (claude, cursor,
opencode, pi, agy, codex, …) by coloring the tab title area — an unmistakable yellow
tab per agent session.

## How it works

- `SessionController` runs a lightweight 2 s timer per session.
- It gets the session's **foreground process group** (`tcgetpgrp` on the tab's pty)
  and scans `/proc` for any process in that group whose name matches a configurable
  list of AI-agent process names (catches wrappers too, e.g. `npx`/`node`/`python`
  launching the agent).
- On a match the tab gets the configured color; when the agent exits the tab reverts
  to its profile/user color. Colors the user set manually (Rename Tab dialog) are
  never overridden.
- Rendering (`DetachableTabBar::paintEvent`): the whole tab title area is tinted
  (translucent wash) plus Konsole's existing solid color stripe under the title.

## Settings (konsolerc → `[AgentDetection]`)

```ini
[AgentDetection]
AgentDetectionEnabled=true          # master switch
AgentTabColor=#ffff00               # the highlight color
AgentProcessNames=claude,cursor,cursor-agent,cursor-cli,opencode,pi,agy,agent,codex,aider,goose,amp,ollama,gemini
```

Unknown agent binaries: just add their process name (what shows in `/proc/<pid>/comm`,
≤15 chars, case-insensitive). Restart Konsole or `killall -HUP konsole` after editing.

## Build & try it

```bash
cmake -S src -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=$HOME/.local
cmake --build build -j$(nproc) && cmake --install build
# run the patched build (data from both ~/.local/share and /usr/share is found)
~/.local/bin/konsole
```

System-wide install instead: replace the prefix with `/usr` (or `/usr/local`).
Ready-made diff: `agent-tab-color.patch`

```bash
git clone --depth 1 --branch v25.12.3 https://github.com/KDE/konsole.git src
cd src && git apply ../agent-tab-color.patch && cd ..
```

## Verified (Konsole 25.12.3, Ubuntu 26.04, Qt 6.10 / KF6 6.24)

- ✅ tab running an agent (`claude` comm) → whole tab title area tinted + solid color bar
- ✅ agent launched from a shell (`bash` → `/tmp/claude`) detected via process-group scan
- ✅ agent exit → tab color reverts automatically within ~2 s
- ✅ tabs without agents unaffected
- ✅ `AgentTabColor=#ff0000` override in `konsolerc` honored

## Patch summary

| File | Change |
|---|---|
| `src/settings/konsole.kcfg` | new `AgentDetection` group (enabled/color/names) |
| `src/session/Session.{h,cpp}` | `foregroundProcessGroupId()` accessor |
| `src/session/SessionController.{h,cpp}` | 2 s watcher timer + `updateAgentTabColor()` + `/proc` pgrp scan |
| `src/widgets/DetachableTabBar.cpp` | full-tab wash in `paintEvent` |

## Known limitations

- Agents nested inside `tmux`/`screen` are not seen (they own the pty's foreground
  group — detection is per-Konsole-tab only).
- Agents that fully daemonize into their own process group are not detected.
- On non-Linux platforms detection falls back to the foreground process name only.