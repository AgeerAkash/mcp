# MCP + LangChain — 6 MCP Servers, One Docker Container per API

*Cloud Soft Solutions — APEX / NEXUS Lab*

Six MCP servers, each with **its own folder, Dockerfile, requirements and container**, plus a LangChain agent container. Everything is deployed on one EC2 instance with a single script.

```
┌──────────────────────────── EC2 (Ubuntu 24.04 + Docker) ─────────────────────────────┐
│  Docker network: mcp-net                                                             │
│                           ┌──────────────────────┐                                   │
│                     ┌───▶ │ mcp-weather   :8001  │──▶ Open-Meteo        (no key)     │
│                     ├───▶ │ mcp-jira      :8002  │──▶ Jira Cloud        (API token)  │
│  ┌─────────────┐    ├───▶ │ mcp-ec2       :8003  │──▶ AWS EC2           (IAM role)   │
│  │  mcp-agent  │────┼───▶ │ mcp-github    :8004  │──▶ GitHub REST       (optional)   │
│  │  LangChain  │    ├───▶ │ mcp-currency  :8005  │──▶ Frankfurter / ECB (no key)     │
│  └─────────────┘    └───▶ │ mcp-wikipedia :8006  │──▶ Wikipedia         (no key)     │
│                           └──────────────────────┘                                   │
│   Ports published on 127.0.0.1 only — nothing is exposed to the internet             │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. The 6 MCP servers and 22 tools

| # | Server | Port | Tools | Needs |
|---|---|---|---|---|
| 1 | **weather** | 8001 | `get_current_weather`, `get_forecast` | nothing |
| 2 | **jira** | 8002 | `list_projects`, `search_issues`, `get_issue`, `create_issue`, `add_comment` | Jira URL + email + token |
| 3 | **ec2** | 8003 | `list_instances`, `describe_instance`, `get_instance_summary`, `start_instance`\*, `stop_instance`\* | IAM role |
| 4 | **github** 🆕 | 8004 | `get_repo`, `list_repo_issues`, `get_latest_release`, `search_repositories`, `get_user` | optional token |
| 5 | **currency** 🆕 | 8005 | `convert_currency`, `get_exchange_rates`, `list_currencies` | nothing |
| 6 | **wikipedia** 🆕 | 8006 | `search_wikipedia`, `get_wikipedia_summary` (any language: `en`, `te`, `hi`...) | nothing |

\* disabled unless `EC2_ALLOW_WRITE=true`

---

## 2. Project structure (API-wise)

```
mcp-6-servers-docker/
├── docker-setup.sh            # ⭐ ONE script: install Docker → .env → build → run → test
├── mcp.sh                     # ⭐ daily commands
├── docker-compose.yml         # 6 servers + agent
├── .env.example
├── mcp-servers/
│   ├── weather/    ├── Dockerfile  ├── requirements.txt  └── server.py
│   ├── jira/       ├── Dockerfile  ├── requirements.txt  └── server.py
│   ├── ec2/        ├── Dockerfile  ├── requirements.txt  └── server.py   (+ boto3)
│   ├── github/     ├── Dockerfile  ├── requirements.txt  └── server.py
│   ├── currency/   ├── Dockerfile  ├── requirements.txt  └── server.py
│   └── wikipedia/  ├── Dockerfile  ├── requirements.txt  └── server.py
├── agent/
│   ├── Dockerfile  ├── requirements.txt
│   ├── agent.py               # LangChain agent (chat)
│   └── test_servers.py        # smoke test, no LLM needed
└── deploy/
    ├── fix_imds_hop_limit.sh  # lets containers use the EC2 IAM role
    └── iam/                   # IAM policies + create_role.sh
