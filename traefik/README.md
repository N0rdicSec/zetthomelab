# Traefik + Caddy Reverse Proxy Stack (Zettlab NAS)

This repository contains a **production-grade reverse proxy architecture** designed for **Zettlab NAS environments**, combining:

* **Traefik** as the internal service router (Docker label–driven)
* **Caddy** as the single LAN entrypoint (macvlan, TLS termination)
* **Self-signed wildcard certificates** (`*.lab.home`)
* **No exposed ports on Traefik**
* **Clean separation between edge and internal routing**

This setup mirrors real-world enterprise patterns while remaining NAS-friendly and easy to maintain.

---

## Architecture Overview

```
LAN Client
   |
   v
Caddy (macvlan IP, :80/:443)
   |
   v
Traefik (bridge-only, internal)
   |
   v
Docker services (traefik-net)
```

---

<img width="2897" height="889" alt="image" src="https://github.com/user-attachments/assets/b84d11fb-0b81-4947-bcc8-2b23ef4ce3ae" />

---

### Why this design?

* Zettlab reserves ports **80/443** on the host
* macvlan gives Caddy a **real LAN IP**
* Traefik remains **internal-only infrastructure**
* Dashboard exposure is tightly controlled
* No port collisions, no host coupling

---

## Key Components

### Caddy (Edge / Front Door)

* Runs on **macvlan**
* Owns **HTTP/HTTPS** on the LAN
* Terminates TLS using a **self-signed wildcard cert**
* Proxies:

  * `traefik.lab.home` → Traefik dashboard
  * `*.lab.home` → Traefik HTTPS entrypoint

### Traefik (Internal Router)

* Runs on **bridge network (`traefik-net`) only**
* No published ports
* Routes services using Docker labels
* Hosts the **dashboard/API internally on port 8080**

---

## Directory Structure

```
traefik/
├── traefik/
│   ├── docker-compose.yml
│   ├── dynamic/
│   │   ├── tls.yml
│   │   ├── dashboard.yml
│   │   └── middlewares.yml
│   └── certs/
│       ├── lab.crt
│       └── lab.key
│
└── Caddy/
    ├── docker-compose.yml
    ├── Caddyfile/
    │   ├── Caddyfile
    ├── certs/
    │   ├── lab.crt
    │   └── lab.key
    ├── data/
    └── config/
```

<img width="1315" height="919" alt="image" src="https://github.com/user-attachments/assets/152186f4-0275-4562-b684-1c7c0ed41c90" />

<img width="1245" height="1146" alt="image" src="https://github.com/user-attachments/assets/52aff7ea-5910-4a4e-99be-2ddb4e2b7804" />

All persistent data is stored using **bind mounts**, aligned with Zettlab’s native app layout for easy backup and restore.

---

## DNS Requirements

Your internal DNS (Pi-hole, router DNS, etc.) must point the following to **Caddy’s macvlan IP** (example: `192.168.x.60`):

```
traefik.lab.home   → 192.168.x.x
jellyfin.lab.home  → 192.168.x.x
*.lab.home         → 192.168.x.x (wildcard recommended)
```

<img width="1149" height="395" alt="image" src="https://github.com/user-attachments/assets/1028c34e-a0d4-4896-b1b9-dc34a2c8f380" />

---

## TLS / Certificates

* Uses a **self-signed wildcard certificate** (`*.lab.home`)
* Same certificate is reused by **Caddy and Traefik**
* Clients must trust the cert (import into OS trust store)

<img width="1289" height="479" alt="image" src="https://github.com/user-attachments/assets/4d0bdcd6-f626-4431-92a0-a862a138d157" />

<img width="1227" height="611" alt="image" src="https://github.com/user-attachments/assets/7effa446-a11a-409c-8f5a-b8c9e6c24347" />

This avoids:

* ACME complexity on internal networks
* Browser warnings once trusted
* Multiple certs per service

---

## Service Onboarding (Example)

Any service that should be exposed:

1. Attach to `traefik-net`
2. Add Traefik labels

Example (Jellyfin):

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.jellyfin.rule=Host(`jellyfin.lab.home`)
  - traefik.http.routers.jellyfin.entrypoints=websecure
  - traefik.http.routers.jellyfin.tls=true
  - traefik.http.services.jellyfin.loadbalancer.server.port=8096
```

No ports. No macvlan. No TLS config per app.

---

## Security Considerations

* Traefik has **no externally reachable ports**
* Dashboard is reachable **only through Caddy**
* macvlan isolates the edge from the NAS host
* Optional IP allowlists or auth can be enforced in Caddy
* Docker socket is mounted **read-only**

---

## Why Not Expose Traefik Directly?

| Option           | Outcome                                    |
| ---------------- | ------------------------------------------ |
| Expose Traefik   | Port conflicts, weaker isolation           |
| macvlan Traefik  | Dashboard + routers share same exposure    |
| Caddy front door | ✅ Clean separation, minimal attack surface |

This repository intentionally chooses the **cleanest boundary**.

---

## Tested Environment

* Zettlab NAS
* Docker / Docker Compose
* macvlan network pre-created by NAS
* Bridge network: `traefik-net`
* Internal domain: `lab.home`

---

## Future Enhancements

* 🔐 Caddy IP allowlist for dashboard
* 📊 Traefik Prometheus metrics
* 🧱 Docker socket proxy
* 🔄 Gradual migration of legacy apps behind Traefik
* 🌐 Split-DNS / mTLS

---

## Philosophy

> **Caddy is the door.
> Traefik is the hallway.
> Containers are rooms.**

Each component does one job — and does it well.

---
