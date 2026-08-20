# cc-whereis

**Claude Code doesn't know where your other projects are. This tells it.**

You're in a session rooted at project A. You ask for something from project B.
Claude has never heard of B. So you paste an absolute path — and paste it again
tomorrow, and again next week.

`cc-whereis` keeps one map of the projects on your machine, addressable by name
or alias:

```ini
[dashboard]
path    = ~/code/acme-dashboard
aliases = explore, analytics-ui
```

Now *"grab the auth middleware from **Explore**"* resolves, from any session.

## Why not just put the paths in CLAUDE.md?

Four reasons, and the last one is the interesting one.

1. **It pollutes every project's context with unrelated paths.** Project A's
   context should not carry the filesystem layout of thirty repos it will never
   touch. Context hygiene is the entire point of this thing.
2. **Projects move.** They get renamed, archived, moved to another disk. A
   hard-coded path in a markdown file is wrong the moment anything shifts.
3. **It's an N-way sync problem.** One map per project, all drifting
   independently, all needing the same edit.
4. **A stale CLAUDE.md line cannot know it's stale.** It just quietly lies, and
   Claude burns a turn discovering a directory that isn't there — or, worse,
   finds something plausible nearby and uses it.

cc-whereis fixes 1–3 by construction: one file, outside every project, loaded
only when a lookup actually happens.

For 4, it does the one thing a markdown line can't — it **checks**. A resolved
path is `ls`-ed before it's used, and a dead entry is reported as
*"known label, missing directory"* rather than quietly worked around. It will
never substitute a similar-looking sibling directory, and a name claimed by two
projects is an error naming both, not a coin flip. **If it isn't sure, you find
out.**

## Install

```
/plugin marketplace add mihai-valentin/cc-whereis
/plugin install cc-whereis
```

Then write your map to `~/.config/cc-whereis/projects.conf` — see
[`examples/projects.conf`](./examples/projects.conf) for the full format. It's
five lines to get started:

```ini
[api]
path    = ~/code/acme-api
aliases = backend, core-api

[dotfiles]
path = ~/dotfiles
```

No quotes, no escapes, no indentation — so a Windows path like
`C:\Users\you\proj` is just a value. `#` starts a comment at the beginning of a
line, and nowhere else.

## Use

Mostly you don't. Name a project and Claude looks it up:

> "compare this middleware against the one in **hub**"

Or ask directly:

```
/whereis hub          # resolve one name
/whereis              # list the map
```

## Design

No binary, no Node, no Python, no per-platform build, no MCP server. The plugin
is one markdown file plus your config; Claude resolves against it with its own
`Read`, and validates with `ls`. That's why it installs anywhere Claude Code
runs, native Windows included.

It costs **zero context until you actually need it** — the skill's body loads on
demand, not at session start — which is rather the point of a tool whose pitch
is "stop putting paths in your context."

## Status: v0, deliberately small

**What's here:** the config format, name/alias resolution, and `/whereis`.

**What isn't, yet:** `doctor` (bulk-validate every path in one pass), `add` /
`rm` commands, filesystem discovery, importing from shell jumpers like
[`cdp`](https://github.com/mihai-valentin/cdp) or zoxide, and team/remote maps.
v0 is read-only on purpose — the map is a file you edit, and everything else is
a feature that has to earn its way in.

**Known limitation:** headless runs (`claude -p`, hooks, CI) have nobody to
approve directory access, so resolution works but reading the resolved project
may not.

## Not doing, ever

Frequency tracking, telemetry, network calls, or writes into a mapped project.
The map resolves; it does not act.

## License

MIT.
