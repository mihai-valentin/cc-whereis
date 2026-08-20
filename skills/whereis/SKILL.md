---
name: whereis
description: >-
  Resolve a project name or alias to its absolute path on this machine, by
  reading the user's project map at ~/.config/cc-whereis/projects.conf. Use
  this whenever the user refers to a project, repo, or codebase by a name that
  is not the one this session is rooted in — "grab the auth middleware from
  Dashboard", "how does Explore do this", "is that fixed in the API repo yet"
  — and you do not already know where it lives on disk. Also use it when the
  user asks where a project is, or asks which projects are on the map. Do NOT
  use it for paths inside the current workspace, and do not use it to go
  looking for a project that the map does not list.
argument-hint: "[project name or alias]"
allowed-tools: Read, Bash(ls *)
---

# Resolving a project name to a path

The user keeps one hand-written map of the projects on this machine. Your job
is to turn a name they used into the **absolute path of that project's root** —
or to say clearly that you can't, which is a perfectly good outcome.

**The map is the only source of truth.** You do not search the filesystem, you
do not guess from a similar-looking directory nearby, and you do not reason
from a path you saw earlier in the session. If it isn't in the map, you don't
know it. Read that sentence again before you improvise: a wrong path that looks
right is the single worst thing this skill can produce, because the user has no
reason to double-check a path they never saw.

## Step 1 — Read the map

```
~/.config/cc-whereis/projects.conf
```

If `XDG_CONFIG_HOME` is set, the map is at `$XDG_CONFIG_HOME/cc-whereis/projects.conf`
instead. Read it with `Read`.

**If the file does not exist**, the user has no map yet. Say so, show them the
format below, and tell them where to put it. Do not create it for them unless
they ask, and do not go hunting for their projects.

## Step 2 — Understand the format

```ini
[dashboard]
path    = ~/code/acme-dashboard
aliases = explore, analytics-ui
tags    = frontend, work

[dotfiles]
path = ~/dotfiles
```

- `[name]` is the project's primary label.
- `path` is required. Absolute, or starting with `~/`.
- `aliases` is a comma-separated list. **Aliases are peers of the name** — a
  match on an alias is exactly as good as a match on the name.
- `tags` is optional grouping, useful when the user asks something like "which
  frontend projects do I have".
- Values run to the end of the line and are trimmed. There is no quoting and no
  escaping, so a Windows path like `C:\Users\x\proj` is just a value.
- A line whose first non-space character is `#` is a comment. A `#` anywhere
  else belongs to the value.

## Step 3 — Match

Compare the user's word against every `[name]` and every entry in every
`aliases` list, **case-insensitively**. Then exactly one of four things is true:

**One match.** Good. Go to Step 4.

**More than one match.** This is an error, not a decision you get to make.
Report every candidate with its path and ask which one they meant:

> `explore` is claimed by two projects in your map: **dashboard**
> (`~/code/acme-dashboard`) and **explore-infra**
> (`~/code/explore-infra`). Which did you mean?

Never pick the first one. Never pick the one that seems more likely.

**No match.** Say plainly that the name isn't in the map, and list the names
that *are* if the map is small enough to be useful. You may offer to add a line
to the map. You may not go looking on disk, and you may not offer a path that
merely resembles the name.

**No map at all.** Handled in Step 1.

## Step 4 — Verify the directory is really there

Expand `~` and check the path exists before you hand it over:

```bash
ls -d <expanded path>
```

**If it exists**, report the resolved root — the name that matched, the path,
and, if a tag or two makes it clearer, those. Then use it.

**If it does not exist**, this is the loud case. The map has rotted, and saying
so is the whole point:

> **Known label, missing directory.** Your map has **dashboard** at
> `~/code/acme-dashboard`, but nothing is there.
> The project has probably moved or been removed. Want me to fix that line?

Then stop. Do not look in the parent directory. Do not try a
similarly-named sibling. Do not fall back to a filesystem search. A stale entry
that gets reported is a two-second fix; a stale entry that gets silently
replaced with the wrong project is a bug the user finds out about much later.

## Step 5 — Hand back the root, not a file

Always resolve to the **project root**, even when the user asked about one file
inside it. Reading files under a directory outside the workspace needs the
user's approval, and approving a root costs them one keypress for the whole
project instead of one per file.

Do not run `/add-dir` yourself, and do not ask Claude Code to widen access on
the user's behalf. Report the path and continue; the normal access prompt will
appear when you actually read something, and the user decides then.

## Listing the map

When invoked with no argument — or when the user asks what's on the map — read
the file and present the entries as a small table: name, aliases, path. Don't
stat every path while listing; a listing is not a health check.

## What this skill does not do

- It does not write anything into a mapped project, run commands there, or
  touch the network.
- It does not track how often projects get used, and it does not reorder
  anything by frequency. The map is authored intent, in the order the user
  wrote it.
- It does not edit the map on its own. If the user asks you to add, change, or
  remove an entry, that's ordinary file editing — use `Edit`, keep the format
  above, and show them the diff.
