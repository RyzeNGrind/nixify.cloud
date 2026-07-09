---
name: nixify.cloud
description: >
  Cloud and service-surface operations skill for the DAS cluster. Covers cloudflared
  tunnel ag-rc-2 operations, service runbooks (Plane, brandkit, ntfy, ops dashboard),
  chat-mirror payload schema, open issues (Q15-TUNNEL, Q14-E2E-2), DASx product hosting
  map, and Vercel vs self-host split. Use when operating, troubleshooting, or extending
  any public-facing DAS service on alpha or via CF Zero Trust.
status: live
created: 2026-07-09
sources:
  - DAS/Reports/PLANE-DEPLOY-2026-07-08.md
  - DAS/Reports/BRANDKIT-DEPLOY-2026-07-08.md
  - DAS/Reports/NTFY-SELF-HOSTED-2026-07-07.md
  - DAS/Reports/CF-TUNNEL-ROOT-CAUSE-FIXED-2026-07-07.md
  - DAS/Reports/CF-DNS-TUNNEL-DEPLOYMENT-2026-07-05.md
  - ~/Workspaces/CLAUDE.md (CHAT-MIRROR SOP §📡)
---

# nixify.cloud

Cloud and service-surface runbook for the DAS cluster. No secrets are stored here.
All CF dashboard operations require operator session unless CF_API_TOKEN_ACCOUNT is available.

---

## 1. Cloudflared Tunnel ag-rc-2 — Operations

**Tunnel ID:** `b185f8f2-f3fb-4ed2-b75d-0418804f7564`  
**Name:** `ag-rc-2`  
**Mode:** token-mode (`--token`), dashboard-managed ingress  
**Host:** alpha (100.125.115.81)  
**Token path:** `/var/lib/nixbuilder/.cloudflared/tunnel-token.env`  
**Cloudflared binary:** `/nix/store/xjwz6kv2f0fi1aw0cdlg1mw0mqqvcdff-cloudflared-2025.5.0/bin/cloudflared`  
**Systemd unit:** `/run/systemd/system/cloudflared.service` (runtime — not yet NixOS declarative; cleared on reboot)  
**Connections:** 4 to yyz Cloudflare PoPs, protocol QUIC  
**Account ID:** `dcf8e0d1ef14f6b24391b0c1ed2a9cee`

source: DAS/Reports/PLANE-DEPLOY-2026-07-08.md §5; DAS/Reports/CF-TUNNEL-ROOT-CAUSE-FIXED-2026-07-07.md

### Verify tunnel health

```bash
# From alpha:
systemctl status cloudflared.service
journalctl -u cloudflared.service -n 50 --no-pager

# From anywhere:
curl -sI https://ops.dasagency.ca/ | head -3  # expect 200 or 302 (CF Access)
```

### Restart tunnel

```bash
ssh alpha "systemctl restart cloudflared.service"
# If cleared on reboot, re-run token-mode start:
ssh alpha "cloudflared tunnel --token \$(cat /var/lib/nixbuilder/.cloudflared/tunnel-token.env) run &"
```

### Add public hostname (OPERATOR-GATED)

CF dashboard does not accept API mutations without Account-scoped token (CF_API_TOKEN_ACCOUNT — not yet minted, blocked on CF-1).

**Manual path (CF Zero Trust Dashboard):**
1. Navigate: CF Zero Trust → Networks → Tunnels → ag-rc-2 → Public Hostnames → Add
2. Fill: Subdomain, Domain (dasagency.ca or dasgroup.ca), Service type: HTTP, URL: `127.0.0.1:<port>`
3. Save → DNS CNAME auto-created; propagates in ~30s

**Agent-declarative path (status: pending — CF-1 blocker):**
```bash
# When CF_API_TOKEN_ACCOUNT is minted:
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/cfd_tunnel/${TUNNEL_ID}/configurations" \
  -H "Authorization: Bearer ${CF_API_TOKEN_ACCOUNT}" \
  -H "Content-Type: application/json" \
  -d '{"config":{"ingress":[{"hostname":"plane.dasagency.ca","service":"http://127.0.0.1:8082"},{"service":"http_status:404"}]}}'
```

