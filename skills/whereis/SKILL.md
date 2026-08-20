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

Turn a name the user used into the **absolute path of that project's root** —
or say clearly that you can't, which is a perfectly good outcome.

**The user's map is the only source of truth.** Never search the filesystem,
never guess from a similar-looking directory, never reason from a path you saw
earlier in the session. A wrong path that looks right is the worst thing this
skill can produce: the user cannot audit a path they never saw.

## 1. Read the map

`~/.config/cc-whereis/projects.conf`, or `$XDG_CONFIG_HOME/cc-whereis/projects.conf`
if that variable is set.

**No file?** The user has no map yet. Say so, show the format below, name the
path. Don't create it unasked, and don't go hunting for their projects.

## 2. The format

```ini
[dashboard]
path    = ~/code/acme-dashboard
aliases = explore, analytics-ui
tags    = frontend, work
```

- `[name]` is unique; `path` is required, absolute or `~/`-relative;
  `aliases` and `tags` are optional, comma-separated.
- **Aliases are peers of the name** — matching one is as good as the other.
- A value is everything after the `=`, trimmed. No quotes, no escapes, so
  `C:\Users\x\proj` is just a value. `#` comments only at a line's start.

## 3. Match

Compare the user's word **case-insensitively** against every name and every
alias. Then exactly one of these is true:

| Result | What you do |
|---|---|
| One match | Step 4. |
| Two or more | **This is an error, not a decision.** Name every candidate with its path and ask which they meant. Never pick the first. Never pick the likelier one. |
| None | Say the name isn't on the map, list what is if the map is short enough to help, offer to add a line. Do not search the disk. Do not offer a path that merely resembles the name. |

## 4. Verify before handing it over

```bash
ls -d <expanded path>
```

**It exists** → report the name that matched and its path, then use it.

**It doesn't** → this is the loud case. Say so and stop:

> **Known label, missing directory.** Your map has **dashboard** at
> `~/code/acme-dashboard`, but nothing is there. Want me to fix that line?

Do not check the parent directory, try a similarly-named sibling, or fall back
to a search. A reported stale entry is a two-second fix; a silently substituted
one is a bug the user discovers much later.

## 5. Hand back the root, not a file

Always resolve to the project root, even when the user asked about one file in
it — approving a root costs one keypress for the whole project instead of one
per file. Report the path and continue; don't run `/add-dir` or widen access
yourself. The normal prompt appears when you read something, and they decide
then.

## Listing

Invoked with no argument, or asked what's on the map: present name, aliases and
path as a small table. Don't stat the paths — a listing is not a health check.

## Never

- Write into a mapped project, run commands in one, or touch the network.
- Track usage or reorder by frequency. The map is authored intent, in the
  order the user wrote it.
- Edit the map on your own initiative. If they ask you to add, change or remove
  an entry, that's ordinary file editing — use `Edit`, keep the format above,
  show the diff.
