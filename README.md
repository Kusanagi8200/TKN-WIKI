#### THE KUZLAB - INFRASTRUCTURE (IN REVIEW)

[![TKN-SYNOPTIC-2026.jpg](https://wiki.kuzlab.org/uploads/images/gallery/2026-04/scaled-1680-/21atkn-synoptic-2026.jpg)](https://wiki.kuzlab.org/uploads/images/gallery/2026-04/21atkn-synoptic-2026.jpg)

---

Last updated -->   2026-04-08

###### This page describes the current infrastructure for The KUZ NETWORK
Servers, domains, public websites, internal tools, and how everything fits together.

---

##### 1) OVERVIEW

- ONE LOCAL AI SERVER (LLM TEST BENCH  -->   NOT PUBLICLY EXPOSED)
  - LOCAL LLM DUO CHAT STACK INSIDE THE LAB -->   SRV3 + SRV4, LOCAL ORCHESTRATION, DUAL LLAMA.CPP ENGINES, LOCAL WEB-UI MANAGEMENT, PROMPT / TURNS / TRANSCRIPT WORKFLOW
---
- TWO DEBIAN VPS
  - VPS 1 - INFOMANIAK (83.228.192.116) -->   DOCKER HOST, INTERNAL TOOLING, DOCUMENTATION, PROJECT MANAGEMENT
  - VPS 2 - LWS (91.234.194.49) -->   APACHE2 FRONT FOR PUBLIC WEBSITES
---
DOMAINS
- KUZAI.ORG → PUBLIC PORTAL + PRODUCT SUBDOMAINS
- KUZLAB.ORG → INTERNAL TOOLING / INFRASTRUCTURE

---

##### 2) COMPONENTS & PURPOSES

##### 2.1 LOCAL SERVER - AI SERVER (LLM LAB)
- PURPOSE -->   MODEL LOADING, BENCHMARKING, RAG/AGENTS POCS, WITHOUT TOUCHING PRODUCTION
- EXPOSURE -->   LOCAL / LAB ONLY (NOT PUBLIC)
- LOCAL LLM DUO CHAT -->   DUAL LOCAL INFERENCE / CHAT STACK WITH SRV3 AND SRV4, DESIGNED FOR MULTI-MODEL CONVERSATION FLOWS, SYSTEM PROMPT MANAGEMENT, TURN STORAGE, TRANSCRIPT GENERATION, AND LOCAL FINE-TUNING PREPARATION
- SRV3 - LOCAL / LLM INFRA 1 -->   APACHE SRV + PROXY, VIRTUAL HOST, ORCHESTRATOR PYTHON, LLAMA.CPP ENGINE, KUZAI LLM MODEL
- SRV4 - LOCAL / LLM INFRA 2 -->   LLAMA.CPP ENGINE, DARKAI LLM MODEL
- DUO CHAT MANAGEMENT LAYER -->   WEB-UI MANAGEMENT, MODEL SYSTEM PROMPT, TURNS FINE TUNING, TURNS TRANSCRIPT
- LOCAL FLOW -->   WEB-UI → APACHE / VHOST / PROXY (SRV3) → PYTHON ORCHESTRATOR → LLAMA.CPP ENGINES (SRV3 / SRV4) → LOCAL MODELS / TRANSCRIPTS / TURN DATA

##### 2.2 VPS INFOMANIAK - 83.228.192.116 (DOCKER + TOOLS)
- HOST OS / ROLE -->   DEBIAN; DOCKER ENGINE FOR INTERNAL SERVICES + MONITORING

##### Services
- PORTAINER - <https://docker.kuzlab.org>  
  _Docker administration (stacks, images, logs, deployments)._  

- N8N - <https://n8n.kuzai.org>  
  _Workflow engine for AI news harvesting & GitHub search that feeds KUZAI.ORG._  

- UPTIME-KUMA - <https://uptime-kuma.kuzlab.org>  
  _Availability monitoring (HTTP/HTTPS), alerts, uptime history._  

- WIKI (BOOKSTACK) - <https://wiki.kuzlab.org>  
  _Central documentation (procedures, access, architecture, checklists)._

- AZURACAST / KUZFM RADIO - <https://radio.kuzai.org/public/kuzfm>  
  _Radio backend / public stream service for KUzFM,._

##### 2.3 VPS LWS - 91.234.194.49 (APACHE2 WEB FRONT)
- HOST OS / ROLE -->   DEBIAN; APACHE2 SERVING PUBLIC WEBSITES

##### Public websites
- KUZAI.ORG (DESKTOP) - <https://kuzai.org>  
  _Universal AI Index - desktop experience._

- MOB.KUZAI.ORG (MOBILE) - <https://mob.kuzai.org>  
  _Universal AI Index - mobile-optimized._

- FIRST.KUZLAB.ORG (CNRS CLONE) - <https://first.kuzlab.org>  
  _Reference clone of the CNRS “First” website._

- KANBOARD - <https://kan.kuzai.org -->  9443>  
  _Project management (kanban, backlog, roadmaps) served via Apache2 on this VPS._

##### 2.4 SOCIAL & VERSIONING
- GITHUB (GENERAL) - <https://github.com/kusanagi8200>  
  _General versioning for The KUZ NETWORK._

- GITHUB (N8N) - <https://github.com/KuzaiN8N>  
  _n8n workflows & automations._

---

##### 3) HOW IT FITS TOGETHER (FLOW)

1. CODE & CONTENT ARE VERSIONED ON GITHUB (GENERAL + N8N).
2. N8N (INFOMANIAK) HARVESTS/CURATES DATA (E.G., AI NEWS) → FEEDS KUZAI.ORG.
3. PUBLIC SITES ARE SERVED BY APACHE2 ON THE LWS VPS.
4. UPTIME-KUMA (INFOMANIAK) MONITORS ALL EXPOSED FQDNS (ALERTS/HISTORY).
5. KANBOARD DRIVES PROJECT EXECUTION; WIKI IS THE DOCUMENTATION SOURCE OF TRUTH.
6. AI SERVER REMAINS THE SAFE R&D PLAYGROUND FOR LLMS.
7. RADIO.KUZAI.ORG / KUZFM EXPOSES THE PUBLIC RADIO ENTRY POINT, BACKED BY THE AZURACAST STACK VISIBLE ON THE INFOMANIAK SIDE OF THE SYNOPTIC.
8. LOCAL LLM DUO CHAT RUNS LOCALLY THROUGH SRV3 + SRV4, WITH PYTHON ORCHESTRATION, DUAL LLAMA.CPP ENGINES, MODEL-SPECIFIC PROMPTS, TURN STORAGE, AND TRANSCRIPT / FINE-TUNING WORKFLOWS.

---

##### 4) SERVER TREES

##### INFOMANIAK - 83.228.192.116
```text
Infomaniak (Debian)
├─ Apache2 (reverse/virtual host)
|
└─ Docker Engine
    ├─ Portainer                 → docker.kuzlab.org
    ├─ n8n                       → n8n.kuzai.org
    ├─ Uptime-Kuma               → uptime-kuma.kuzlab.org/dashboard
    ├─ BookStack (Wiki)          → wiki.kuzlab.org/login
    └─ AzuraCast (Radio)         → radio.kuzai.org/public/kuzfm
```

##### LWS - 91.234.194.49
```text
LWS (Debian)
└─ Apache2 (public front)
    ├─ kuzai.org                 → Universal AI Index (Desktop)
    ├─ mob.kuzai.org             → Universal AI Index (Mobile)
    ├─ first.kuzlab.org          → “First” CNRS clone
    └─ kan.kuzai.org -->  9443        → Kanboard (project management)
```

##### AI SERVER (LOCAL)
```text
AI SERVER (Local / Lab)
└─ LLM environments             → model tests, benchmarks, POCs (not publicly exposed)
   └─ Local LLM DUO Chat
      ├─ SRV3 - LOCAL           → LLM INFRA 1 / Apache SRV + Proxy / Virtual Host / Orchestrator Python
      │  ├─ llama.cpp Engine
      │  └─ KUZAI LLM Model
       |
      ├─ SRV4 - LOCAL           → LLM INFRA 2
      │  ├─ llama.cpp Engine
      │  └─ DARKAI LLM Model
      └─ Local management       → Web-UI Management / Model System Prompt / Turns Fine Tuning / Turns Transcript
```

---

##### 5) DNS SUMMARY
```text
kuzai.org -->  
  @ (apex)              → 91.234.194.49
  mob.kuzai.org         → 91.234.194.49
  n8n.kuzai.org         → 83.228.192.116
  kan.kuzai.org         → 83.228.192.116 (port 9443)
  radio.kuzai.org       → 83.228.192.116

kuzlab.org -->  
  docker.kuzlab.org     → 83.228.192.116
  uptime-kuma.kuzlab.org→ 83.228.192.116
  wiki.kuzlab.org       → 83.228.192.116
  first.kuzlab.org      → 91.234.194.49
```

---

##### SOFTWARE INVOLVED IN THE INFRASTRUCTURE

This page lists the software, its purpose, and how components connect to each other.

---

##### SUMMARY (TABLE)

| Software | Host / IP | Endpoints | Purpose | Connected to… |
|---|---|---|---|---|
| APACHE2 (WEB FRONT) | VPS LWS - 91.234.194.49 | `https://kuzai.org`, `https://mob.kuzai.org`, `https://first.kuzlab.org` | Serve public sites (HTTP/HTTPS) | Consumes n8n feeds; deployments from GitHub |
| APACHE2 (KANBOARD VHOST) | VPS INFOMANIAK - 83.228.192.116 | `https://kan.kuzai.org -->  9443` | Expose Kanboard over HTTPS | Serves Kanboard; internal users access |
| DOCKER ENGINE | VPS INFOMANIAK - 83.228.192.116 | - | Run internal containers | Hosts Portainer, n8n, Uptime-Kuma, BookStack |
| PORTAINER | VPS INFOMANIAK - 83.228.192.116 | `https://docker.kuzlab.org` | Docker administration (stacks, images, logs, deployments) | Controls Docker Engine; used by admins |
| n8n (WORKFLOWS) | VPS INFOMANIAK - 83.228.192.116 | `https://n8n.kuzai.org` | Automation -->   harvest/curate, export feeds | Publishes feeds → kuzai.org / mob.kuzai.org |
| UPTIME-KUMA (MONITORING) | VPS INFOMANIAK - 83.228.192.116 | `https://uptime-kuma.kuzlab.org/dashboard` | HTTP/HTTPS monitoring, alerts, history | Probes all public FQDNs |
| BOOKSTACK (WIKI) | VPS INFOMANIAK - 83.228.192.116 | `https://wiki.kuzlab.org/login` | Central documentation | Linked with Kanboard; DB/uploads backups |
| KANBOARD | VPS INFOMANIAK - 83.228.192.116 (via Apache2) | `https://kan.kuzai.org -->  9443` | Project management | Internal users; referenced by the Wiki |
| AZURACAST / KUZFM RADIO | VPS INFOMANIAK - 83.228.192.116 | `https://radio.kuzai.org/public/kuzfm` | Public radio / streaming service | Exposes radio.kuzai.org; backed by AZURACAST-CONT |
| GITHUB (kusanagi8200 / KuzaiN8N) | External | `https://github.com/kusanagi8200`, `https://github.com/KuzaiN8N` | Versioning code & n8n workflows | Deployment source for web front / Docker stacks |
| AI SERVER (LLM LAB) | Local (not exposed) | - | LLM tests (bench, RAG/agent POCs) | Push code/data to GitHub (as needed) |
| LOCAL LLM DUO CHAT | Local (not exposed) | Local Web-UI only | Dual-node local chat / orchestration layer | Uses SRV3, SRV4, Python Orchestrator, llama.cpp, KUZAI LLM Model, DARKAI LLM Model |

---

##### DETAILS BY COMPONENT

##### APACHE2 - WEB FRONT (VPS LWS - 91.234.194.49)
- Purpose -->   serve `kuzai.org`, `mob.kuzai.org`, `first.kuzlab.org` over HTTP/HTTPS.
- Connections -->  
  - Inbound -->   user traffic (Internet), content/data (n8n feeds over HTTP).
  - Outbound -->   web responses; logs to admins.
  - Deploys -->   artifacts/content from GitHub (per your deployment process).

##### APACHE2 - KANBOARD VHOST (VPS INFOMANIAK - 83.228.192.116)
- Purpose -->   publish `kan.kuzai.org -->  9443` over HTTPS (dedicated vhost).
- Connections -->  
  - Inbound -->   internal users.
  - Outbound -->   Kanboard UI/app, optional webhooks (if enabled).

##### DOCKER ENGINE (VPS INFOMANIAK - 83.228.192.116)
- Purpose -->   runtime for Portainer, n8n, Uptime-Kuma, BookStack.
- Connections -->  
  - Managed by Portainer (admin UI).
  - Exposes each service through its FQDN (HTTPS).

##### PORTAINER - `https://docker.kuzlab.org`
- Purpose -->   Docker administration UI (stacks, images, logs).
- Connections -->  
  - To Docker Engine (control plane).
  - Access -->   admins (MFA recommended).

##### n8n - `https://n8n.kuzai.org`
- Purpose -->   workflows (AI news harvesting, search, normalization), exporting feeds.
- Connections -->  
  - Inbound -->   public sources (HTTP).
  - Outbound -->   kuzai.org / mob.kuzai.org consume feeds over HTTP.
  - Backups -->   export workflows + persistent volume.

##### UPTIME-KUMA - `https://uptime-kuma.kuzlab.org/dashboard`
- Purpose -->   monitoring HTTP/HTTPS endpoints, alerts.
- Connections -->  
  - Probes all public FQDNs.
  - Outbound -->   notifications (email/IM per config).

##### BOOKSTACK (WIKI) - `https://wiki.kuzlab.org/login`
- Purpose -->   documentation and source of truth (procedures, access, architecture).
- Connections -->  
  - Cross-links to Kanboard (projects/tasks) and internal resources.
  - Backups -->   DB + uploads + `.env`.

##### KANBOARD - `https://kan.kuzai.org -->  9443`
- Purpose -->   project management (kanban, backlog, roadmaps).
- Connections -->  
  - Internal users; linked with the Wiki.
  - Optional webhooks to n8n (if you automate).

##### RADIO.KUZAI.ORG / AZURACAST - `https://radio.kuzai.org/public/kuzfm`
- Purpose -->   public KUzFM radio page / streaming entry point.
- Connections -->  
  - Public users access the radio page and stream entry point.
  - Backed by the AZURACAST stack shown on the INFOMANIAK side of the synoptic.
  - Can be exposed through the Apache / reverse proxy layer used for public access.

##### GITHUB - `https://github.com/kusanagi8200` / `https://github.com/KuzaiN8N`
- Purpose -->   versioning of site code/scripts and n8n workflows.
- Connections -->  
  - Deployment source for web front (Apache2) and Docker stacks (Portainer/compose).
  - History/PR/CI as needed.

##### AI SERVER - (LOCAL)
- Purpose -->   LLM lab (tests, benchmarks, RAG/agent POCs) without impacting production.
- Connections -->  

##### LOCAL LLM DUO CHAT - (LOCAL)
- Purpose -->   dual-node local chat / orchestration environment for model-to-model and user-to-model conversation workflows without impacting production.
- Architecture -->  
  - SRV3 - LOCAL / LLM INFRA 1
    - Apache SRV + Proxy
    - Virtual Host
    - Orchestrator Python
    - llama.cpp Engine
    - KUZAI LLM Model
  - SRV4 - LOCAL / LLM INFRA 2
    - llama.cpp Engine
    - DARKAI LLM Model
- Conversation / management layer -->  
  - Web-UI Management
  - Model System Prompt
  - Turns Fine Tuning
  - Turns Transcript
- Connections -->  
  - Inbound -->   local-only UI / management actions.
  - Internal path -->   Web-UI → Apache / Virtual Host / Proxy → Python Orchestrator → SRV3/SRV4 llama.cpp engines.
  - Outputs -->   local prompts, turn history, transcripts, fine-tuning preparation data.
  - Exposure -->   no public endpoint.

---

##### MAIN FLOWS (AT A GLANCE)

- Users → `kuzai.org` / `mob.kuzai.org` (Apache2 on LWS).
- n8n → publishes HTTP feeds → kuzai.org/mob.kuzai.org consume those feeds.
- Uptime-Kuma → probes all FQDNs and sends alerts.
- Portainer → admin for the Docker Engine (deploy/upgrade).
- Wiki ↔ Kanboard → documentation & project delivery (cross-linked).
- GitHub → single source of code/workflows (front & containers).
- AI SERVER → LLM experimentation (push to GitHub if needed).
- Radio / KUzFM → public access through `radio.kuzai.org`, backed by the AzuraCast service shown in the synoptic.
- Local LLM DUO Chat → local Web-UI / prompts / turns / transcript workflow routed through SRV3 and SRV4.

---
#### THE KUZ NETWORK - KUSANAGI8200 - @2026
