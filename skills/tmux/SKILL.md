---
name: tmux
description: Manage tmux windows (tabs) and panes in the session where the agent started. Use when the user asks to work in another tab or split, read or type into another pane, show terminal output, or close what the agent opened. The user creates sessions; the agent stays in place.
---

# tmux

Control tmux through `scripts/tmux-agent`, located in the same directory as this `SKILL.md`. Run it with no arguments for full usage. Below it is written as `ta`; define that first in each shell, replacing `SKILL_DIR` with the absolute path of this skill's directory:

```sh
ta() { bash SKILL_DIR/scripts/tmux-agent "$@"; }
```

## Identify your starting session

The user creates and names sessions before starting agents. Keep your pane where it is and create new windows and splits in that session. Do not automatically create, rename, move, or delete sessions or move your own pane.

Begin by checking your location:

```sh
ta whoami
```

The output distinguishes:

- `self`: your pane and its current session.
- `starting session`: the session recorded on first use for this conversation; new windows go here.
- `viewing`: what the user's terminal currently shows, which may be elsewhere.

There is no `ta start` step. Checking your location does not move a pane or change focus.

If `self` is unknown or inconsistent with this conversation, use `ta ls` and `ta read PANE` to identify the pane showing this conversation. Command processes can lack `TMUX_PANE` or inherit another terminal's value. Do not use the focused pane or the command process's parent terminal as proof of your location.

Once the pane is verified, pass its ID explicitly in every shell invocation. For example, if it is `%52`:

```sh
ta() { TMUX_AGENT_PANE='%52' bash SKILL_DIR/scripts/tmux-agent "$@"; }
ta whoami
```

If several panes fit, ask the user which one. The helper refuses to create windows or split panes without an identified starting session; it does not fall back to the session currently on screen.

## Terms

- **Session**: a named group of windows. Sessions keep running when no terminal shows them. The user's terminal (the "client") shows one session at a time.
- **Window**: a tab inside a session.
- **Pane**: one terminal inside a window. A window can be split into several panes.
- **IDs**: panes are `%N`, windows `@N`, sessions `$N`. IDs never change while the thing exists; indexes and names can. Always target by ID once you have it.

## Choosing where to act

- No location given, "another tab", "in the background": a new `ta tab` in your starting session.
- "beside you", "next to you", "here": `ta split` of your own pane (the default target).
- "the current tab", "the focused pane", "what I'm looking at": the `viewing` location.
- A name ("the psql pane", "the api tab"): the window, pane, or session with that name or running that command. A name overrides the rules above.
- If more than one target fits, list the candidates and ask.

Use `ta ls` to find named panes. New windows and splits stay in your starting session even when the user is viewing another session.

## Commands

Creating prints the new pane ID. Nothing takes focus unless `--focus` is passed.

| Need | Command |
| --- | --- |
| Identify your pane and starting session | `ta whoami` |
| New tab in your starting session | `ta tab -n NAME [-c DIR] [-- CMD]` |
| Split within your starting session | `ta split [-t TARGET] [-h\|-v] [-b] [-l SIZE] [-c DIR] [-- CMD]` |
| Run a command that finishes, get output + exit code | `ta run PANE [-T SECS] -- CMD` |
| Type a line (shell command, REPL input, `y`) | `ta send PANE -- TEXT` |
| Multi-line text (prompt for another agent, TUI input) | `printf '%s' "$TEXT" \| ta paste PANE` |
| Special keys | `ta keys PANE C-c` / `Escape` / `Up` / `q` |
| Read the screen | `ta read PANE [-n LINES \| --since]` |
| Wait for output | `ta wait PANE 'REGEX' [-T SECS]` |
| Show your starting session / a pane to the user | `ta focus [TARGET]` |
| Rename a window or pane | `ta rename window\|pane TARGET NAME` |
| Close one thing you created | `ta close window\|pane TARGET` |
| Close your windows and panes, preserving sessions | `ta close-mine` |

Split directions: `-h` = right (default), `-v` = below, add `-b` for left/above. `-l 33%` sets the new pane's size.

## Examples

All examples assume your pane and starting session have been verified.

- "Start the dev server in another tab and tell me when it's ready":
  `P=$(ta tab -n dev -- npm run dev)` then `ta wait "$P" 'ready|listening|Local:' -T 120`. On timeout, `ta read "$P"` and report what it shows.
- "Open nvim with top in a narrow pane on the right, and show me":
  `P=$(ta tab -n edit -- nvim)`, `ta split -t "$P" -h -l 33% -- top`, then `ta focus "$P"`.
- "Run the tests beside you and fix whatever fails":
  `P=$(ta split -v -l 40%)` then `ta run "$P" -T 900 -- pytest -x`. Fix based on the printed output and `[exit N]`.
- "Run `\dt` in the psql pane": find the pane with `ta ls`, `ta read` it to confirm it shows a psql prompt, then `ta send PANE -- '\dt'` and `ta read PANE --since`.
- "Show me what you're running": `ta focus`.
- "Close everything you opened": `ta close-mine`.

## Rules

- **Never type into your own pane.** The script refuses, but don't try.
- **Don't take focus unless asked.** Only use `--focus` or `ta focus` when the user wants to see the result.
- **Read before typing into a pane you didn't create.** Confirm it is waiting for the kind of input you will send: a shell prompt takes commands, a REPL takes its own syntax, a `y/n` prompt takes one key. Multi-line text sent to a shell runs each line as a command.
- **Use `run` for commands that finish** (tests, builds, scripts). It needs a shell prompt and prints the exit code. **Use `send` + `wait` for commands that keep running** (servers, watchers, REPLs).
- **Close only what you created.** `close-mine` preserves your pane and the user's sessions. `close` refuses other windows and panes; pass `--force` only when the user explicitly names something they want closed.
- **Name windows and panes after their purpose** (`dev`, `tests`) so the user and later commands can find them. Leave session management to the user.
- **Report the session name and pane IDs** of things you start so the user can find them.