### Known root-cause history

- **2026-07-07 P0 dual failure:** (A) OCI VCN blocked TCP 3000/3001 inbound; (B) `nixbuilder` session-exit killed services via `Exit the Session` systemd target.
  - Fix A: `iptables -I nixos-fw` allow TCP 3000/3001 from tailscale `100.64.0.0/10` + CF edge CIDRs.
  - Fix B: `loginctl enable-linger nixbuilder` (permanent: `users.users.nixbuilder.linger = true` in NixOS config).
- **Previous tunnels** (retired): `b5c6c5e8` (legacy, creds JSON + cert.pem era), `7ce35609`. ag-rc-2 is canonical.

source: DAS/Reports/CF-TUNNEL-ROOT-CAUSE-FIXED-2026-07-07.md

---

## 2. Service Runbooks

### Plane (project tracking) — alpha :8082

| Key | Value |
|---|---|
| URL | `https://plane.dasagency.ca` (status: CF 530 pending OPERATOR-STEP-1) |
| Local port | `127.0.0.1:8082` |
| Stack | Docker Compose at `/var/lib/plane/docker-compose.yml` |
| Domain | `APP_DOMAIN=plane.dasagency.ca` |
| RAM usage | ~1.2 GiB steady-state |
| Postgres | Container `plane-plane-db-1` (postgres:15.7-alpine), port 5432 |

**Smoke test:**
```bash
ssh alpha "curl -sI http://127.0.0.1:8082/" | head -3
# expect: HTTP/1.1 200 OK
```

**Start/stop:**
```bash
ssh alpha "cd /var/lib/plane && docker compose up -d"
ssh alpha "cd /var/lib/plane && docker compose down"
```

**Container status:**
```bash
ssh alpha "cd /var/lib/plane && docker compose ps"
```

**OPERATOR-STEP-1 (pending):** Add CF public hostname `plane → dasagency.ca → http://127.0.0.1:8082` in CF Zero Trust Dashboard.

**OPERATOR-STEP-2 (after step 1):** Mint `PLANE_API_TOKEN` via Plane UI → Settings → API Tokens → Create token (`das-ops-sync`). Required for harness dashboard sync (FPH-12).

**Backup gap:** No pg_dump cron exists. Add:
```bash
# On alpha as root:
echo '0 3 * * * docker exec plane-plane-db-1 pg_dump -U plane plane | gzip > /var/backups/plane/plane-$(date +%Y%m%d).sql.gz' | crontab -
```

source: DAS/Reports/PLANE-DEPLOY-2026-07-08.md

### Brandkit Generator — alpha :8083

| Key | Value |
|---|---|
| URL | `https://brandkit.dasagency.ca` (status: CF 530 pending OPERATOR-STEP-1) |
| Local port | `127.0.0.1:8083` |
| Image | `brandkit-generator:latest` (python:3.12-slim + Typst 0.12.0 aarch64-musl, 324 MB) |
| Runtime user | `brandkit` (UID 999, non-root) |
| RAM limit | 384 MiB |
| CPU limit | 0.5 vCPU |
| Data volumes | `/var/lib/brandkit/data`, `/var/lib/brandkit/clients` |
| Source branch | `das-grp/brandkit-generator @ claude/feat/webapp-api-ui @ 624cee9` |

**Smoke test:**
```bash
ssh alpha "curl -sI http://127.0.0.1:8083/" | head -3
# expect: HTTP/1.1 200 OK
```

**OPERATOR-STEP-1 (pending):** Add CF hostname `brandkit → dasagency.ca → http://127.0.0.1:8083`.

source: DAS/Reports/BRANDKIT-DEPLOY-2026-07-08.md

### ntfy (push notifications) — alpha :2586

