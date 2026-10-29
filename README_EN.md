# Home Lab & AI Control Plane

> Personal Proxmox VE infrastructure — every operation is an MCP tool, every management task is a conversation.

**→ [Türkçe versiyon (ana doküman)](README.md)** | **→ [Technical Details](DETAILS_EN.md)**

---

## How It Started

The motivation was simple: content filtering for the kids and network-wide ad blocking for every device in the house, without configuring each one separately. There was an old PC gathering dust in a corner — instead of throwing it away, I decided to put it to use.

I installed AdGuard Home directly on the machine, no Proxmox yet. On the router, I changed the DNS address distributed via DHCP to AdGuard's IP. From that point, every phone, TV, computer, and guest's device was automatically filtered — no settings touched on any of them. This setup running on a few hundred MB of RAM was using almost none of the machine's capacity — and the rest of it sitting idle started to bother me. *"With this much capacity available, why just DNS?"* — that question was where everything began.

I installed Proxmox on the machine. I configured the two SSDs as a ZFS mirror so that if one failed, the other would keep running with no data loss. Then I moved AdGuard into an LXC container; now I could run multiple isolated services on the same machine.

As services multiplied, memorizing IPs and ports became annoying. I set up Nginx Proxy Manager and added DNS rewrite rules in AdGuard. Now typing `adguard.cihad.local` or `proxmox.cihad.local` was enough; DNS resolves it, NPM routes it to the right service. A simple setup, but it gave the infrastructure its first sense of *"an organized home."*

A while later I needed to monitor a website regularly. Instead of checking manually, I set up N8N and wrote a workflow. I quickly hit a wall: the site had bot protection. The solution was to behave like a real browser. I tried connecting to an existing Chrome session via DevTools Protocol — it worked. As the load grew, I needed to queue jobs; I added RabbitMQ in between and turned a separate VM into a worker. This was the old PC's first *"real workload"* test, and it passed without complaint.

I wanted results delivered to Telegram — which wasn't new to me. I'd already used simple Telegram notifications in an earlier project. But as things grew, a question stuck with me: could I send commands back, not just receive notifications? While researching Telegram-based remote control options, I found Hermes. Not just Telegram — a bridge that connects Discord, Signal, Slack, and Matrix all in one place, with custom commands on top. I set it up, tested it; it worked. The messaging app was already in my pocket; why open a separate panel?

One day I needed to reach a VM from the office. The classic route was a VPN or leaving a port open permanently; both felt like unnecessary security risks. Instead, I wired up an ngrok + VNC integration: one Telegram message, and a time-limited connection URL arrived on my phone seconds later. When the session ended, the tunnel terminated cleanly without leaving any open attack surface.

Around the same time, integrating AI models into the infrastructure highlighted another challenge: I was juggling multiple AI providers — Claude, Gemini, Groq, and several free-tier endpoints. Each had different rate limits, quotas, and distinct strengths. Manually switching when one hit its cap while trying to balance costs was becoming unsustainable. I initially wrote my own proxy. It worked for a while, but providers kept shifting their endpoint signatures and auth formats, making maintenance tedious. At that point, I discovered 9router — an open-source, MIT-licensed, OpenAI-compatible local gateway. Supporting 40+ providers with automated fallback and built-in token optimization, it solved the problem cleanly. I deployed it inside an LXC on Proxmox. Now all local tools and Hermes point to a single endpoint, while 9router handles provider routing dynamically.

There was one more friction point: every AI session started from zero. Yesterday's discussion and last week's architectural decisions vanished across sessions. Re-explaining context was both time-consuming and token-wasteful. Deploying Mem0 solved this completely: a self-hosted vector memory engine on PostgreSQL 17 + pgvector using Gemini embeddings for semantic search. Decisions and learned facts are persisted and recalled across sessions. I containerized it inside the `docker-host` LXC alongside 9router and Hermes.

