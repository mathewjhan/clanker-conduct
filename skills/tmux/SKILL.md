---
name: tmux
description: Open tmux windows (tabs), split panes, run commands in other panes, read their output, wait for servers to be ready, show your work to the user, and close what you opened. The first action is always `tmux-agent start TASK-NAME`, which moves you into your own tmux session; every tab you open goes there. Use when the user asks to do something "in another tab", "beside you", "in a split", "in your session", to read or type into another pane, to switch their view, or to close what you opened.
---

# tmux

Control tmux through `scripts/tmux-agent`, located in the same directory as this `SKILL.md`. Run it with no arguments for full usage. Below it is written as `ta`; define that first in each shell, replacing `SKILL_DIR` with the absolute path of this skill's directory:

```sh
ta() { bash SKILL_DIR/scripts/tmux-agent "$@"; }
```

## Step 1, always: move into your own session

The first tmux action in every conversation is:

```sh
ta start TASK-NAME
```

`TASK-NAME` is a short name for the task, such as `fix-login-redirect` or `bump-deps`. Run it before `whoami`, `ls`, `tab`, `split`, or anything else, even if you only intend to read a pane. It:

1. creates a tmux session named `TASK-NAME`,
2. moves your own pane into it as window `0:agent`,
3. switches the user's terminal to that session.

If it prints "agent session already exists", you already did this step; continue.

## Terms

- **Session**: a named group of windows. Sessions keep running when no terminal shows them. The user's terminal (the "client") shows one session at a time.
- **Window**: a tab inside a session.
- **Pane**: one terminal inside a window. A window can be split into several panes.
- **IDs**: panes are `%N`, windows `@N`, sessions `$N`. IDs never change while the thing exists; indexes and names can. Always target by ID once you have it.

## Your session

Each agent has one tmux session that holds its own pane and everything it opens.

- It is created by `ta start` (Step 1). `tab` and `split` refuse to run until then.
- `ta tab` puts new windows in your session and `ta split` splits next to you, also in your session.
- If the task changes, rename the session: `ta name NEW-NAME`.
- `ta focus` (no argument) switches the user's terminal back to your session.

## Step 2: check where things are

After Step 1, run `ta whoami`. It prints:

- `self`: the session, window, and pane you run in. After Step 1 this is your session.
- `viewing`: what the user's terminal is showing right now. This can differ from `self`.
- `agent session`: your session. If it says "none yet", you skipped Step 1; run it now.

Run `ta ls` when you need to find a pane by its name, command, or directory.

## Choosing where to act

- No location given, "another tab", "in the background": a new `ta tab` in your session.
- "beside you", "next to you", "here": `ta split` of your own pane (the default target).
- "in my session" (the session the user was in before you moved): `ta tab --user-session`.
- "the current tab", "the focused pane", "what I'm looking at": the `viewing` location.
- A name ("the psql pane", "the api tab"): the window, pane, or session with that name or running that command. A name overrides the rules above.
- If more than one target fits, list the candidates and ask.

## Commands

Creating prints the new pane ID. Nothing takes focus unless `--focus` is passed.

| Need | Command |
| --- | --- |
| Move into your own session (Step 1, always first) | `ta start TASK-NAME` |
| New tab in your session | `ta tab -n NAME [-c DIR] [-- CMD]` |
| New tab in the user's original session | `ta tab --user-session -n NAME [-- CMD]` |
| Split | `ta split [-t TARGET] [-h\|-v] [-b] [-l SIZE] [-- CMD]` |
| Rename your session | `ta name NAME` |
| Run a command that finishes, get output + exit code | `ta run PANE [-T SECS] -- CMD` |
| Type a line (shell command, REPL input, `y`) | `ta send PANE -- TEXT` |
| Multi-line text (prompt for another agent, TUI input) | `printf '%s' "$TEXT" \| ta paste PANE` |
| Special keys | `ta keys PANE C-c` / `Escape` / `Up` / `q` |
| Read the screen | `ta read PANE [-n LINES \| --since]` |
| Wait for output | `ta wait PANE 'REGEX' [-T SECS]` |
| Show your session / a pane to the user | `ta focus [TARGET]` |
| Rename | `ta rename window\|session\|pane TARGET NAME` |
| Close one thing you created | `ta close window\|session\|pane TARGET` |
| Close everything you created (incl. your session) | `ta close-mine` |

Split directions: `-h` = right (default), `-v` = below, add `-b` for left/above. `-l 33%` sets the new pane's size.

## Examples

All examples assume `ta start TASK-NAME` already ran.

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
- **Close only what you created.** `close` refuses other things; pass `--force` only when the user explicitly names something they want closed.
- **Name things after their purpose** (`dev`, `tests`, and your session after the task) so the user and later commands can find them.
- **Report the session name and pane IDs** of things you start so the user can find them.
