# 🚦 Traefik Edge Proxy — Production-Grade Infrastructure

> A modular, environment-aware reverse proxy setup using **Traefik v3.6**, designed to act as the single entry point for a multi-service Docker ecosystem. Features automatic SSL certificate management, real-time service discovery, and a hardened monitoring dashboard.

<p align="center">
  <img src="https://img.shields.io/badge/Traefik-v3.6-00ADD8?style=for-the-badge&logo=traefikproxy&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/Let's%20Encrypt-Auto--SSL-003A70?style=for-the-badge&logo=letsencrypt&logoColor=white" />
  <img src="https://img.shields.io/badge/Azure-Ready-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white" />
  <img src="https://img.shields.io/badge/Security-BasicAuth%20%2B%20TLS-critical?style=for-the-badge&logo=shield&logoColor=white" />
</p>

---

## 💡 What This Demonstrates

This repository is part of my infrastructure portfolio. It showcases my ability to design and operate **production-ready, cloud-deployed infrastructure** using modern DevOps tooling:

- **Modular Docker Compose architecture** — separation of base, development, and production concerns
- **Automated SSL/TLS lifecycle** — zero-touch certificate provisioning and renewal via ACME
- **Dynamic service discovery** — label-driven routing without proxy restarts
- **Security-first design** — read-only socket mounts, bcrypt-hashed auth, and strict file permissions
- **Cloud deployment** — battle-tested on Azure virtual machines

---

## 🏗️ Architecture

```
                        Internet
                            │
                            ▼
          ┌─────────────────────────────────────┐
          │       Traefik v3.6 — Edge Proxy      │
          │                                     │
          │  Entrypoint :80  (web)              │
          │  ┌─────────────────────────────┐    │
          │  │  Global HTTP → HTTPS        │    │
          │  │  301 Redirect (all traffic) │───▶│
          │  └─────────────────────────────┘    │
          │                                     │
          │  Entrypoint :443 (websecure)        │
          │  ┌─────────────────────────────┐    │
          │  │  TLS Termination via ACME   │    │
          │  │  Let's Encrypt Auto-Certs   │    │
          │  └─────────────────────────────┘    │
          │                                     │
          │  Docker Provider                    │
          │  ┌─────────────────────────────┐    │
          │  │  docker.sock (read-only)    │    │
          │  │  Live label-based discovery │    │
          │  └─────────────────────────────┘    │
          └─────────────────────────────────────┘
                 │            │            │
                 ▼            ▼            ▼
            Service A    Service B    monitor.your-domain.com
          (auto-routed) (auto-routed) (Dashboard — BasicAuth)
```

Traefik watches the Docker socket in real time. Any container joining the shared network with the correct labels is **automatically registered** as a live route — no proxy restart, no manual config changes.

---

## ⚙️ Configurable Values — Set by the Traefik Administrator

The following values are **not hardcoded** — they are decisions made once by whoever manages this Traefik instance and must be communicated to every developer working with services in the ecosystem.

| Value | Where it appears | Example |
|---|---|---|
| **Network name** | `compose.yaml`, every application's `compose.yaml` | `your-network` |
| **Dashboard subdomain** | `compose.prod.yaml` label `Host()` | `monitor.your-domain.com` |
| **Admin email** | `compose.prod.yaml` ACME resolver | `your-email@your-domain.com` |
| **Dashboard credentials** | `compose.prod.yaml` BasicAuth middleware | Generated with `htpasswd` |
| **Local hostnames** | Each app's `compose.yaml` label + hosts file | `your-app.localhost` |
| **Production subdomains** | Each app's `compose.yaml` label + DNS A record | `your-app.your-domain.com` |

> 🔑 **Rule:** If you are a developer integrating a new service, ask the Traefik admin for the shared network name and the hostname conventions before writing your labels.

---

## ✨ Key Features

