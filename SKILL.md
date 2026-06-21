---
name: docker-compose-deploy-hardening
description: This skill should be used when preparing a Docker Compose application for public/production deployment, or when the user asks how to "expose only port X", "hide a port before going live", "deploy docker-compose to a cloud server", add a reverse proxy / HTTPS, or harden which ports are reachable. Covers minimizing published ports, adding a Caddy/Nginx reverse proxy for HTTPS, cloud security-group rules, and the Docker-bypasses-ufw firewall trap.
version: 1.0.0
---

# Docker Compose Deploy Hardening

Guidance for taking a multi-container Docker Compose app live on a cloud VM with
a minimal, safe public surface. Use before deploying, or when locking down ports
and adding HTTPS.

## Core principle: published ports ≠ inter-container traffic

A `ports:` mapping (`"8088:8081"`) publishes a port onto the **host's public
IP**. Containers talk to each other over their shared Compose/bridge network
(by service name or static IP) **regardless** of whether any port is published.

Therefore: **only the single public entry point needs a published port.** Every
other service (API, database, admin UI) keeps working over the internal network
with its `ports:` removed. Trace the data flow and find the one hop that crosses
the host boundary (usually browser → web frontend) — publish only that.

### Applying it
- `ports:` live in the **module/service** compose files. A top-level file that
  uses `extends:` cannot un-publish a port declared in the extended file — edit
  the source file.
- Prefer **commenting out** unused `ports:` with a one-line note (why it's safe +
  how to re-enable for local debugging) rather than deleting.
- Verify the merged result: `docker-compose -f ./docker-compose.yml config`,
  then confirm exactly one `published:` entry remains.
- Runtime check: `docker ps --format '{{.Names}}: {{.Ports}}'` — only the entry
  point should show a `0.0.0.0` mapping.

## Reverse proxy for HTTPS (Caddy recommended)

A frontend like Next.js does not terminate TLS. To serve `https://<domain>/`
instead of `http://<ip>:3000`, add a proxy container on 80/443.

**Caddy** is the low-config choice (automatic Let's Encrypt):
- Add a `caddy:2-alpine` service publishing `80` and `443`, on the same network.
- Mount a `Caddyfile`:
  ```
  your-domain.com {
      reverse_proxy <frontend-host>:<port>
  }
  ```
  The bare-domain block auto-provisions a cert and redirects HTTP→HTTPS.
- **Persist certs** in a named volume (`caddy_data:/data`). Without it, every
  restart re-requests a cert and can hit Let's Encrypt's limit (5/week/domain).
  Make any volume-prune step skip caddy volumes
  (`docker volume ls -qf dangling=true | grep -v caddy | xargs -r docker volume rm`).
- After this, the frontend stops publishing its own port — Caddy fronts it.

**Precondition:** the domain's A record must point at the server IP *before*
first boot, or cert issuance fails.

## Cloud provider firewall (the authoritative layer)

Configure the provider's **security group** (AWS/Aliyun/etc.): allow inbound
only **22 (SSH)**, **80**, **443**. Deny app/DB ports (3000/8086/27017/…).

⚠️ **Docker bypasses `ufw`.** Docker writes rules into the `DOCKER-USER` iptables
chain evaluated *before* `ufw`, so `ufw deny 8088` is silently ineffective for a
published port. Rely on (a) not publishing the port in Compose, and (b) the
cloud security group — not on host `ufw`.

## Region note
For audiences in mainland China, Hetzner/DO can be slow/blocked; a Singapore/HK
node or a China region is better. China *mainland* regions require ICP filing;
Singapore does not.

## Verification checklist (on the host after deploy)
1. `docker ps` → only the proxy/entry point shows `0.0.0.0:80/443`.
2. `curl -I http://<domain>` → redirects to https; `curl https://<domain>/` renders.
3. Full chain through the proxy: hit an endpoint that exercises frontend→API→DB.
4. `curl --max-time 3 http://<domain>:<hidden-port>` → fails (confirms hidden).
5. Restart and confirm the proxy does NOT re-request a TLS cert (volume persisted).
