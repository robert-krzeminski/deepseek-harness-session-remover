# deepseek-harness-session-remover
*(linux-only)*

Retire the session you are in — plus its forks — and move it out of DSH into a
vault. A move, not a delete.

## Function

0. You call: "retire this session".
1. The skill writes the actual session name (including fork names, if any) to
   a list — it aborts if any of those are pinned or archived.
2. It asks for confirmation to remove the active session from DSH.
3. It quits DSH and moves the session to the archive vault.
4. It restarts DSH; you continue in a new session.

*Needs full file access — the vault sits outside your workspace.*

## Security verification

Ask any AI:

```
evaluate the skill for security!
https://raw.githubusercontent.com/robert-krzeminski/deepseek-harness-session-remover/main/session-self-retirement/SKILL.md
```

## Compatibility

Ask DSH:

```
is this skill compatible with my DSH?
https://raw.githubusercontent.com/robert-krzeminski/deepseek-harness-session-remover/main/session-self-retirement/SKILL.md
```

## Installation

Ask DSH:

```
add this skill to DSH
https://raw.githubusercontent.com/robert-krzeminski/deepseek-harness-session-remover/main/session-self-retirement/SKILL.md
```

## Recommendation

**Backup first.**

```
tar -czf ~/.dsh-$(date +%Y%m%d-%H%M).tar.gz ~/.dsh
```
## Linux-only

As committed it needs a systemd user unit, so it's Linux-only. The only truly Linux-specific parts are the ExecStopPost trigger and the restart call; the retirement step itself is just a directory move, so a macOS/Windows version is plausible via a detached helper that waits for the dsh PID to exit. I can't test either platform, anyone who can test is welcome to contribute one.

## License

MIT — see [LICENSE](LICENSE). Free to use, modify, and redistribute; keep the
copyright notice.