With dozens of services, containers, and a solid AI backbone operational, managing them through separate web dashboards and SSH terminals became the final bottleneck. Asking *"Can I unify this entire infrastructure into a single standard interface that AI agents can directly converse with and operate?"* led to **ProxmoxMCP**. Built with FastMCP and FastAPI, this control plane turned 60+ infrastructure operations (LXC/VM lifecycle, Docker workloads, AdGuard DNS rewrites, NPM proxy rules, and atomic 3-step service deployments) into standardized AI tools. Now, whether from desktop IDE agents (Antigravity, Kiro, Claude) or conversational Telegram messages, managing the entire homelab requires nothing more than natural language.

Each solution created the next need. What I have now is infrastructure designed from scratch, managed with a single Telegram message or a conversation with an AI agent. Nothing was built speculatively. And that old PC is no longer just blocking ads — it's running as the brain of the house.

---

## Infrastructure Overview

```mermaid
graph TB
    subgraph OUTSIDE["External Access"]
        ME["Phone / Laptop"]
        TG["Telegram"]
        NGROK["ngrok tunnel\nVNC access"]
    end

    subgraph WS["Windows Workstation"]
        AI["Kiro IDE · Claude · Antigravity\nAny MCP-compatible client"]
    end

    subgraph HOME["Home Network — 192.168.1.0/24"]
        ROUTER["Router · DHCP · NAT"]
        AG["AdGuard Home\nDNS + Filtering"]
        NPM["Nginx Proxy Manager\nReverse Proxy + SSL"]

        subgraph PVE["Proxmox VE · 192.168.1.123 · 30GB RAM · ZFS"]
            MCP["LXC 160\nProxmoxMCP\nControl Plane"]
            HRM["LXC 180\nHermes\nMessaging Bridge"]
            DOCKER["LXC 141\nN8N · RabbitMQ · Mem0"]
            GIT["LXC 100 · Gitea"]
            SVN2["LXC 110 · Shadow-SVN"]
            VM201["VM 201\nScraper Worker\nChrome CDP"]
            DONORS["VM 200 / 210\nDonor Templates\nstopped"]
        end
    end

    ME -->|bot command| TG
    TG <-->|bridge| HRM
    HRM -->|MCP tools| MCP
    HRM --> NGROK -.->|VNC| VM201

    AI -->|stdio proxy → SSE| MCP
    MCP -->|proxmoxer API| PVE
    MCP --> AG & NPM & DOCKER & GIT & SVN2

    ROUTER --> AG -->|*.cihad.local| NPM --> PVE
    DONORS -.->|clone| VM201
    DOCKER -->|notification| HRM
```

---

## What I Can Do

### Full Control From Anywhere

From Telegram or a browser (`hermes.cihad.local`), regardless of whether I'm at home:

- Start and stop any VM or LXC
- Deploy a new service (LXC + DNS + proxy in a single step)
- Run an infrastructure health check
- Trigger an N8N workflow
- **Open a temporary ngrok VNC tunnel to any VM**

Kiro IDE, Claude Desktop, Antigravity, or any other MCP-compatible AI client connects to the same tools. Managing infrastructure conversationally is my normal workflow.

### Instant Isolated Environments

I keep pre-configured donor VMs for different OS types (Ubuntu, Windows, Android/BlissOS). One Telegram command via Hermes:

1. Donor VM is cloned
2. Started
3. ngrok tunnel opens, VNC address arrives on Telegram
4. When done, deleted — no trace left

Website testing, app trials, Docker stack experiments — each in an isolated, temporary, clean environment.

### Personal Git Server and AI-Assisted Code Changes

I keep local projects on Gitea (`git.cihad.local`). From my phone via Hermes, I can instruct an AI agent to open a branch, apply changes, run the pre-commit guard, and commit — then check the result in a cloned test VM. No desk required.

This is powered by **DelphiMCP** for Delphi projects — a purpose-built MCP server that gives AI agents actual semantic understanding of Delphi code: 5 specialized servers, 108 tools, blast radius analysis before any change, automatic UTF-8 BOM enforcement, MSBuild integration, and a Telegram approval gate before anything gets committed. The same pattern applies to any stack.

