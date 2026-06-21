# docker-compose-deploy-hardening

A reusable [Claude Code](https://claude.com/claude-code) skill for taking a
multi-container Docker Compose app live on a cloud VM with a minimal, safe public
surface.

## What it does

- **Minimize published ports.** Core principle: a `ports:` mapping publishes to
  the host's public IP, but containers talk to each other over the internal
  network regardless — so only the single public entry point needs a published
  port. Everything else (API, DB, admin UI) keeps working with `ports:` removed.
- **Add HTTPS via a reverse proxy.** Drop in a Caddy container on 80/443 for
  automatic Let's Encrypt certs, with cert persistence in a named volume.
- **Lock down at the provider.** Security-group rules allowing only 22/80/443.
- **Avoid the classic trap:** Docker bypasses `ufw` (rules land in `DOCKER-USER`
  before `ufw`), so rely on not publishing the port + the cloud firewall.

Includes a verification checklist (`docker ps`, `curl` probes, cert-persistence
check) and a region note (mainland-China ICP filing vs. Singapore/HK).

## When it triggers

When preparing a Docker Compose app for production, or when you ask to "expose
only port X", "hide a port before going live", "deploy docker-compose to a cloud
server", or add a reverse proxy / HTTPS.

## Install

```bash
mkdir -p ~/.claude/skills/docker-compose-deploy-hardening
cp SKILL.md ~/.claude/skills/docker-compose-deploy-hardening/
```

The skill is model-invoked: Claude activates it automatically when the task
matches the description above.
