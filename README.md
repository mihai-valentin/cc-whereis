# cc-whereis

[![Built with Claude Code](https://img.shields.io/badge/built%20with-Claude%20Code-d97757)](https://claude.com/claude-code)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

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

## Written by Claude Code

Every line here — the skill, the manifests, the example config, this README —
was written by Claude Code. No part of it was hand-authored; a human directed
the work, reviewed it, and decided what shipped.

Worth stating plainly rather than hiding, because it's also why the repo is
shaped the way it is. The entire implementation is one markdown file that a
model reads at runtime. A tool whose only moving part is a prompt, written by
the thing that runs it, ought to be legible to the next one.

## Agents welcome

**Pull requests from coding agents are welcome here on the same terms as any
other.** No disclosure penalty, no separate review queue, no "human-authored
only" rule. If the change is right, it's right.

There is nothing to build and nothing to install — the whole repo is seven
files, and it runs offline with no network, no API keys and no services to
stand up:

| Path | What it is |
|---|---|
| `skills/whereis/SKILL.md` | **The entire implementation.** Prose, not code — it is read by a model, so edits here are behaviour changes that no test can judge |
| `examples/projects.conf` | The config format, annotated. The normative spec for what a valid map looks like |
| `.claude-plugin/plugin.json` | Plugin manifest |
| `.claude-plugin/marketplace.json` | Marketplace entry; its `version` must agree with `plugin.json` |

Because the implementation is a prompt, "correct" is defined by invariants
rather than by assertions. A change that breaks one of these is wrong even if
it reads well:

1. **No runtime.** No binary, no Node, no Python, no per-platform build. If
   something seems to need a process, it needs a better prompt. This is what
   makes it installable everywhere Claude Code runs, native Windows included.
2. **Zero context cost until invoked.** Only the skill's `description` is
   resident; the body and the map load on demand. No hooks, no `SessionStart`,
   nothing eager. A regression here is a correctness bug, not a perf nit — the
   whole pitch is context hygiene.
3. **Never guess a path.** A name whose directory is gone is reported as
   "known label, missing directory". No fuzzy match, no similarly-named
   sibling, no filesystem search. Silently resolving to the wrong project is
   worse than not resolving at all, because the user can't audit a path they
   never saw.
4. **Ambiguity is an error.** A name claimed by two projects fails with both
   candidates named. Never pick the first match.
5. **No quoting in the config format.** A value is everything after the `=`,
   trimmed. That is what makes `C:\Users\you\proj` safe in a format with no
   parser to catch a bad escape. Don't "improve" it back toward TOML or YAML.
6. **The map resolves; it does not act.** No writes into mapped projects, no
   commands run in them, no network, no telemetry, no frequency tracking.

If you want to change one of those rather than work within it, open an issue
first — they're the design, not accidents.

## License

MIT.