| Key | Value |
|---|---|
| URL | `https://ntfy.dasagency.ca` (live) |
| Local port | `:2586` |
| Config | `/root/.config/ntfy/server.yml` (base-url: https, listen :2586, behind-proxy: true) |
| Binary | `ntfy-sh` v2.12.0 via `nix profile install nixpkgs#ntfy-sh` (root nix profile) |
| Systemd | `/root/.config/systemd/user/ntfy-sh.service` |
| Topic | `das-agents-e9c17fe926da80ea` |

**Subscribe (phone):** `https://ntfy.dasagency.ca/das-agents-e9c17fe926da80ea`

**Known issue Q14-E2E-2:** ntfy root URL `https://ntfy.dasagency.ca/` returns 404 (no index route). Operator notifications via topic slug still work; root 404 is cosmetic but fails E2E check.

**Restart:**
```bash
ssh alpha "systemctl --user restart ntfy-sh.service"
```

source: DAS/Reports/NTFY-SELF-HOSTED-2026-07-07.md

### ops.dasagency.ca (operator dashboard) — alpha

| Key | Value |
|---|---|
| URL | `https://ops.dasagency.ca` (live, CF Access magic-link auth) |
| CF tunnel | ag-rc-2, hostname `ops → dasagency.ca` |
| Repo | `DAS-Grp/dasagency.ca` (apps/web/) |
| Status | Live (2026-07-06) |

**Chat-mirror API card — `/api/ops/chat-mirror`**

Every agent turn mirrors a JSON payload here. Payload schema (≤4 KB):

```json
{
  "ts": "<ISO-8601-UTC>",
  "session_id": "<uuid>",
  "host": "<hostname>",
  "repo": "<repo-slug>",
  "branch": "<branch-name>",
  "turn_kind": "TaskCreate|TaskUpdate|TaskDelete|text",
  "summary_line": "<one-line headline ≤200 chars>",
  "delta": "<full turn text, truncated at 4 KB>",
  "refs": {
    "prs": [],
    "tags": [],
    "task_ids": [],
    "commit_shas": []
  },
  "status": "in_progress|shipped|blocked|needs_input"
}
```

Enforcement: `Stop` hook at `~/.claude/settings.json` runs `~/Workspaces/.agents-gate/chat-mirror.sh` (non-blocking curl POST). Failures logged to `~/.claude/logs/chat-mirror.jsonl`. A lapsed mirror = same severity as a lapsed validation marker (DoD rule).

Redaction: `chat-mirror.sh` strips lines matching `password|secret|api_key|token=|PRIVATE KEY|SOPS_AGE_KEY` before POST.

source: ~/Workspaces/CLAUDE.md §CHAT-MIRROR SOP

---

## 3. DASx Product Hosting Map

| Product | Domain | Repo | Host | Deploy method | Status |
|---|---|---|---|---|---|
| DAS Agency site (village, launchpad, security) | dasagency.ca | DAS-Grp/dasagency.ca | alpha (via ws cloudflared) | Docker + CF tunnel ag-rc-2 | Live |
| DAS Group site (cybersecops) | dasgroup.ca | DAS-Grp/dasgroup.ca | alpha (via ws cloudflared) | Docker + CF tunnel ag-rc-2 | Live |
| ops dashboard | ops.dasagency.ca | DAS-Grp/dasagency.ca | alpha | CF tunnel ag-rc-2 | Live |
| Plane (project tracking) | plane.dasagency.ca | self-hosted Docker on alpha | alpha :8082 | Docker Compose | **status: pending** (CF 530, Q15-TUNNEL) |
| Brandkit generator | brandkit.dasagency.ca | DAS-Grp/brandkit-generator | alpha :8083 | Docker | **status: pending** (CF 530, Q15-TUNNEL) |
| ntfy (push notifications) | ntfy.dasagency.ca | ntfy-sh (nix profile) | alpha :2586 | systemd + CF tunnel | Live (root 404 known) |
| NexVerse / village 3D | nexverse.dasagency.ca | DAS-Grp/dasagency.ca | Vercel (CNAME vercel-dns.com) | Vercel auto-deploy | Pending Vercel project |
| Forgejo (git primary) | — (tailnet only: beta:3000) | self | beta :3000 | Standalone | Live (no public hostname — by design) |

