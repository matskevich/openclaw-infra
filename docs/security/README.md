# security — openclaw bot hardening

defense-in-depth для personal AI assistant на openclaw. 5 уровней, каждый ловит то что пропустил предыдущий.

```
Layer 1: PROMPT        — behavioral rules, anti-injection, role-based access
Layer 2: OS ISOLATION   — systemd hardening, egress filter (UFW), file permissions
Layer 3: EXEC SANDBOX   — bwrap namespace isolation + fs-guard + optional SOPS vault
Layer 4: OUTPUT DLP     — regex + entropy + known secrets scanning (post-send)
Layer 5: BLAST RADIUS   — spend limits, API key restrictions, rotation runbook
```

## docs

| doc | что | время |
|-----|-----|-------|
| [security-philosophy.md](security-philosophy.md) | архитектура defense-in-depth, threat model, gaps, lessons learned | 15 мин чтения |
| [exec-sandbox-playbook.md](exec-sandbox-playbook.md) | bwrap sandbox setup (recommended), allowlist (legacy), docker (not recommended) | 30 мин setup |

## tools

| tool | что |
|------|-----|
| secureclaw (openclaw-brain) | 42-check automated security audit, OWASP ASI mapping, `--telegram` reporter |

## scripts (в этом репо)

| script | что |
|--------|-----|
| [scripts/check-secrets.sh](../../scripts/check-secrets.sh) | grep по workspace на утечки ключей |
| [scripts/pre-commit-secrets-check.sh](../../scripts/pre-commit-secrets-check.sh) | pre-commit hook — блокирует коммит с секретами |

## hooks (в этом репо)

| hook | layer | что |
|------|-------|-----|
| [hooks/output-filter/](../../hooks/output-filter/) | 4 (DLP) | post-send detection секретов в исходящих сообщениях (regex + entropy + known secrets) |
| [hooks/memory-logger/](../../hooks/memory-logger/) | — | raw log всех message events (memory pipeline) |

## quick test: у вас есть проблема?

пошлите боту:

```
запусти: python3 -c "print('hello')"
```

- ответил `hello` без sandbox → [exec-sandbox-playbook.md](exec-sandbox-playbook.md)
- запросил approval → у вас allowlist mode
- команда в bwrap → уже защищены

## key insight (260216)

**docker sandbox ≠ universal solution.** openclaw's `sandbox.mode` moves workspaces → breaks memory, heartbeat, skills. **bwrap** wraps individual commands without relocating anything — recommended approach.

**exec approvals = friction, not enforcement.** bwrap provides real namespace isolation. approval popups killed bot autonomy without adding real security.
