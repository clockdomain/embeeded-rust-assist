# claude-config

Curated, shareable subset of `~/.claude` under version control.

## What's tracked

- `skills/` — authored user-global skills (`peripheral-parity-port`,
  `userspace-driver-client`, `design-patterns`, `stop-and-instrument`).
- `settings.json` — harness config: hooks, permission allowlist, plugins.

## What's deliberately NOT here

No secrets and no machine-local state. Excluded by the allowlist
`.gitignore`: `~/.claude/.credentials.json`, `sessions/`, `projects/`
(transcripts contain verbatim code from every repo on the machine),
`history.jsonl`, `todos/`, `file-history/`, `cache/`, `telemetry/`, etc.

`settings.json` here is **sanitized**: a Gerrit HTTP password that was
embedded in a permission entry is replaced with `__REDACTED__`. The live
`~/.claude/settings.json` still holds the real value — keep them in sync
by hand, and never copy a raw secret back into this repo.

## Sync model

This is a **copy**, not a symlink target. To deploy onto a machine:

```sh
cp -r skills/* ~/.claude/skills/
# review settings.json by hand (it is redacted) before merging
```

Treat `~/.claude` as the source of truth for skills; refresh this repo
with `cp -r ~/.claude/skills .` and re-check for secrets before commit.