source: DAS/Reports/CF-DNS-TUNNEL-DEPLOYMENT-2026-07-05.md; DECISIONS-2026-07-02.md §Domains

---

## 4. Vercel vs Self-Host Split

**Vercel (Hobby tier):** Marketing/landing pages where CDN edge performance matters and origin infra cost is avoided. Commit-per-deploy; preview URLs per PR. Governed by `DECISIONS-2026-07-02.md §7 v2 — Vercel Hobby discipline` (no overages, no team seats).

**Self-hosted on alpha:** Stateful services (Plane, brandkit API, btcpay, n8n, listmonk) — require persistent volumes, long-lived processes, or privileged networking.

**Rule of thumb:** static/SSG → Vercel; API + DB + Docker → alpha (via CF tunnel, no public OCI port).

---

## 5. Zone Map (CF DNS)

| Zone | CF Zone ID |
|---|---|
| dasagency.ca | 3b5f05f9dc7ee928006c2b0e7e0e8435 |
| dasgroup.ca | 40e2e99669cecc5aaae4075b7a446646 |
| dasdope.com | fb51e953da72273cf2f017f6bc886040 |
| dasmed.ca | 2b8593bd9a8ea44f03797b7bb3980c13 |
| digitalassetsyndicate.com | 9c0f4ecb4f9a5e6de7cfd166dd29fd80 |

Token note: current `CF_API_TOKEN` is Zone-scoped (read/write DNS records). Tunnel CRUD + public hostname management requires Account-scoped token (`CF_API_TOKEN_ACCOUNT` — not yet minted, CF-1 blocker).

source: DAS/Reports/CF-DNS-TUNNEL-DEPLOYMENT-2026-07-05.md §Zone map

---

## 6. Known Open Issues

| ID | Description | Status | Operator action required |
|---|---|---|---|
| Q15-TUNNEL | plane.dasagency.ca + brandkit.dasagency.ca return CF 530 | **pending** | Add public hostnames in CF Zero Trust Dashboard (5 min) |
| Q14-E2E-2 | ntfy.dasagency.ca root returns 404 | **pending** | Add ntfy index route or accept as cosmetic |
| CF-1 | CF_API_TOKEN_ACCOUNT not yet minted | **blocked** | Operator mints Account-scoped token in CF Dashboard → API Tokens → Create Token (scope: Account, Cloudflare Tunnel: Edit) |
| FPH-14 | cloudflared token-mode NixOS module on alpha (A11) | staged-for-review | Merge claude branch after validate-host alpha |
| FPH-12 | ops dashboard substrate visibility (`/api/ops/substrate`) | todo | Requires PLANE_API_TOKEN (OPERATOR-STEP-2) |
| CROSS-04 | ntfy.dasagency.ca root 404 breaks harness push notifications | MEDIUM | Investigate ntfy root config |
| CROSS-03 | plane.dasagency.ca live needed for harness Plane sync | blocked on Q15-TUNNEL | OPERATOR-STEP-1 first |

---

## 7. Alpha Resource Headroom (2026-07-09 baseline)

| Metric | Value |
|---|---|
| RAM total | 23 GiB (A1.Flex, 3 OCPU / 18 GB allocated) |
| RAM used | ~6 GiB (Plane ~1.2 GiB + brandkit ~384 MiB + cloudflared ~100 MiB + system) |
| RAM free | ~17 GiB available |
| Disk total | ~48 GiB (boot + docker overlay) |
| Disk used | ~32 GiB |
| Disk free | ~17 GiB |

Do not add new Docker services above 1 GiB RAM without operator capacity sign-off. Respect §8 no-new-volumes discipline (oracle-infra-mgmt-devops SKILL.md §4).