| Feature | Implementation |
|---|---|
| 🔒 **Automatic SSL** | Let's Encrypt ACME via TLS Challenge — certificates issued and renewed with zero manual intervention |
| 🔁 **Global HTTP → HTTPS Redirect** | Entrypoint-level redirect — all port 80 traffic is permanently forced to 443, no per-route config needed |
| 🐳 **Zero-Config Service Discovery** | Docker label-based routing — new containers are registered as live routes the moment they start |
| 🛡️ **Hardened Dashboard** | Monitoring UI exposed only over HTTPS and protected by BasicAuth with bcrypt-hashed credentials |
| ♻️ **Persistent Certificate Store** | `acme.json` persisted via Docker volume with `chmod 600` — survives restarts and respects Let's Encrypt rate limits |
| 🧩 **Three-File Modular Architecture** | `compose.yaml` (base) · `compose.override.yaml` (dev) · `compose.prod.yaml` (prod) — each environment is explicit, isolated, and zero-duplication. No environment-specific logic bleeds across files |

---

## 📁 Project Structure

```
.
├── compose.yaml           # Base layer — image, socket, network, restart policy
├── compose.override.yaml  # Local dev — insecure dashboard on :8081, HTTP only
├── compose.prod.yaml      # Production — HTTPS, ACME, BasicAuth, full TLS stack
└── acme.json              # Certificate store — auto-generated, must be chmod 600
```

### Design Rationale

The three-file pattern follows Docker Compose's native merge strategy, keeping each environment **isolated and explicit** without duplicating configuration:

- **`compose.yaml`** is the single source of truth for what never changes: the image version, the read-only `/var/run/docker.sock` mount, and the shared external network declaration (`your-network` — defined by the Traefik admin).
- **`compose.override.yaml`** is picked up automatically by `docker compose up`, activating insecure mode and the local dashboard port. Safe for development, never for production.
- **`compose.prod.yaml`** must be explicitly passed with `-f`. It replaces development defaults with production-grade settings: the `letsencrypt` certificate resolver, dual entrypoints (`:80` + `:443`), and the `auth-dashboard` BasicAuth middleware.

---

## 🗺️ How Local Routing Works — The Hosts File

Traefik routes traffic based on **hostnames**, not IP addresses or ports. In production this works automatically because DNS resolves your domain to the server. Locally, your machine has no idea what `your-app.localhost` means — it needs to be told explicitly.

This is done by mapping the hostname to `127.0.0.1` in your operating system's **hosts file**. Without this entry, the request never reaches Traefik, even if both the proxy and the service container are running perfectly.

The hostname used in the hosts file **must exactly match** the value defined in the Traefik label of that application's `compose.yaml`:

```yaml
- "traefik.http.routers.my-service.rule=Host(`your-app.localhost`)"
```

> 🔑 The Traefik admin decides what hostname convention to use (e.g. `your-app.localhost`). Ask them before configuring your hosts file.

---

### 🖥️ Hosts File Setup by OS

> Do this **once per project** you work with locally. Every service that defines a Traefik `Host()` rule needs a corresponding entry.

#### 🐧 Linux

```bash
sudo nano /etc/hosts
```

Add a line at the bottom:

```
127.0.0.1   your-app.localhost
```

Save with `Ctrl+O`, exit with `Ctrl+X`. Changes take effect immediately — no restart required.

#### 🍎 macOS

```bash
sudo nano /etc/hosts
```

Add the same entry:

```
127.0.0.1   your-app.localhost
```

Save with `Ctrl+O`, exit with `Ctrl+X`. Then flush the DNS cache:

```bash
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder
```

#### 🪟 Windows

1. Open **Notepad** (or any text editor) **as Administrator** — right-click → *Run as administrator*
2. Open the file: `C:\Windows\System32\drivers\etc\hosts`
3. Add a line at the bottom:

```
127.0.0.1   your-app.localhost
```

4. Save the file. Changes apply immediately.

