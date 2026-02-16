---
name: fs-guard
description: "workspace-only file access — blocks Read/Edit/Write/Glob/Grep outside workspace"
metadata:
  {
    "openclaw":
      {
        "emoji": "🛡️",
        "events": ["PreToolUse"],
      },
  }
---

# fs-guard

restricts all file tools to workspace only. prevents agent from reading secrets, configs, or system files.

## what it does

- intercepts PreToolUse for Read, Edit, Write, Glob, Grep
- resolves symlinks and `../` traversal via `realpath`
- checks resolved path against allowlist (workspace + /tmp)
- blocks everything else with `permissionDecision: "deny"`

## threat model

| attack | without hook | with hook |
|--------|-------------|-----------|
| `Read ~/.openclaw/.env` | leaks secrets | DENIED |
| `Edit ~/.openclaw/openclaw.json` | tampers config | DENIED |
| `Grep API_KEY /home/user` | finds secrets | DENIED |
| `Read /etc/age/keys/bot.key` | leaks encryption key | DENIED |
| `Read workspace/../../.ssh/id_rsa` | traversal attack | DENIED (realpath resolves) |
| `Read /tmp/test.txt` | temp file access | ALLOWED |
| `Read ~/workspace/SOUL.md` | normal work | ALLOWED |

## install

1. copy `hook.sh` to your hooks directory
2. edit `WORKSPACE` path in `hook.sh` to match your setup
3. add to `~/.openclaw/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Edit|Write|Glob|Grep",
        "hooks": ["/path/to/hooks/fs-guard/hook.sh"]
      }
    ]
  }
}
```

4. test: `Read ~/.openclaw/.env` should be blocked

## requirements

- `jq` for JSON processing
- `realpath` (coreutils) or `python3` as fallback
- works on both linux and macOS

## why not just use linux file permissions?

linux ACLs work but are fragile — one wrong `chmod` and you're exposed. fs-guard is:
- **declarative**: allowlist is in the script, visible, auditable
- **defense in depth**: works alongside unix permissions, not instead of
- **portable**: same hook works on any OS with bash+jq
- **native**: uses openclaw's own PreToolUse mechanism
