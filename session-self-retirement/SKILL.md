---
name: session-self-retirement
description: Move the current DSH session, together with its forked and subagent descendants, into the Session-Time-Vault once its communication goals are achieved, by staging their directories for the dsh unit's ExecStopPost hook and restarting dsh.
whenToUse: The user asks to "use session-self-retirement", to "retire this session", or to "retire this chat and its forks". One run retires exactly one session tree — the one you are running in.
---

# Session self-retirement

You retire the session you are running in, because its communication goals are
achieved. The directories are **not** moved by you: you write their absolute
paths to a list, and the dsh unit's `ExecStopPost` moves them into
`~/.dsh/Session-Time-Vault` after dsh has exited. That guarantees the log is
flushed and the `session.lock` flock is released before the move, and it keeps
the retired session retrievable.

## Non-negotiables

- One tree per run. The staged set must always contain `$DSH_SESSION_ID`.
- Never remove a session, never edit a log, never touch `~/.dsh/attachments`
  (content-addressed and shared) or `workspace.json`.
- Abort if any member of the tree is pinned or archived.
- Move, never destroy. The Session-Time-Vault is the only destination.

## Preconditions

Check these first; if any fails, report it and stop.

1. `$DSH_SESSION_ID` is set.
2. `~/.dsh/session-to-retire/` and `~/.dsh/Session-Time-Vault/` exist and are
   writable.
3. The unit carries the hook and was reloaded since it was added:

   `ExecStopPost=/bin/sh -c 'xargs -r -a %h/.dsh/session-to-retire/list.txt /usr/bin/mv -t %h/.dsh/Session-Time-Vault -- && rm -f %h/.dsh/session-to-retire/list.txt'`

   Verify with `systemctl --user show dsh -p ExecStopPost`.
4. `~/.dsh/session-to-retire/list.txt` does not exist or is empty, and no
   `.list.*` temp is left behind. A pending `list.txt` belongs to an earlier
   run; never overwrite it — report it and stop. A leftover temp is inert, but
   it means an earlier run died between `mktemp` and `mv`.

## Step 1 — sentinel

Generate a unique token, e.g. `selfretirement-$(openssl rand -hex 8)`. Put it in
your next assistant message as a plain line. The message is persisted as a
session event, so the token now identifies the live log.

The live session is always the most recently written log — take the newest
file and retry only that one:

```sh
newest=$(ls -t ~/.dsh/sessions/*/session-*/session.v4.jsonl.zstd | head -1)
for i in 1 2 3 4 5; do
  zstd -dc "$newest" | grep -q "$TOKEN" && break
  sleep 0.25
done
zstd -dc "$newest" | grep -q "$TOKEN" || {
  # a concurrently written tab can win the race; fall back to the known id
  zstd -dc ~/.dsh/sessions/*/"$DSH_SESSION_ID"/session.v4.jsonl.zstd | grep -q "$TOKEN" \
    || { echo "ABORT: token not in the live log"; exit 1; }
}
```

The retries cover the ~200 ms drain window only, not a search over sessions.
Take the session directory from `dirname "$newest"` and require its basename
to be `$DSH_SESSION_ID`; anything else means another tab was written a moment
ago, so stop.

Also read that header and require `id == $DSH_SESSION_ID`; take `cwd` from it.

## Step 2 — build the tree from disk

Scan every stored header, not the UI: archived forks are invisible there.

```sh
for f in ~/.dsh/sessions/*/session-*/session.v4.jsonl.zstd; do
  zstd -dc "$f" | head -1
done
```

Each header carries `id`, `cwd`, optional `parentSession`, `isSeeded`,
`origin`, `delegationDepth`. The tree is the transitive closure of
`parentSession` starting at `$DSH_SESSION_ID`:

- `isSeeded: true` with no `origin` → a fork.
- `origin: "subagent"` → a subagent session (also has `parentSession`).
- A fork of a fork points at the intermediate session, so walk until no
  children remain.

Rules:

- Every member must have the same `cwd` as the root. If one differs, abort:
  that is a dsh bug, and a partial tree must never be retired.
- The root must be in the set. If the root has no stored log, abort.
- Never include an id you did not read from a header.

