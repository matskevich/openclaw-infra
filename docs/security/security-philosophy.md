# Security Philosophy & Architecture

## Personal AI Agent on Dedicated Infrastructure

**Template version** — sanitized from production deployment.
Adapt IPs, usernames, groups, and verification commands to your setup.

**Last updated:** 2026-02-16

---

## AI-First Reader Guide (start here)

If you are an AI helper/reviewer, read in this order:
1. **TL;DR State** (below)
2. **Guarantees + Gaps** (Sections III, VI)
3. **Verification Commands** (Section IX)
4. **secureclaw** — automated audit (42 checks, `node skills/secureclaw/secureclaw-audit.mjs audit --telegram`)

If anything conflicts with reality, update the doc. This file is the
canonical security truth.

---

## TL;DR State (2026-02-16)

- **Defense-in-depth**: prompt rules + **bwrap sandbox** (namespace isolation) + **fs-guard** (workspace-only file tools) + output DLP + optional **SOPS vault** (no secrets on disk).
- **Exec approvals**: can be enabled (`ask: "on-miss"`) or disabled (`ask: "off"`). With bwrap, approvals are redundant — bwrap provides real enforcement.
- **Docker sandbox**: tested and **NOT recommended** — incompatible with openclaw agent architecture (breaks memory, heartbeat, skills).
- **Mount-namespace hardening NOT working** for user services (documented, replaced by bwrap).
- **secureclaw**: 42-check automated audit skill, OWASP ASI mapping, community distribution via openclaw-brain.
- **Agent-first principle**: agent acts autonomously by default within hard boundaries. Security layers enforce WITHOUT blocking legitimate work.

---

## Non-Negotiables (AI helpers)

- **No consumer OAuth / setup-token** for automation.
- **Never read or output** `~/.openclaw/.env` or secrets.
- **Do not disable** DLP hooks, bwrap sandbox, or fs-guard.
- **If unsure** about risk, ask the owner before acting.

---

## I. Philosophy

### The core tension

An AI agent is useful precisely because it has access: to your files, your APIs, your conversations, your server. Strip the access — the agent becomes useless. Grant full access — a single jailbreak compromises everything.

The standard industry answer is "don't give agents access to secrets." This is correct at scale (5+ agents, multiple owners, enterprise). For a personal AI assistant running 24/7 on a dedicated VPS with one owner — it's the wrong frame. The agent needs API keys to call models. It needs the bot token to send Telegram messages. It needs filesystem access to maintain memory. **The question isn't whether the agent has access. It's what happens when the agent is compromised.**

### Assume breach

We don't design for "the agent will never be jailbroken." We design for "when the agent is jailbroken, the blast radius is contained." This is defense-in-depth — multiple independent layers, each catching what the previous one missed.

### Agent-first security

Security must enable, not cripple. Each layer must add protection WITHOUT blocking legitimate work. If a security measure kills the agent's core functionality — it's misconfigured, not "strict."

**The test:** after deploying any security layer, can the bot still perform its daily tasks (memory operations, scheduled tasks, arena communication) without human intervention? If not, the layer needs adjustment.

**Learned the hard way:** exec approval popups (every Bash = telegram approval) killed bot autonomy. bwrap provides stronger isolation without the friction.

### Honest accounting

Most security documentation describes what's protected. Ours also describes what isn't. We document every known gap, every failed mitigation, every limitation we've discovered. Security theater is worse than no security — it creates false confidence.

---

## II. Threat Model

### What we're protecting

| Asset | Impact if compromised | Likelihood |
|-------|----------------------|------------|
| API keys (Anthropic, OpenAI, Groq, Gemini, etc.) | Financial ($100-500/mo spend), service abuse | Medium |
| Telegram bot token | Bot impersonation, message interception | Medium |
| SSH keys | Full server access, lateral movement | Low |
| Conversation history / memory | Privacy breach | Low-Medium |
| Server (exec access) | Arbitrary code execution, pivot point | Medium |

### Who attacks

1. **Untrusted users in Telegram groups** — social engineering, prompt injection via group messages
2. **Malicious content** — injected instructions in web pages, forwarded messages, file attachments
3. **Supply chain** — compromised skills, malicious tools, bad npm packages
4. **The agent itself** — emergent behavior, confused context, hallucinated commands

### Attack vectors, ranked

```
1. Prompt injection via Telegram     [likelihood: 6/10, impact: 10/10]
2. Credential exfiltration           [likelihood: 4/10, impact: 8/10]
3. Malicious skill / supply chain    [likelihood: 2/10, impact: 8/10]
4. SSH key theft                     [likelihood: 1/10, impact: 10/10]
```

---

## III. Architecture: Defense in Depth

```
                        ┌──────────────────────┐
 telegram message ──────► Layer 1: PROMPT       │  Behavioral control
                        │ SECURITY.md rules     │  Role-based access
                        │ Anti-injection detect  │  Allowlists
                        └──────────┬─────────────┘
                                   │
                        ┌──────────▼─────────────┐
                        │ Layer 2: OS ISOLATION   │  Systemd hardening
                        │ Egress filter (UFW)     │  NoNewPrivileges
                        │ File permissions        │  Process restrictions
                        └──────────┬─────────────┘
                                   │
                        ┌──────────▼─────────────┐
                        │ Layer 3: EXEC SANDBOX   │  bwrap namespace isolation
                        │ + FS-GUARD              │  Workspace-only file access
                        │ + SOPS VAULT (optional) │  No secrets on disk
                        └──────────┬─────────────┘
                                   │
                        ┌──────────▼─────────────┐
                        │ Layer 4: DETECTION      │  Output DLP hook
                        │ Regex + Entropy + Known │  Post-send alerting
                        │ Secrets scanning        │  secureclaw audit
                        └──────────┬─────────────┘
                                   │
                        ┌──────────▼─────────────┐
                        │ Layer 5: BLAST RADIUS   │  Spend limits
                        │ API key restrictions    │  IP locks
                        │ Rotation runbook        │  Immediate response
                        └────────────────────────┘
```