> 💡 On Windows you can also use [Hosts File Editor](https://hostsfileeditor.com/) for a GUI experience.

---

### Verification

After adding the entry, verify it resolves correctly before starting any container:

```bash
# Linux / macOS
ping your-app.localhost

# Windows (PowerShell)
Resolve-DnsName your-app.localhost
```

You should see `127.0.0.1` in the response. If you do, Traefik will be able to route traffic to the right container.

---

## 🛠️ Local Development

> **Requirements:** Docker Engine 24+ · Docker Compose v2+

### First-time setup

```bash
# 1. Clone this repository
git clone https://github.com/your-username/traefik-proxy.git
cd traefik-proxy

# 2. Create the shared external network (only once — persists across reboots)
# Use the network name defined by the Traefik administrator
docker network create your-network
```

Then configure the hosts file for each project you plan to work with locally, as described in the section above.

---

### Startup order — this matters

> 🚦 **Traefik must always be started before your application.** It is the gateway — if it is not running, no request can reach any service, even if the app container itself is up.

Follow this order every time you start a local development session:

**Step 1 — Start Traefik (this repo)**

```bash
# Inside the traefik-proxy directory
# compose.yaml + compose.override.yaml are merged automatically
docker compose up -d
```

Verify it is ready:

```bash
# Should return the Traefik dashboard
open http://localhost:8081
```

**Step 2 — Start your application (its own repo)**

```bash
# Inside your application's directory
docker compose up -d
```

Traefik will detect the new container via its labels and register the route instantly. Open `http://your-app.localhost` (the hostname defined by the Traefik admin) in your browser — it works.

---

## 🚀 Production Deployment

> Tested on **Azure VM** (Ubuntu 22.04 LTS). Applicable to any Linux cloud or bare-metal host.

> 💡 In production there is **no hosts file configuration needed**. DNS is handled by your domain registrar or cloud DNS service (e.g., Azure DNS, Cloudflare). Simply point an A record for each subdomain to your server's public IP and Traefik + Let's Encrypt take care of the rest.

### Pre-flight Checklist

- [ ] Ports **80** and **443** open in your firewall / NSG
- [ ] DNS **A record** for each subdomain pointing to your server's public IP
- [ ] Docker Engine installed on the host (Linux only)

### 1 — Provision the certificate store

Run the following inside your local clone of this repository:

```bash
touch acme.json
chmod 600 acme.json
```

> ⚠️ **Critical — do not skip.** Traefik will silently fail to write or read certificates if `acme.json` has incorrect permissions. The file must be owned by and readable only by the process running the container. `chmod 600` enforces this.

### 2 — Create the shared network

```bash
# This name must match the network declared in compose.yaml and in every application service
docker network create your-network
```

### 3 — Generate hardened credentials for the dashboard

```bash
# Install htpasswd
sudo apt-get install -y apache2-utils

# Generate credentials using Bcrypt (-B flag)
# Bcrypt is the modern hashing standard recommended for Traefik v3 — significantly stronger than the default MD5-based apr1
htpasswd -nB admin your_secure_password
# → admin:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Update `compose.prod.yaml` with the output:

```yaml
- "traefik.http.middlewares.auth-dashboard.basicauth.users=admin:$$2y$$05$$..."
```

> 💡 Every `$` in the bcrypt hash **must be doubled** (`$$`) inside a Docker Compose YAML file. This is a Compose escaping requirement, not a Traefik one.

### 4 — Deploy

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

### 5 — Validate

```bash
# Stream logs and confirm ACME certificate issuance
docker logs traefik --follow
```

> 💡 Look for log lines containing `"Obtained certificate"` or `"Adding certificate"` to confirm Let's Encrypt issued the cert successfully.

Once issued, the dashboard is live at `https://monitor.your-domain.com` — HTTPS-only, protected by BasicAuth.

---

## 🌐 Connecting an Application to Traefik

> ⚠️ **This repository does not need to be modified to add new services.**

Traefik uses **Docker labels** to discover and route services automatically. The labels live inside each application's **own repository** — not here. This Traefik repo is deployed once and never touched again when a new app is added.

The separation of responsibilities looks like this:

```
traefik-proxy/          ← this repo — deploy once, leave it running
└── compose.yaml

my-web-app/             ← your application's own repo
└── compose.yaml        ← labels go HERE, not in the Traefik repo
```

In your application's `compose.yaml`, join the shared network and declare the Traefik labels. The **network name**, **hostname**, and **subdomain** must be provided by the Traefik administrator:

```yaml
# my-web-app/compose.yaml
services:
  my-service:
    image: my-service:latest
    networks:
      - proxy_network                   # must use the network name defined by the Traefik admin
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=your-network"                          # network name — ask the Traefik admin
      - "traefik.http.routers.my-service.rule=Host(`your-app.your-domain.com`)"
      - "traefik.http.routers.my-service.entrypoints=websecure"
      - "traefik.http.routers.my-service.tls.certresolver=letsencrypt"
      - "traefik.http.services.my-service.loadbalancer.server.port=3000"

networks:
  proxy_network:
    name: your-network            # must match the network name in the Traefik repo
    external: true
```

When you run `docker compose up -d` in your app's repo, Traefik reads the labels via the Docker socket and **registers the route instantly** — no restarts, no changes to this repo, no manual config.

---

## 🔐 Security Design

| Layer | Mechanism |
|---|---|
| **Transport** | TLS enforced on all `websecure` routes — no plain HTTP passthrough at any point |
| **Certificates** | ACME TLS Challenge — certificate authority never contacts an open port on your server |
| **Dashboard Auth** | `BasicAuth` middleware (`auth-dashboard`) with **bcrypt**-hashed credentials (`-B` flag) |
| **Docker Socket** | Mounted **read-only** (`:ro`) — Traefik can observe containers but never modify the host |
| **ACME Store** | `acme.json` with `chmod 600` — readable only by the owning process, never world-readable |

---

## 📈 Future-Proofing & Scalability

This setup is deliberately designed as a foundation that can grow without rearchitecting. The following integrations slot in with minimal configuration changes:

| Capability | Integration Path |
|---|---|
| 📊 **Observability** | Traefik exposes a native Prometheus metrics endpoint (`/metrics`). Connect a Prometheus scraper and visualize with **Grafana** dashboards — latency, request rate, TLS cert expiry, all out of the box |
| 🚦 **Advanced Deployment Strategies** | Weighted load balancing in Traefik labels enables **Canary Releases** and **Blue-Green deployments** without any external tooling — split traffic between service versions at the proxy level |
| 🏗️ **High Availability** | The current single-instance ACME store (`acme.json`) can be replaced with a distributed backend. Traefik v3 supports **Redis** and **Consul** as ACME certificate storage, enabling multi-instance Traefik clusters with shared cert state |
| 🔌 **Additional Providers** | Beyond Docker, Traefik supports Kubernetes (IngressRoute CRDs), Consul Catalog, and file-based dynamic configuration — the same proxy can front both Docker workloads and cloud-native services |

> 💡 None of these require replacing this infrastructure. They are additive layers built on top of the same Traefik instance.

---

## ⚙️ Environment Comparison

| Config        | Local Dev                       | Production                                    |
| ------------- | ------------------------------- | --------------------------------------------- |
| Compose files | `compose.yaml` + `override` | `compose.yaml` + `compose.prod.yaml`      |
| Dashboard     | `localhost:8081` (no auth)    | `monitor.your-domain.com` (BasicAuth + TLS) |
| HTTP          | Port 80, direct                 | Port 80 → 301 redirect to HTTPS              |
| HTTPS         | ❌                              | Port 443, ACME-issued cert                    |
| Auth          | ❌                              | bcrypt BasicAuth                              |

---

## 🧰 Tech Stack

| Tool                                                                         | Role                              |
| ---------------------------------------------------------------------------- | --------------------------------- |
| [Traefik v3.6](https://traefik.io)                                              | Reverse proxy & load balancer     |
| [Docker Compose v2](https://docs.docker.com/compose/)                           | Container orchestration           |
| [Let&#39;s Encrypt](https://letsencrypt.org)                                    | Free, automated TLS certificates  |
| [ACME TLS Challenge](https://letsencrypt.org/docs/challenge-types/)             | Certificate provisioning protocol |
| [Apache `htpasswd`](https://httpd.apache.org/docs/2.4/programs/htpasswd.html) | BasicAuth credential generation   |

---

<p align="center">
  <em>Infrastructure as code, built for clarity, scalability, and zero-surprise deployments.</em>
</p>