## Step 3 — titles

The header has no title. The title is the **last** `session/title` event in
the log (`data.title`). If absent, use the first 60 characters of the first
user message; else `(untitled)`.

## Step 4 — show the tree

Read the tree once and print it once, as soon as it is built and titled. Do
this in every run, before any gate: showing is read-only and is never gated.
Sizes come from `du -k` on the session directory.

```
session-e2ec5ba6  "goal achieved"        [you, 48 kB]
├── session-fada6275  "fork A"            (fork, 61 kB)
│   └── session-1a2b3c4d  "fork of fork"  (fork, 12 kB)
└── session-9f8e7d6c  "subagent: research" (subagent, depth 1, 9 kB)
```

The later steps reuse this printout instead of regenerating it:

- Step 5 marks the offending nodes on it when the gate fails;
- Step 6 repeats it alongside the confirmation.

If the root has no stored log there is no tree to show; report that and stop.

## Step 5 — pinned / archived gate

Both lists live in `~/.dsh/storages/workspace.json`:

```sh
jq -r '.global.pinnedSessionIds[], .global.archivedSessionIds[]' \
  ~/.dsh/storages/workspace.json
```

Intersect with the tree. If the intersection is non-empty:

- stage nothing, write no list;
- print the tree from Step 4 with the offending nodes marked;
- name which ids are pinned and which are archived;
- tell the user to unpin/unarchive, or to abort.

## Step 6 — confirm

Print the Step 4 tree again, then state plainly:

- N sessions move into `~/.dsh/Session-Time-Vault` and leave the UI;
- the restart ends the in-flight turns of **every** live session of the dsh
  service, not just this one; the sessions themselves persist, and the Web UI
  reconnects on its own;
- this session leaves the UI together with its forks.

Wait for an explicit yes. Anything else → stop, leave no list behind.

## Step 7 — stage the list

Only after yes. One absolute directory path per line, nothing else: the
session directories themselves (`~/.dsh/sessions/<encoded-cwd>/session-<uuid>`),
derived from the log path you scanned — never reconstructed from memory. They
start with `/`, so no operand can be read as an option.

No shell array and no variable carrying state between steps. Put the tree's
paths directly in the block, one quoted line each, and let the rename publish
the file atomically:

```sh
umask 077
dir=~/.dsh/session-to-retire
tmp=$(mktemp "$dir/.list.XXXXXX")
printf '%s\n' \
  '/home/USER/.dsh/sessions/--home-USER-tmp--/session-00000000-0000-0000-0000-000000000000' \
  '/home/USER/.dsh/sessions/--home-USER-tmp--/session-11111111-1111-1111-1111-111111111111' > "$tmp"
mv "$tmp" "$dir/list.txt"
wc -l < "$dir/list.txt"
```

Replace the example paths with the tree's members; the printed line count must
equal the count shown in Step 6.

The list is consumed by **any** stop of the unit, including an unrelated
restart. Stage it only at this moment, with the confirmation already given.

## Step 8 — restart detached

Do not call `systemctl --user restart dsh` directly: it stops the unit you are
running in, so your turn is cut before the user sees the confirmation. Arm a
transient timer owned by the user manager and finish the turn:

```sh
systemd-run --user --collect --unit="dsh-selfretirement-$(date +%s)" \
  --on-active=2 systemctl --user restart dsh
```

Fallback when `systemd-run` is unavailable: `systemctl --user restart --no-block dsh`
and tell the user the reply may be truncated.

## Step 9 — report

State how many directories were staged and that they move into
`~/.dsh/Session-Time-Vault` when dsh stops.

No restore recipe. The answer carrying one would retire with the session, so it
reaches nobody. Bringing a session back is a later manual job — unpack the
directory from the vault and inspect or move it — not part of this run.

Nothing else is required: dsh is already gone when the move happens, and the
Web UI reconnects on its own when it comes back.

## Abort conditions

- `$DSH_SESSION_ID` unset, or header id differs.
- Token absent from the live log after retries.
- Any tree member pinned or archived.
- A member's `cwd` differs from the root's.
- A non-empty `list.txt` is already staged.
- The user does not answer with an explicit yes.

On abort: no list, no restart, report the reason.
