# hdave-edge

Zentraler Reverse-Proxy (Caddy) für alle Apps auf dem Hetzner-Server. Hält die Host-Ports `:80` und `:443`, terminiert TLS und leitet Traffic anhand des Hostnamens an die jeweiligen App-Container weiter.

## Architektur

```
Internet
   ↓ :80/:443
┌──────────────┐
│  edge/Caddy  │  ← dieses Repo
└──────┬───────┘
       │ docker network "web"
       └────────────── tp-web        (Tourenplanung)
       └────────────── fs-*          (FifthSeason, nach Migration)
```

Alle App-Stacks hängen sich an ein gemeinsames externes Docker-Netz `web`. Nur Caddy hat Ports nach außen freigegeben; Datenbanken & interne Services bleiben auf projekt-privaten Netzen.

## Dateien

- `docker-compose.yml` — Caddy-Container, bindet :80/:443, hängt am `web`-Netz.
- `Caddyfile` — Site-Blöcke pro Domain. Domains und ACME-Email sind hardcoded; Änderungen triggern die Pipeline, fertig.

## Server-Setup (einmalig)

1. Docker & Docker Compose installieren.
2. Gemeinsames Netz anlegen — passiert automatisch beim ersten Deploy, aber manuell geht auch:
   ```bash
   docker network create web
   ```
3. DNS-Records auf den Server zeigen lassen (A-Record für jede Domain bzw. Wildcard für Subdomains).

## Secrets (im GitHub-Repo)

| Name | Zweck |
|---|---|
| `SERVER_HOST` | Hostname/IP des Servers |
| `SERVER_USER` | SSH-User |
| `SERVER_SSH_KEY` | privater SSH-Key (OpenSSH-Format, ohne Passphrase) |

Mehr braucht's nicht — alles andere ist direkt im Caddyfile.

## Deployment

Auto-Deploy auf jeden Push auf `main`. Der Workflow kopiert `Caddyfile` + `docker-compose.yml` per SCP auf den Server, stellt sicher dass das `web`-Netz existiert, zieht neue Images und lädt die Caddy-Config live nach (`caddy reload`, keine Downtime).

## FifthSeason-Migration (separat durchzuführen)

Beim Umzug von FifthSeason auf den Edge:

1. `caddy`-Service aus FS `docker-compose.yml` entfernen.
2. Services ins externe Netz `web` hängen, eindeutige `container_name` vergeben:
   - `frontend` → `fs-frontend`
   - `backend` → `fs-backend`
   - `zitadel` → `fs-zitadel`
3. In diesem Repo einen Site-Block für FS im `Caddyfile` ergänzen (Pattern siehe alter FS-Caddyfile).
4. Wartungsfenster:
   ```bash
   cd ~/fifthseason && docker compose down
   cd ~/edge && docker compose up -d
   cd ~/fifthseason && docker compose up -d
   ```
5. Optional: Zertifikate aus dem alten FS-`caddy_data`-Volume ins neue Edge-Volume kopieren, um Neuausstellung zu sparen.

## Neue App hinzufügen

1. In der App-`docker-compose.yml`: Service ins externe Netz `web` hängen, eindeutigen `container_name` vergeben, Port **nicht** nach außen exposen.
2. Im `Caddyfile` dieses Repos einen neuen Site-Block ergänzen:
   ```caddyfile
   app.hdave.net {
       reverse_proxy my-app:3000
   }
   ```
3. Commit & push. Workflow reloaded Caddy live.

## Lokale Validierung

```bash
docker run --rm -v "$(pwd)/Caddyfile:/etc/caddy/Caddyfile" \
  caddy:2-alpine caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```
