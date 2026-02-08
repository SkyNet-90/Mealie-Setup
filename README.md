# 🍲 Mealie Self-Hosted Stack

Self-hosted [Mealie](https://docs.mealie.io/) recipe manager deployed via **Portainer**, exposed publicly through a **Cloudflare Tunnel** at **recipes.jo-jo.net**, with automatic container updates via **Watchtower**.

## Architecture

| Service | Purpose |
|---|---|
| **Mealie** (`ghcr.io/mealie-recipes/mealie`) | Recipe manager (SQLite backend) |
| **Cloudflared** (`cloudflare/cloudflared`) | Zero-trust tunnel → `recipes.jo-jo.net` |
| **Watchtower** (`containrrr/watchtower`) | Automatic image updates (every 24 h) |

All three services sit on an isolated bridge network. No host ports are published — all public traffic flows through the Cloudflare Tunnel.

---

## Prerequisites

- Docker host with [Portainer](https://www.portainer.io/) installed
- A [Cloudflare](https://dash.cloudflare.com/) account with the `jo-jo.net` domain
- This repo pushed to GitHub (for Portainer Git-based stacks)

---

## Setup

### 1. Create the Cloudflare Tunnel

1. Go to **Cloudflare Zero Trust → Networks → Tunnels → Create a tunnel**.
2. Name it `mealie` (or whatever you prefer).
3. Copy the **Tunnel Token** — you'll need it in Step 3.
4. Add a **Public Hostname** route:
   | Field | Value |
   |---|---|
   | Subdomain | `recipes` |
   | Domain | `jo-jo.net` |
   | Type | `HTTP` |
   | URL | `mealie:9000` |

   > The URL uses the Docker service name because `cloudflared` and `mealie` share the same Docker network.

5. **Recommended** — under the tunnel's **Settings → TLS**, leave *"No TLS Verify"* **off** (default). Cloudflare handles HTTPS at the edge; the internal hop is plain HTTP on port 9000 inside the private network.

### 2. Push this repo to GitHub

```bash
git init
git add .
git commit -m "Initial Mealie stack"
git remote add origin git@github.com:<your-user>/Mealie-Setup.git
git push -u origin main
```

### 3. Deploy in Portainer

1. **Stacks → Add Stack → Git Repository**
2. Fill in:
   | Field | Value |
   |---|---|
   | Repository URL | `https://github.com/<your-user>/Mealie-Setup` |
   | Repository reference | `refs/heads/main` |
   | Compose path | `docker-compose.yml` |
3. Scroll to **Environment variables** and add each variable from [.env.example](.env.example).
   - The **only hard requirement** is `CLOUDFLARE_TUNNEL_TOKEN`.
   - Everything else has sensible defaults baked into the compose file.
4. **Enable "GitOps updates"** if you want Portainer to auto-redeploy when you push changes to the repo.
5. Click **Deploy the stack**.

> **Why enter env vars in Portainer instead of a `.env` file?**
> Portainer stores env vars encrypted in its own database, so secrets never touch disk on the Docker host and are never committed to Git.

### 4. First Login

1. Browse to **https://recipes.jo-jo.net**
2. Log in with the default credentials:
   | | |
   |---|---|
   | **Email** | `changeme@example.com` |
   | **Password** | `MyPassword` |
3. **Immediately** change the admin email & password under your profile settings.
4. Verify the install at **Admin → Site Settings** — Mealie shows a summary of config status.

---

## Security Hardening Checklist

- [x] `ALLOW_SIGNUP` is `false` — no open registration
- [x] Containers run with `no-new-privileges` security option
- [x] Mealie memory is capped at 1 GB to limit resource abuse
- [x] No host ports are exposed — all traffic goes through the Cloudflare Tunnel
- [x] The `.env` file is git-ignored; secrets live in Portainer only
- [x] Failed-login lockout is enabled (5 attempts → 24 h lockout)
- [x] Watchtower is label-scoped — it only touches containers in this stack
- [ ] **You** should change the default admin email + password on first login
- [ ] **You** should enable Cloudflare WAF / rate-limiting rules on `recipes.jo-jo.net` to mitigate DoS on scraping endpoints (`/api/recipes/create/url`, `/api/recipes/{id}/image`)
- [ ] **Optional**: configure SMTP so password resets & invitations work via email

---

## Automatic Updates

Watchtower checks for new images every 24 hours (configurable via `WATCHTOWER_POLL_INTERVAL`). It only updates containers labelled with `com.centurylinklabs.watchtower.enable=true` — which includes Mealie and Cloudflared in this stack.

To pin Mealie to a specific version and skip auto-updates, set `MEALIE_IMAGE_TAG` to a fixed tag (e.g. `v3.10.2`) and remove the watchtower label from the mealie service.

---

## Backups

Mealie stores all data in the `mealie-data` Docker volume. Use Mealie's built-in backup feature (**Admin → Backups**) to create `.zip` exports and download them off-server regularly.

---

## Useful Links

- [Mealie Docs](https://docs.mealie.io/)
- [Backend Config Reference](https://docs.mealie.io/documentation/getting-started/installation/backend-config/)
- [Security Considerations](https://docs.mealie.io/documentation/getting-started/installation/security/)
- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Watchtower Docs](https://containrrr.dev/watchtower/)