```

**Why one container per API?**
- **Small images.** Only the ec2 image carries boto3; only the agent carries LangChain.
- **Least privilege.** Each container gets *only its own* secrets. The Jira token never enters the GitHub container.
- **Independent lifecycle.** You can rebuild, restart or read logs for one API without touching the others.
- **Portable.** The same layout maps directly to ECS tasks or Kubernetes Deployments later.

---

## 3. Deploy on EC2

### Step 1: IAM role (from CloudShell or your laptop)
```bash
bash deploy/iam/create_role.sh            # add --with-write to allow start/stop
```

### Step 2: Launch the instance
| Setting | Value |
|---|---|
| AMI | Ubuntu Server 24.04 LTS |
| Type | **t3.medium** recommended (7 images to build); t3.small works but builds slowly |
| Storage | 20–30 GB |
| Security group | SSH (22) from **My IP** only |
| IAM instance profile | `mcp-ec2-agent-role` |
| Advanced → Metadata response hop limit | **2** (needed for containers to use the IAM role) |

### Step 3: Copy and run the setup script
```bash
scp -i mykey.pem mcp-6-servers-docker.zip ubuntu@<EC2_IP>:~
ssh -i mykey.pem ubuntu@<EC2_IP>

sudo apt-get update && sudo apt-get install -y unzip
unzip mcp-6-servers-docker.zip && cd mcp-6-servers-docker
bash docker-setup.sh
```
The script installs Docker, asks for your keys (you can press Enter to skip any), builds the 7 images, starts 6 containers, waits for them to be healthy, runs the smoke tests and checks IAM access.

### Step 4: Chat
```bash
./mcp.sh agent
```

---

## 4. Commands

| Command | What it does |
|---|---|
| `./mcp.sh up` / `down` | Start / stop all servers |
| `./mcp.sh status` | Health of all 6 containers |
| `./mcp.sh agent` | Interactive chat |
| `./mcp.sh ask "convert 250 USD to INR"` | One question |
| `./mcp.sh test` | Call one tool on each server (no LLM needed) |
| `./mcp.sh logs github` | Logs for one API |
| `./mcp.sh restart jira` | Reload `.env` for one API |
| `./mcp.sh rebuild currency` | Rebuild one API after a code change |
| `./mcp.sh shell ec2` | Shell inside a container |
| `./mcp.sh clean` | Remove everything, including images |

---

## 5. Try these prompts

**New APIs**
```
Convert 1500 USD to INR.
What was 1 EUR in INR on 2024-01-15?
Show today's USD rates for INR, AED and SGD.
Tell me about the GitHub repo langchain-ai/langchain.
What's the latest release of hashicorp/terraform?
Show 5 open issues in kubernetes/kubernetes.
Find the top Python MCP server repositories on GitHub.
Summarise the Wikipedia article on Kubernetes.
Give me the Telugu Wikipedia summary of హైదరాబాదు.
```

**Multi-server (the agent chains tools)**
```
What's the latest Terraform release? Create a Jira task in DEMO to upgrade to it.
Look up Docker on Wikipedia, then show me its most-starred GitHub repos.
Weather in Dubai tomorrow, and how much is 5000 INR in AED?
List my running EC2 instances and create a Jira task in DEMO to review their sizes.
```

---

## 6. Add your own (7th) MCP server in 5 steps

1. `cp -r mcp-servers/currency mcp-servers/myapi`
2. Edit `server.py`: rename the server, change the port env var (e.g. `MYAPI_PORT`, 8007), and write your `@mcp.tool()` functions with clear docstrings
3. Change `EXPOSE` and the `HEALTHCHECK` port in `mcp-servers/myapi/Dockerfile`
4. Copy a service block in `docker-compose.yml`, add it to the agent's `depends_on`, and set `MYAPI_MCP_URL: http://myapi:8007/mcp`
5. Add `"myapi": ("MYAPI_MCP_URL", "http://127.0.0.1:8007/mcp")` to `_SERVER_URLS` in `agent/agent.py`, then run `./mcp.sh rebuild`

---

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| Container `unhealthy` | `./mcp.sh logs <name>` |
| `No AWS credentials found` | Attach the IAM role **and** set the hop limit to 2 (`deploy/fix_imds_hop_limit.sh <instance-id>`), then `./mcp.sh restart ec2` |
| `GitHub rate limit reached` | Add `GITHUB_TOKEN` to `.env` → `./mcp.sh restart github` |
| `Jira API error 401` | Wrong email or token pair |
| Wikipedia `403` | Set a real contact in `WIKIPEDIA_USER_AGENT` |
| `.env` change not applied | `./mcp.sh restart <name>` recreates the container. Plain `docker restart` does not re-read `.env` |
| Code change not applied | `./mcp.sh rebuild <name>` |
| Build is slow or runs out of memory | Use t3.medium, or add swap: `sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
