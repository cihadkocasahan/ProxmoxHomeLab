# Home Lab Infrastructure & AI Control Plane

> A self-hosted Proxmox VE environment managed entirely through a custom MCP server —
> exposing every infrastructure operation as a callable AI tool.

This document describes the home lab I designed, built, and operate — covering physical infrastructure, service topology, automation architecture, and a real-world bot use case. It is part of my [technical portfolio](https://github.com/cihadkocasahan).

**Diagrams:**
- [01 — Full Network Topology](diagrams/01_topology.md)
- [02 — MCP Control Plane](diagrams/02_mcp_control_plane.md)
- [03 — Scraper Bot Flow](diagrams/03_scraper_bot_flow.md)
- [04 — deploy_service() Provisioning](diagrams/04_deploy_service_flow.md)

---

## Why I Built This

Two questions I bring to every engineering problem: *"How can I automate this instead of doing it manually? What tools can I build to make this easier?"*

Managing a multi-service home lab the traditional way means SSH sessions, manual clicks in web UIs, and constant context-switching. My answer was to build a single AI-controllable control plane over everything — so that spinning up a service, checking health, managing DNS, deploying a scraper bot, or committing to a Git repo all happen through the same interface: a conversation with an AI agent.

---

## Physical Node

```
Proxmox VE — node: pv
├── RAM:     30.3 GB total / ~8.8 GB used (idle)
├── Storage: local-zfs  (ZFS pool)  179.5 GB
│            local      (dir)       104.8 GB
│            MigrationBackup (dir)  116.8 GB
└── Network: vmbr0 bridge — 192.168.1.0/24
             Gateway: 192.168.1.1
             DNS: AdGuard Home (192.168.1.28)
```

---

## Infrastructure Topology

```
192.168.1.0/24 — Home LAN
│
├── 192.168.1.123  Proxmox VE Host (pv)
│
├── LXC 100  │ Gitea-Server          │ 192.168.1.25  │ 1c / 1GB  / 30GB
├── LXC 110  │ Shadow-SVN            │ 192.168.1.26  │ 1c / 1GB  / 20GB
├── LXC 120  │ Nginx-Proxy-Manager   │ 192.168.1.27  │ 2c / 1GB  /  8GB
├── LXC 130  │ AdGuard-Home          │ 192.168.1.28  │ 1c / 512M /  8GB
├── LXC 141  │ docker-host-default   │ 192.168.1.141 │ 4c / 4GB  / 20GB  ← N8N + RabbitMQ + Mem0
├── LXC 160  │ agent-proxmox         │ 192.168.1.60  │ 4c / 512M /  8GB  ← ProxmoxMCP control plane
├── LXC 180  │ hermes-agent          │ 192.168.1.80  │ 2c / 4GB  /  8GB  ← Hermes messaging
│
├── VM  200  │ Scraper-Donor-Std-200 │ —             │ 2c / 2GB  / 40GB  (stopped — clone template)
├── VM  201  │ scraper-worker-01     │ —             │ 2c / 2GB  / 40GB  (running — cloned from 200)
└── VM  210  │ Android-Malware-Analyzer │ —          │ —                  (stopped)
```

**All LXCs are unprivileged, onboot=1, DNS → AdGuard Home (192.168.1.28).**

---

## Network & Routing Architecture

Every service on the LAN is reachable via a human-friendly `.cihad.local` domain. The routing path is:

```
Browser / AI agent
        │
        ▼  DNS query: git.cihad.local
┌─ AdGuard Home (192.168.1.28) ───────────────────────────────────────────┐
│  DNS rewrite: *.cihad.local  →  192.168.1.27 (NPM)                     │
└─────────────────────────────────────────────────────────────────────────┘
        │
        ▼  HTTP/HTTPS request to 192.168.1.27
┌─ Nginx Proxy Manager (LXC 120, :81) ────────────────────────────────────┐
│  git.cihad.local        →  192.168.1.25:80   (Gitea)                   │
│  n8n.cihad.local        →  192.168.1.141:5678 (N8N via Docker)         │
│  svn.cihad.local        →  192.168.1.26:13081 (Shadow-SVN)             │
│  adguard.cihad.local    →  192.168.1.28:3000  (AdGuard dashboard)      │
│  npm.cihad.local        →  192.168.1.27:81    (NPM itself)             │
│  mcp.cihad.local        →  192.168.1.60:8000  (ProxmoxMCP)             │
│  hub.mcpimpact.cihad.local → 192.168.1.60:8000 (MCPImpact Hub)        │
│  hermes.cihad.local     →  192.168.1.80:9119  (Hermes UI)              │
│  pv.cihad.local         →  192.168.1.123:8006 (Proxmox web UI)        │
│  devhost.cihad.local    →  192.168.1.35:80    (Dev host)               │
└─────────────────────────────────────────────────────────────────────────┘
```

Adding a new service = one `deploy_service()` MCP tool call → LXC created + DNS rewrite + NPM proxy, all in one step.

---

## LXC 141 — The Docker Host

The old topology had separate LXCs for N8N (140), RabbitMQ (151), and Mem0 (141 pgvector). The current production setup consolidates all Docker workloads into **a single 4-core / 4 GB LXC** running multiple containers:

```
LXC 141 — docker-host-default (192.168.1.141)
4 cores / 4 GB RAM / 20 GB ZFS / unprivileged + nesting + keyctl + fuse
│
└── Docker
    ├── N8N            :5678  — workflow automation engine
    ├── RabbitMQ       :5672/:15672 — message broker
    └── Mem0           :8888  — semantic vector memory (PostgreSQL 17 + pgvector)
```

**Design rationale:** Three lightly-used services share one LXC to conserve RAM. The `nesting=1` + `fuse=1` + `keyctl=1` feature flags are required for Docker-in-LXC to work reliably on Proxmox.

---

## The Control Plane: ProxmoxMCP (LXC 160)

```
LXC 160 — agent-proxmox (192.168.1.60)
4 cores / 512 MB RAM / 8 GB ZFS / unprivileged + nesting
│
└── Docker
    └── proxmox-mcp container
        ├── FastMCP  → /sse          MCP over SSE (AI client entry point)
        ├── FastAPI  → /call         REST tool execution
        │            → /status       health check
        │            → /credentials  runtime secret management
        └── Dashboard → /dashboard
```

The entire server bootstraps in **46 lines** (`main.py`):

```python
sm     = SecretManager()                          # ENV-first, SQLite fallback
client = ProxmoxClient(sm)                        # proxmoxer API + SSH wrapper
ag     = AdGuardManager(sm)                       # AdGuard REST
npm    = NPMManager(sm)                           # Nginx PM REST
gitea  = GiteaManager(sm)                         # Gitea REST
svn    = SVNManager(sm, exec_fn=client.exec_in_lxc)

mcp = FastMCP("CihadEnv-Proxmox-MCP")
register_all(mcp, pve, client, ag, npm, gitea, svn)   # 8 tool modules, 60+ tools
```

**Credential architecture:** No secrets in code. `SecretManager` loads from `Master-Credentials.env` at startup (ENV primary, SQLite `/data/secrets.db` fallback). Runtime update without restart:

```bash
curl -X POST http://192.168.1.60:8000/credentials \
  -H "Content-Type: application/json" \
  -d '{"key": "PVE_PASS", "value": "new_password"}'
```

---

## MCP Tool Catalog

```
proxmox-mcp server (60+ tools across 8 modules)
│
├── proxmox_tools.py        VM/LXC lifecycle — create, power, snapshot, exec
├── orchestration_tools.py  Cross-service ops — deploy_service, infra_status
├── adguard_tools.py        DNS management
├── npm_tools.py            Reverse proxy management
├── gitea_tools.py          Git server operations
├── svn_tools.py            SVN + sync engine
├── mem0_tools.py           Semantic vector memory
└── impact_tools.py         MCPImpact distributed agent mesh
```

### Proxmox VM/LXC Management

| Tool | What it does |
|------|-------------|
| `list_resources` | All LXCs/VMs with live status and tags |
| `get_resource_details` | Full config + runtime stats (CPU, RAM, disk, network) |
| `manage_power` | `start` / `stop` / `shutdown` / `reboot` |
| `create_lxc` | Provision new Ubuntu 24.04 LXC (RAM, disk, IP, template configurable) |
| `remove_resource` | Permanently delete LXC or VM |
| `clone_resource` | Full or linked clone (VM 200 → VM 201 scraper workers use this) |
| `create_snapshot` / `list_snapshots` | Snapshot lifecycle |
| `backup_resource` | vzdump to storage |
| `set_tags` | Tag resources (e.g. `scraper;worker`, `infra;shadow`) |
| `get_node_stats` | Live CPU / RAM for Proxmox node |
| `get_storage_info` | ZFS pool + dir storage usage |
| `exec_in_container` | Run any shell command inside an LXC via `pct exec` |

### Cross-Service Orchestration

The highest-level tools — they compose multiple API calls atomically:

```
deploy_service(vmid=170, name="monitor-bot", ip="192.168.1.70",
               domain="monitor.cihad.local", forward_port=8080)

→  [OK] LXC 170 (monitor-bot) created at 192.168.1.70
→  [OK] DNS: monitor.cihad.local → 192.168.1.27
→  [OK] NPM: monitor.cihad.local → 192.168.1.70:8080
```

| Tool | What it does |
|------|-------------|
| `infra_status` | Health check: PVE LXC status + HTTP probe for all services |
| `deploy_service` | Create LXC + DNS rewrite + NPM proxy host (3-step atomic) |
| `destroy_service` | Stop + delete LXC + remove DNS + remove NPM (full teardown) |
| `register_service_domain` | Add DNS + proxy for existing LXC, no LXC touch |

### DNS — AdGuard Home

| Tool | What it does |
|------|-------------|
| `adguard_status` | Filtering state, query stats, top blocked domains |
| `adguard_add_rewrite` / `adguard_delete_rewrite` | DNS rewrite CRUD |
| `adguard_list_filters` / `adguard_add_filter` / `adguard_remove_filter` | Blocklist management |
| `adguard_get_custom_rules` / `adguard_set_custom_rules` | Custom filter rules |
| `adguard_check_host` | Is this hostname blocked? |
| `adguard_set_protection` | Global enable / disable |
| `adguard_clear_cache` | Flush DNS cache |

### Reverse Proxy — Nginx Proxy Manager

| Tool | What it does |
|------|-------------|
| `npm_list_proxy_hosts` / `npm_add_proxy_host` / `npm_delete_proxy_host` | Proxy host CRUD |
| `npm_enable_proxy_host` / `npm_disable_proxy_host` | Toggle routes |
| `npm_list_redirects` / `npm_add_redirect` | HTTP redirect rules |
| `npm_list_certificates` | SSL certificate status |

### Git — Gitea

| Tool | What it does |
|------|-------------|
| `gitea_list_repos` / `gitea_create_repo` / `gitea_delete_repo` | Repository CRUD |
| `gitea_list_branches` / `gitea_create_branch` | Branch management |
| `gitea_list_issues` / `gitea_create_issue` | Issue tracking |
| `gitea_list_releases` | Release management |
| `gitea_add_webhook` | Webhook registration |

### SVN — Shadow-SVN

| Tool | What it does |
|------|-------------|
| `svn_list_repos` / `svn_repo_info` / `svn_get_log` / `svn_list_path` | Repository inspection |
| `svn_create_repo` / `svn_dump_repo` | Repository lifecycle |
| `svn_sync_status` / `svn_sync_trigger` / `svn_sync_logs` | Sync engine control |
| `svn_container_status` / `svn_restart_container` | Docker-in-LXC management |

### Semantic Memory — Mem0 + pgvector

Self-hosted vector memory inside LXC 141: PostgreSQL 17 + pgvector + Gemini embeddings.

| Tool | What it does |
|------|-------------|
| `mem0_search` | Vector similarity search, user-scoped |
| `mem0_add` | Store a memory with category + agent metadata |
| `mem0_list_recent` | Recent memories by user ID |
| `mem0_delete` | Delete by memory ID |

AI agents use this to persist and retrieve context across sessions — effectively giving the agent long-term memory.

### MCPImpact — Distributed Agent Mesh

A separate layer: multiple machines running agent processes, coordinated by a hub.

| Tool | What it does |
|------|-------------|
| `hub_node_list` | All connected nodes: status, role, uptime, LLM state |
| `hub_node_capabilities` | Active tools on a specific node |
| `hub_tool_call` | Execute a tool on a node, wait for result |
| `hub_llm_ask` | Send LLM prompt to a node (Groq / Gemini / local chain) |
| `hub_chat` | Gather system context via tools → pass to node's LLM |
| `hub_broadcast` | Send a tool call to ALL connected nodes simultaneously |
| `hub_morph` | Push a new tool module to a node at runtime |
| `hub_restart_node` / `hub_update_node` | Remote agent lifecycle |

---

## Windows → Proxmox: The Stdio Proxy Bridge

AI clients (Kiro, Claude Desktop) run on Windows and speak MCP over stdio. The Proxmox services expose MCP over SSE/HTTP. Four lightweight Python proxies bridge the gap:

```
Windows workstation
│
├── stdio_proxy.py      ──HTTP/SSE──► ProxmoxMCP LXC 160 :8000/sse
│     Handles full MCP handshake on behalf of legacy SSE server.
│     Intercepts server/discover (→ -32601), caches capabilities,
│     routes all tool calls bidirectionally.
│
├── hermes_mcp_proxy.py ──SSH──► Proxmox host ──pct exec 180──► hermes mcp serve
│     Bidirectional stdin/stdout relay via 3 threads.
│
├── n8n_mcp_proxy.py    ──SSH──► LXC 180 :22 ──► n8n MCP server
│     Injects N8N_API_URL (192.168.1.141:5678) at connection time.
│
└── mem0_mcp_proxy.py   ──HTTP──► Mem0 API LXC 141 :8888
      Pure Python stdio MCP server — no SSH needed.
      Implements full MCP protocol natively.
```

The `stdio_proxy.py` protocol-level detail worth noting: when an MCP client sends `server/discover` (a newer spec addition the remote SSE server doesn't implement), the proxy immediately replies with `-32601 Method not found` — so the client falls back to standard `initialize` handshake without any timeout. This kind of edge case is not covered in any tutorial; it was diagnosed and fixed from raw MCP traffic analysis.

---

## LXC 180 — Hermes Agent

```
LXC 180 — hermes-agent (192.168.1.80)
2 cores / 4 GB RAM / 8 GB ZFS
│
├── Hermes messaging bridge (port 9119 — web UI)
│     Connects to Telegram, Discord, Slack, Signal, WhatsApp, Matrix
│     Exposes MCP tool server via: hermes mcp serve
│
└── N8N MCP Server
      /root/.hermes/mcp-servers/n8n/
      Connects to N8N at 192.168.1.141:5678
      Allows AI agents to create/manage/trigger N8N workflows
```

Hermes is the notification backbone. When any automation produces a result — scraper finds a new listing, infra health check fails, a workflow completes — the agent routes the message through Hermes to Telegram (or any other connected platform).

---

## Scraper / Bot Infrastructure

### VM Architecture

```
VM 200 — Scraper-Donor-Standard-200  (stopped — golden template)
         2 cores / 2 GB RAM / 40 GB
         Contains: Chrome + undetected-chromedriver + worker agent
         Used as clone source for new scraper workers
                    │
                    │  clone_resource()
                    ▼
VM 201 — scraper-worker-01  (running)
         2 cores / 2 GB RAM / 40 GB (cloned from VM 200)
         uptime: active | tags: scraper;worker
```

The donor VM pattern: one configured VM is kept as a "golden template". New workers are cloned from it via the `clone_resource` MCP tool — no manual setup.

### Bot Execution Flow

```
Schedule trigger (N8N cron — every N minutes)
        │
        ▼
N8N workflow (LXC 141)
        │
        ├── Simple site: direct HTTP request → parse → compare → notify
        │
        └── Protected site (bot detection):
                │
                ▼
        POST http://192.168.1.51:8080/tasks
        {"command": "run_tool", "tool": "listing_scraper", "params": {...}}
                │
                ▼  RabbitMQ queue (LXC 141)
        VM 201 — scraper-worker-01
                │
                ├── Connects to Chrome via DevTools Protocol (port 9222)
                │     Non-headless, existing Chrome profile
                │     navigator.webdriver = false
                ├── undetected-chromedriver fingerprint masking
                ├── Human-like timing (randomized sleep)
                └── Extracts structured data
                        │
                        ▼  RabbitMQ master_results queue
        N8N listens for result
                │
                ▼
        Hermes (LXC 180) → Telegram notification → Your phone
```

**Why connect to an existing Chrome session?**
Some sites attempt to detect and block automated access. Connecting to an existing Chrome instance via Chrome DevTools Protocol (the same mechanism Playwright and Puppeteer use internally) makes the session indistinguishable from normal browser activity.

### AI Agent Management of the Scraper Fleet

```
User: "Deploy a new monitoring bot for site X, 30-minute interval, Telegram alerts"

Agent calls:
  1. clone_resource(200, new_vmid=202)         → new worker VM from donor template
  2. manage_power(202, "start")                → boot it
  3. [N8N MCP] create workflow with cron       → schedule setup
  4. [N8N MCP] add RabbitMQ + Hermes nodes    → result routing
  5. deploy_service(202, "bot-sitex", ...)     → DNS + NPM proxy for monitoring UI

Done. Entire fleet lifecycle through conversation.
```

---

## Full System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Windows Workstation                                  │
│                                                                             │
│   Kiro IDE / Claude Desktop                                                 │
│   ┌─────────────────────────────────────────────────────┐                  │
│   │  MCP Client                                         │                  │
│   │  stdio_proxy.py      hermes_mcp_proxy.py            │                  │
│   │  n8n_mcp_proxy.py    mem0_mcp_proxy.py              │                  │
│   └──────┬───────────────────┬───────────────┬──────────┘                  │
└──────────┼───────────────────┼───────────────┼──────────────────────────── ┘
           │ HTTP/SSE          │ SSH tunnel    │ HTTP
           ▼                   ▼               ▼
┌──────────────────────────────────────────────────────────────────────────── ┐
│                    Proxmox VE Host — 192.168.1.123                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LXC 160 — agent-proxmox (ProxmoxMCP)  192.168.1.60                │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │  FastMCP + FastAPI                                           │  │   │
│  │  │  ├── proxmox_tools    → Proxmox API (192.168.1.123:8006)    │  │   │
│  │  │  ├── adguard_tools    → AdGuard API  (192.168.1.28:3000)    │  │   │
│  │  │  ├── npm_tools        → NPM API      (192.168.1.27:81)      │  │   │
│  │  │  ├── gitea_tools      → Gitea API    (192.168.1.25)         │  │   │
│  │  │  ├── svn_tools        → SVN HTTP     (192.168.1.26:13081)   │  │   │
│  │  │  ├── mem0_tools       → Mem0 API     (192.168.1.141:8888)   │  │   │
│  │  │  └── orchestration    → all of the above, composed          │  │   │
│  │  └──────────────────────────────────────────────────────────── ┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ LXC 100  Gitea   │  │ LXC 110 Shd-SVN  │  │ LXC 130  AdGuard Home   │  │
│  │ 192.168.1.25     │  │ 192.168.1.26     │  │ 192.168.1.28            │  │
│  │ Git server       │  │ SVN + sync       │  │ DNS + filtering         │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────────────────────────────────────┐ │
│  │ LXC 120  NPM     │  │ LXC 141  docker-host-default  192.168.1.141     │ │
│  │ 192.168.1.27     │  │ ┌──────────┐ ┌────────────┐ ┌────────────────┐  │ │
│  │ Reverse proxy    │  │ │  N8N     │ │ RabbitMQ   │ │ Mem0           │  │ │
│  │ SSL termination  │  │ │  :5678   │ │ :5672      │ │ :8888          │  │ │
│  └──────────────────┘  │ │ Workflow │ │ Msg broker │ │ pgvector       │  │ │
│                         │ └──────────┘ └────────────┘ └────────────────┘  │ │
│                         └──────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────┐ │
│  │ LXC 180  hermes-agent            │  │ VM 200  Scraper-Donor (stopped)  │ │
│  │ 192.168.1.80                     │  │ Chrome + worker agent            │ │
│  │ Hermes :9119                     │  │ Golden template for cloning      │ │
│  │ Telegram/Discord/Slack/Signal    │  └──────────────────────────────────┘ │
│  │ N8N MCP server                   │                                      │
│  └──────────────────────────────────┘  ┌──────────────────────────────────┐ │
│                                         │ VM 201  scraper-worker-01 (run) │ │
│                                         │ Cloned from VM 200              │ │
│                                         │ Chrome :9222 debug port         │ │
│                                         └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Deployment Pipeline

```bash
# First-time: create LXC 160 + install Docker + start container
python MasterExecutor/autonomous_deploy.py

# Redeploy (teardown + fresh start)
python MasterExecutor/autonomous_deploy.py --recreate

# Incremental update (push new code to running container via SSH)
python MasterExecutor/ssh_deploy.py
```

`autonomous_deploy.py` uses the Proxmox API directly (proxmoxer) — no clicking in the web UI. It provisions the LXC, installs Docker inside via `pct exec`, copies the project, and starts the container.

---

## Live State at Time of Writing

Pulled directly from Proxmox API:

| VMID | Name | Status | RAM used | Uptime |
|------|------|--------|----------|--------|
| 100 | Gitea-Server | running | 160 MB / 1 GB | ✓ |
| 110 | Shadow-SVN | running | 130 MB / 1 GB | ✓ |
| 120 | Nginx-Proxy-Manager | running | — | ✓ |
| 130 | AdGuard-Home | running | — | ✓ |
| 141 | docker-host-default | running | 951 MB / 4 GB | ✓ |
| 160 | agent-proxmox | running | 184 MB / 512 MB | ✓ |
| 180 | hermes-agent | running | 1.24 GB / 4 GB | ✓ |
| 200 | Scraper-Donor | stopped | — | — |
| 201 | scraper-worker-01 | **running** | 1.69 GB / 2 GB | ✓ |
| 210 | Android-Malware-Analyzer | stopped | — | — |

Node total: **8.8 GB / 30.3 GB RAM** at idle. ZFS pool: 85.6 GB used of 179.5 GB.

---

## Technology Stack

| Category | Technologies |
|----------|-------------|
| Hypervisor | Proxmox VE, LXC (unprivileged containers), KVM (QEMU VMs) |
| Storage | ZFS (local-zfs), vzdump backups |
| Container runtime | Docker, Docker Compose (inside LXC with nesting) |
| Reverse proxy | Nginx Proxy Manager |
| DNS | AdGuard Home |
| Automation | N8N (Docker, LXC 141) |
| Message broker | RabbitMQ (Docker, LXC 141) |
| Version control | Gitea (Git), Shadow-SVN (SVN with custom sync engine) |
| Vector memory | Mem0, PostgreSQL 17 + pgvector, Gemini embeddings |
| Messaging | Hermes (Telegram / Discord / Slack / Signal / Matrix bridge) |
| AI protocol | MCP (Model Context Protocol) — SSE server + 4 stdio proxies |
| Control plane | FastMCP + FastAPI (Python, ~46-line bootstrap) |
| Scraping | undetected-chromedriver, Chrome DevTools Protocol (:9222) |
| AI clients | Kiro IDE, Claude Desktop |

---

## What This Demonstrates

**End-to-end ownership.** Every layer is self-designed and self-operated — from Proxmox node configuration and ZFS storage layout to the application protocol proxy running on Windows.

**Infrastructure as code mindset.** No manual provisioning. Every operation is a reproducible API call or script. The entire service catalog can be rebuilt from `autonomous_deploy.py`.

**AI-native control plane.** Built the MCP server before MCP became mainstream tooling — treating AI agents as first-class infrastructure clients rather than an afterthought. The result: I manage my entire lab conversationally, including fleet operations on scraper VMs.

**Protocol-level engineering.** The stdio proxy handles MCP handshake edge cases (`server/discover` → `-32601`, capability caching, SSE bridging) not covered in any documentation. These were diagnosed from raw JSON-RPC traffic.

**Practical automation.** The monitoring bot is a real problem solved with real tooling. The Chrome debug port approach and donor VM cloning pattern both solve concrete, production-grade engineering problems.

**Security architecture.** Credential management separated from code — `SecretManager` with ENV-primary / SQLite-fallback, runtime-updatable without restarts.

---

## Related Projects

- **[Shadow-SVN](https://github.com/cihadkocasahan/shadow-svn)** — The SVN sync engine running in LXC 110, also open-sourced.
- **[Profile-PDF-Automator](https://github.com/cihadkocasahan/Profile-PDF-Automator)** — GitHub Actions CI/CD pipeline that auto-generates PDF resumes from Markdown.

---

*Cihad Kocaşahan — [github.com/cihadkocasahan](https://github.com/cihadkocasahan) — [linkedin.com/in/cihadkocasahan](https://linkedin.com/in/cihadkocasahan)*