### Layer 3: bwrap Sandbox + fs-guard (recommended)

**What it does:** OS-level namespace isolation for exec, workspace-only file access.

**bwrap exec sandbox** (PreToolUse hook):
- Every Bash command wrapped in bubblewrap with namespace isolation
- `.openclaw/`, `.ssh/`, `secrets/`, `/run/` — hidden via tmpfs overlay
- `~/clawd/` — writable (workspace)
- No access to secrets even if prompt injection succeeds

**fs-guard** (PreToolUse hook):
- Built-in file tools (Read/Edit/Write/Glob/Grep) restricted to workspace + /tmp
- Realpath resolution + `..` traversal protection

**SOPS+age vault** (optional):
- `.env` encrypted at rest, decrypt to tmpfs at startup
- Plaintext `.env` shredded from disk

**What it catches:**
```bash
cat ~/.openclaw/.env           # BLOCKED (bwrap hides .openclaw)
fs.read ~/.openclaw/.env       # BLOCKED by fs-guard
python3 -c "import os; ..."    # runs in namespace, no secrets visible
```

**What it DOESN'T catch:**
- `process.env` in RAM (needed for API calls, inherent to architecture)

Setup: **[exec-sandbox-playbook.md](exec-sandbox-playbook.md)**

### Docker sandbox — NOT recommended

**Tested and removed.** Both `sandbox.mode: "all"` and `"non-main"` break openclaw:
- `"all"` moves ALL agent workspaces → breaks memory, heartbeat, skills
- `"non-main"` sandboxes embedded agents that need workspace → backwards protection
- bwrap achieves same isolation without workspace relocation

---

## VI. Known Gaps (Honest)

### Medium: process.env in RAM
All API keys live in `process.env`. bwrap isolates spawned commands but the main process retains env vars.

### Medium: Post-send DLP
Output DLP fires after message delivery. Detection, not prevention.

### Low: Config mutability
Bot can modify `openclaw.json` via internal fs.write tool. Config watchdog detects and reverts (5-min window).

---

## VII. The Meta Question

### Why not vault/gateway?

**Correct architecture — for scale.** For 1-2 bots on 1 server, bwrap+fs-guard+SOPS vault achieves 90% of vault/gateway benefit at 10% of the complexity.

**Decision:** bwrap approach for 1-2 bots. Full vault/gateway when scaling to 5+ agents.

### Philosophy in one line

**We don't build perfect walls. We build layers where each one catches the failure of the previous one, and we honestly document which walls are actually made of cardboard.**

---

## VIII. Roadmap

| Priority | Item | Status |
|----------|------|--------|
| Done | bwrap sandbox + fs-guard | Recommended |
| Done | secureclaw v1.3.0 (42-check audit) | Available |
| Done | Output DLP hook | Active |
| Done | Egress filter (port-level) | Active |
| Recommended | SOPS vault (optional) | Available |
| Recommended | Config watchdog | Available |
| Not recommended | Docker sandbox | Incompatible with openclaw |
| Dead | InaccessiblePaths/PrivateTmp | Non-functional for user services |

---

## IX. Verification Commands

Adapt SSH user and server IP to your setup:

```bash
SERVER="youruser@your.server.ip"

# 1. Check bwrap hooks registered
ssh $SERVER 'cat ~/.openclaw/settings.json | python3 -m json.tool'
# Expected: sandbox-exec hook for Bash, fs-guard hook for Read|Edit|Write|Glob|Grep

# 2. Check no plaintext .env on disk (if using vault)
ssh $SERVER 'ls -la ~/.openclaw/.env 2>&1'
# Expected: No such file or directory

# 3. Check UFW egress rules
ssh $SERVER 'sudo ufw status verbose'
# Expected: Default deny outgoing, whitelist of allowed ports

# 4. Check systemd hardening
ssh $SERVER 'systemctl --user show YOUR_SERVICE | grep -E "NoNew|Restrict|Limit|UMask"'
# Expected: NoNewPrivileges=yes, RestrictSUIDSGID=yes, LimitCORE=0

# 5. Run secureclaw audit
ssh $SERVER 'cd ~/clawd && node skills/secureclaw/secureclaw-audit.mjs audit --telegram'
# Expected: Score ≥ 70, 0 CRITICAL

# 6. Test blocked egress
ssh $SERVER 'curl -s --max-time 3 http://example.com:8080 2>&1'
# Expected: Connection timeout (blocked)
```

---

## X. Lessons Learned

1. **Verify, don't trust config.** systemd `InaccessiblePaths` was silently ignored for user services. Always test from the process's perspective.

2. **User services are second-class citizens.** Mount namespaces, capability bounding — none work without CAP_SYS_ADMIN. bwrap provides the isolation that systemd cannot.

3. **Docker sandbox ≠ universal solution.** openclaw's sandbox.mode moves workspaces — breaks memory, heartbeat, skills. bwrap wraps individual commands without relocating anything.

4. **Security that kills functionality = sabotage.** Exec approval popups blocked every Bash call. bwrap provides stronger isolation without the friction.

5. **Agent-first security.** The agent should operate autonomously within hard boundaries. Approvals only for genuinely dangerous operations.

6. **Document what doesn't work.** False security documentation is an active hazard.

7. **Honest self-audit beats manual review.** secureclaw (42 automated checks) catches things humans miss.
