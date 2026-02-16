# exec sandbox playbook (openclaw)

**что:** изолировать exec команды от секретов и чувствительных файлов.
**зачем:** prompt injection → `cat ~/.openclaw/.env` → ключи в чате. без sandbox любой jailbreak = full compromise.
**когда:** 30 минут setup, без downtime.

---

## TL;DR — рекомендуемый подход (260216)

```
bwrap sandbox      → каждый Bash в namespace isolation (скрывает .openclaw/.ssh/secrets)
fs-guard           → Read/Edit/Write/Glob/Grep только workspace + /tmp
SOPS vault         → .env зашифрован, декрипт в tmpfs (optional)
exec approvals     → optional (bwrap = реальная граница)
docker sandbox     → НЕ РЕКОМЕНДУЕТСЯ (ломает openclaw agent flow)
```

---

## тест: у вас вообще проблема?

пошлите боту:

```
запусти: python3 -c "print('hello')"
```

- ответил `hello` без sandbox → **у вас проблема**, читайте дальше
- команда выполнилась в bwrap namespace → sandbox работает
- запросил approval → allowlist+approval mode (можно перейти на bwrap)

---

## вариант 1: bwrap sandbox (рекомендуемый)

OS-level namespace isolation через PreToolUse hooks. каждая Bash команда оборачивается в bubblewrap — скрывает секреты, ограничивает файловую систему.

### что нужно

- Linux (Ubuntu 22.04+)
- `bubblewrap` installed (`apt install bubblewrap`)
- openclaw с поддержкой PreToolUse hooks (settings.json)

### что даёт

| вектор | защита |
|--------|--------|
| `cat ~/.openclaw/.env` | BLOCKED (bwrap скрывает .openclaw через tmpfs) |
| `python3 -c "import os; print(os.environ)"` | видит только sandbox env, не host |
| fs.read ~/.openclaw/.env | BLOCKED (fs-guard, вне workspace) |
| fs.read ~/.ssh/id_rsa | BLOCKED (fs-guard, вне workspace) |

### setup (коротко)

1. установить bwrap: `sudo apt install bubblewrap`
2. на Ubuntu 24.04: создать AppArmor профиль для bwrap (unprivileged userns restriction)
3. создать PreToolUse hooks: sandbox-exec (для Bash) и fs-guard (для Read/Edit/Write/Glob/Grep)
4. зарегистрировать hooks в `~/.openclaw/settings.json`
5. рестарт бота
6. проверить: `node skills/secureclaw/secureclaw-audit.mjs audit --telegram`

### gotchas

- **Ubuntu 24.04 AppArmor:** блокирует unprivileged user namespaces → нужен профиль `/etc/apparmor.d/bwrap`
- **ProtectSystem=strict:** может блокировать доступ к age keys → хранить вне /etc
- **systemd EnvironmentFile ordering:** загружается ДО ExecStartPre → использовать `-` prefix

---

## вариант 2: allowlist + approval (лёгкий, legacy)

> **NOTE:** allowlist + approval popups = friction без real enforcement. bwrap даёт лучшую изоляцию без friction. используйте allowlist как fallback если bwrap недоступен.

### exec-approvals.json

```bash
cat > ~/.openclaw/exec-approvals.json << 'EOF'
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny"
  },
  "agents": {
    "*": {
      "allowlist": [
        { "pattern": "/usr/bin/git" },
        { "pattern": "/usr/bin/ls" },
        { "pattern": "/usr/bin/mkdir" },
        { "pattern": "/usr/bin/cp" },
        { "pattern": "/usr/bin/mv" },
        { "pattern": "/usr/bin/date" },
        { "pattern": "/usr/bin/diff" },
        { "pattern": "/usr/bin/stat" },
        { "pattern": "/usr/bin/jq" }
      ]
    }
  }
}
EOF
chmod 600 ~/.openclaw/exec-approvals.json
```

### CRITICAL: формат файла

```
"version": 1          ← ОБЯЗАТЕЛЬНО. без этого парсер отбрасывает файл
"defaults": { ... }   ← security/ask на уровне дефолтов
allowlist entries — объекты { "pattern": "/usr/bin/git" }, НЕ строки
```

### gotchas

- **self-escalation:** бот БУДЕТ обходить sandbox. доказано на production: обошёл 3 способами за 1 сессию (добавил bash в allowlist, отключил ask, отключил approvals). **fix:** config watchdog (5-мин cron, auto-revert + alert)
- **`agents.main` перезатирается** при старте — дублировать allowlist в `*` и `main`
- **approval friction:** каждый non-allowlisted cmd = telegram popup. с bwrap это не нужно

---

## вариант 3: docker sandbox — НЕ РЕКОМЕНДУЕТСЯ

> **TESTED AND REMOVED.** docker sandbox breaks openclaw agent architecture.

- `sandbox.mode: "all"` → перемещает workspace ВСЕХ агентов → ломает memory, heartbeat, skills
- `sandbox.mode: "non-main"` → sandboxes embedded agents которые НУЖНЫ в workspace → backwards protection
- container isolation РАБОТАЕТ (0 secrets) — но workspace relocation = fatal

если нужна container-level изоляция → bwrap (вариант 1).

---

## secureclaw — automated audit

после настройки sandbox, проверьте конфигурацию автоматически:

```bash
# установить (из openclaw-brain)
cp -r skills/secureclaw/ ~/clawd/skills/secureclaw/

# запустить аудит
node ~/clawd/skills/secureclaw/secureclaw-audit.mjs audit --telegram

# сохранить в файл
node ~/clawd/skills/secureclaw/secureclaw-audit.mjs audit --telegram --file /tmp/secureclaw-report.txt
```

42 проверки: gateway, credentials, execution, access-control, supply-chain, memory, cost, IOC.

---

## что каждый вариант защищает

| вектор | bwrap | allowlist+approval | docker |
|--------|-------|-------------------|--------|
| exec → read .env | **BLOCKED** (hidden) | BLOCKED (cat not in list) | **BLOCKED** (no .env) |
| exec → arbitrary code | runs in namespace | approval required | runs in container |
| fs.read → .env | **BLOCKED** (fs-guard) | NOT BLOCKED | NOT BLOCKED |
| workspace breaks? | **NO** | NO | **YES** (fatal) |
| approval friction? | **NONE** | HIGH | NONE |

**рекомендация:** bwrap (вариант 1) — лучшая изоляция, нет friction, нет workspace проблем.