→ Details: **[DETAILS_EN.md — DelphiMCP section](DETAILS_EN.md#delphimcp-a-custom-solution-for-ai-editor-blind-spots)**

### Smart Notifications

I prefer the system alerting me over manually checking things. Active scenarios:
- **Morning infra health report** — automatic every morning; if a service is down, I know with my first coffee
- **Monitoring bot results** — instant notification when a tracked page changes
- **Operation completion** — kick off clone/backup/deploy, walk away, get a message when done
- **Third-party update tracking** — I install applications manually into LXCs or Docker containers on Proxmox and have no way of knowing when a new version is released. I tell Hermes: "check these services, notify me if there are updates, and set up a morning cron" — the agent runs `exec_in_container` to check package state, creates a scheduled N8N workflow, and a Telegram message arrives whenever an update is available
- **Any external API** — any service integration wired to N8N + Hermes takes a few minutes

### Personal Git Server

I keep local projects on Gitea. Accessible at `git.cihad.local` on the local network, or through a Hermes tunnel from outside. From my phone, I can instruct an AI agent via Hermes to make changes to a repo, then check the result in a cloned test VM — no need to be at a desk.

### Data Security and Backups

Data lives on a ZFS pool — integrity is protected at the filesystem level. Proxmox's built-in backup mechanism (`vzdump`) takes automatic snapshots of critical LXCs; backups are stored in a separate storage area (`MigrationBackup`). The snapshot system allows instant rollback before any risky change.

Credential management is fully separated from application code — a central secrets file resolved through a `SecretManager` layer with an encrypted SQLite fallback. Sensitive values are never in source code or plain-text environment variables. For teams, this pattern is straightforwardly replaceable with any open-source vault solution (HashiCorp Vault, Infisical, etc.) without changing the application layer.

---

## Service Map

| ID | Service | IP | Resources | Status |
|----|---------|-----|-----------|--------|
| LXC 100 | Gitea | 192.168.1.25 | 1c/1GB/30GB | ✅ running |
| LXC 110 | Shadow-SVN | 192.168.1.26 | 1c/1GB/20GB | ✅ running |
| LXC 120 | Nginx Proxy Manager | 192.168.1.27 | 1c/1GB/10GB | ✅ running |
| LXC 130 | AdGuard Home | 192.168.1.28 | 1c/4GB/5GB | ✅ running |
| LXC 141 | Docker Host (N8N + RabbitMQ + Mem0) | 192.168.1.141 | 4c/4GB/20GB | ✅ running |
| LXC 1xx | 9router — AI Gateway | — | — | LXC or Docker |
| LXC 160 | ProxmoxMCP — Control Plane | 192.168.1.60 | 4c/512MB/8GB | ✅ running |
| LXC 180 | Hermes — Messaging Bridge | 192.168.1.80 | 2c/4GB/8GB | ✅ running |
| VM 200 | Scraper-Donor (Ubuntu+Chrome) | — | 2c/2GB/40GB | ⏸ stopped |
| VM 201 | scraper-worker-01 | — | 2c/2GB/40GB | ✅ running |
| VM 210 | Android-Malware-Analyzer (BlissOS) | — | 2c/4GB/32GB | ⏸ stopped |

*Pulled from live Proxmox API. Total RAM usage: ~8.8 GB / 30.3 GB at idle.*

---

## Technology Stack

| Category | Technologies |
|----------|-------------|
| Hypervisor | Proxmox VE, LXC (unprivileged), KVM/QEMU |
| Storage | ZFS (local-zfs), vzdump automated backups |
| Container runtime | Docker, Docker Compose (LXC with nesting) |
| Reverse proxy | Nginx Proxy Manager |
| DNS & Filtering | AdGuard Home |
| Automation | N8N |
| Message broker | RabbitMQ |
| Version control | Gitea (Git), Shadow-SVN |
| Vector memory | Mem0, PostgreSQL 17 + pgvector |
| Messaging bridge | Hermes (Telegram / Discord / Signal / Slack / Matrix) |
| AI gateway | 9router (40+ providers, auto-fallback, token optimization) |
| Session memory | Mem0, PostgreSQL 17 + pgvector, Gemini embeddings |
| AI protocol | MCP (Model Context Protocol) — SSE + stdio proxy |
| Control plane | FastMCP + FastAPI (Python, ~46-line bootstrap) |
| Remote access | ngrok (TCP tunnel, ephemeral), VNC |
| AI clients | Kiro IDE, Claude Desktop, Antigravity — any MCP-compatible client |
| Credential management | SecretManager (ENV-first, encrypted SQLite fallback) |

---

## What Kind of Developer I Am

20 years of IT and electronics experience, professional software development since 2021, specializing in Delphi enterprise healthcare systems. This home lab shows something the CV can't: when I encounter a problem, I solve it with tools — and if the tools don't exist, I build them.

- **Systems thinking:** Every layer was designed with intent. The reasoning behind each decision is traceable.
- **Protocol-level engineering:** The MCP stdio proxy handles edge cases not covered in any documentation — diagnosed from raw JSON-RPC traffic.
- **How I work with AI:** I build tools that AI connects to as a client. Kiro, Claude, and Antigravity manage my infrastructure conversationally through the MCP server I wrote. This is my default workflow, not a demo.
- **Automation reflex:** Every component in this stack was built to replace something I was doing manually. The reflex is: see the repetition, automate it.

---

## What's Next? (Future Vision)

The current implementation has matured the environment into a **"conversational, command-driven"** control plane. The next phase aims to equip the infrastructure with **"autonomous reflexes"** that operate without requiring manual intervention:

- 🛡️ **Self-Healing Infrastructure:** In the event of service degradations or memory leaks, AI agents automatically inspect logs (`docker_logs`, `exec_in_container`), diagnose the root cause, restart the degraded service, and deliver a concise resolution report to Telegram.
- 🔄 **Local GitOps Pipeline (Gitea Webhooks → Auto-Deploy):** Triggering zero-touch builds, tests, and container deployments directly on Proxmox upon every `git push` to Gitea, eliminating manual dashboard interaction.
- 🧠 **Engineering Memory & Codebase RAG:** Semantically indexing past Delphi, Python, and infrastructure codebases into Mem0 and pgvector, enabling instant recall of past architectural patterns and implementations during active development.
- 🔐 **Interactive 2FA Approvals for Critical Operations:** Requiring one-tap Telegram inline button verification (`[Approve / Reject]`) before executing destructive or high-impact MCP tool commands (`remove_resource`, backup pruning, etc.).

---

## Related Projects

- **[Shadow-SVN](https://github.com/cihadkocasahan/shadow-svn)** — SVN sync engine running in LXC 110, open-sourced.
- **[Profile-PDF-Automator](https://github.com/cihadkocasahan/Profile-PDF-Automator)** — GitHub Actions pipeline that auto-generates PDF resumes from Markdown.

## Portability: Continue from Any Machine

The CihadEnv repository lives on Gitea. Moving to a new machine: `git clone` + fill in the `.template` files. The `.agents/` folder carries all AI behavior rules, workflow definitions, and standards — Kiro or Claude immediately continues with the same mindset. The Proxmox server stays separate; services keep running without interruption. Cross-session memory is stored in Mem0 — context doesn't disappear when the PC changes.

All credentials live in one place: `Master-Credentials.env`. ProxmoxMCP, Docker containers, LXC services, and Windows proxies all read from the same source. Never enters the repo — carried via `.template`.

→ Full technical details, MCP tool catalog, diagrams, and use case walkthroughs: **[DETAILS_EN.md](DETAILS_EN.md)**

---

*Cihad Kocaşahan — [github.com/cihadkocasahan](https://github.com/cihadkocasahan) — [linkedin.com/in/cihadkocasahan](https://linkedin.com/in/cihadkocasahan)*
