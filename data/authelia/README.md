# Authelia (2FA) for the Traefik + CrowdSec stack

Adds [Authelia](https://www.authelia.com/) as the identity provider, so the
Traefik dashboard (`traefik.yourdomain.com`) requires password + TOTP — the
same `authelia@docker` forward-auth flow any protected service uses.

Flow for a protected service:

```text
Internet -> Traefik -> CrowdSec bouncer -> Authelia login + TOTP -> service
```

Only routers that reference `authelia@docker` (with a matching `two_factor`
rule in `configuration.yml`) are affected.

## Files

- `config/configuration.yml.sample`  — Authelia server config (committed)
- `config/users_database.yml.sample` — user list with Argon2 hash (committed)
- `secrets/`                         — session/JWT keys (NOT committed)
- `database/`, `notification/`       — runtime state (NOT committed)

## 1. DNS

Create an A/AAAA record pointing at the VPS:

```text
auth.yourdomain.com -> <VPS IP>
```

## 2. Copy templates + generate secrets

```bash
cd /opt/containers/traefik-crowdsec-stack

cp data/authelia/config/configuration.yml.sample data/authelia/config/configuration.yml
cp data/authelia/config/users_database.yml.sample data/authelia/config/users_database.yml

mkdir -p data/authelia/secrets data/authelia/database data/authelia/notification
openssl rand -hex 64 > data/authelia/secrets/session_secret
openssl rand -hex 64 > data/authelia/secrets/storage_encryption_key
openssl rand -hex 64 > data/authelia/secrets/jwt_secret
chmod 600 data/authelia/secrets/*
```

## 3. Replace placeholders

In `configuration.yml`, replace `yourdomain.com` with your base domain and
`auth.yourdomain.com` / `traefik.yourdomain.com` with your real hosts. In
`.env`, set `SERVICES_AUTHELIA_LABELS_AUTHELIA_HOST`.

## 4. User + password hash

```bash
docker run --rm -it authelia/authelia:latest authelia crypto hash generate argon2
```

Paste the full `Digest:` value into `users_database.yml`.

## 5. Start

```bash
docker compose up -d
docker compose logs -f authelia
```

## 6. Enroll TOTP

1. Open `https://traefik.yourdomain.com` — you are redirected to
   `https://auth.yourdomain.com`.
2. Log in, then read the registration link (filesystem notifier):

   ```bash
   cat data/authelia/notification/notification.txt
   ```

3. Open the link and scan the QR code with an authenticator app
   (Aegis, Ente Auth, 2FAS, Google/Microsoft Authenticator, …).

## Protecting another service

1. Add `authelia@docker` to the router's `middlewares` in its Compose labels.
2. Add a matching `two_factor` rule under `access_control.rules` in
   `configuration.yml`.
3. `docker compose up -d` to reload.

## Notes

- `default_policy: deny` only affects routers that use `authelia@docker`;
  add a `two_factor` rule for every such service or it will be denied.
- Do not route non-HTTP protocols (e.g. Syncthing 22000/tcp+udp, 21027/udp)
  through Authelia.
- To keep basic-auth as an extra layer on the dashboard, set the router
  middleware to `traefik-dashboard-auth@file,authelia@docker`.
