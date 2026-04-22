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
       ├────────────── fs-frontend, fs-backend, fs-zitadel   (FifthSeason)
       └────────────── tp-web                                (Tourenplanung)
```

Alle App-Stacks hängen sich an ein gemeinsames externes Docker-Netz `web`. Nur Caddy hat Ports nach außen freigegeben; Datenbanken & interne Services bleiben auf projekt-privaten Netzen.

## Dateien

- `docker-compose.yml` — Caddy-Container, bindet :80/:443, hängt am `web`-Netz.
- `Caddyfile` — Site-Blöcke pro Domain.
- `.env.example` — Vorlage für Server-`.env` (wird im Deploy aus GitHub-Secrets erzeugt).

## Server-Setup (einmalig)

1. Docker & Docker Compose installieren.
2. Gemeinsames Netz anlegen:
   ```bash
   docker network create web
   ```
3. DNS-Records auf den Server zeigen lassen (A-Record für jede Domain bzw. Wildcard für Subdomains).

## Secrets (im GitHub-Repo)

| Name | Zweck |
|---|---|
| `SERVER_HOST` | Hostname/IP des Servers |
| `SERVER_USER` | SSH-User |
| `SERVER_SSH_KEY` | privater SSH-Key (OpenSSH-Format) |
| `ACME_EMAIL` | E-Mail für Let's Encrypt/ZeroSSL |
| `FIFTHSEASON_DOMAIN` | Domain von FifthSeason (z. B. `fifth-season.de`) |

## Deployment

Der Workflow liegt aktuell auf **`workflow_dispatch`** (manueller Trigger), solange die FifthSeason-Migration noch nicht abgeschlossen ist. Grund: Die Edge-Caddy würde sofort mit der alten FifthSeason-Caddy um `:80/:443` streiten.

**Nach der Migration** (siehe unten) in `.github/workflows/deploy.yml` auf `push: branches: [main]` umstellen.

## Migration von FifthSeason (in dessen Repo durchzuführen)

1. `caddy`-Service aus `docker-compose.yml` entfernen.
2. Alle verbleibenden Services ins externe Netz `web` hängen und eindeutig benennen:
   - `frontend` → `container_name: fs-frontend`
   - `backend` → `container_name: fs-backend`
   - `zitadel` → `container_name: fs-zitadel`
   - `postgres` bleibt in einem projekt-internen Netz, **nicht** am `web`-Netz.
3. Wartungsfenster:
   ```bash
   # auf dem Server
   cd ~/fifthseason && docker compose down
   cd ~/edge && docker compose up -d
   cd ~/fifthseason && docker compose up -d
   ```
4. Optional: Zertifikate aus dem alten FifthSeason-`caddy_data`-Volume ins neue Edge-Volume kopieren, um Neuabruf zu vermeiden.

## Neue App hinzufügen

1. In der App-`docker-compose.yml`: Service ins externe Netz `web` hängen, eindeutigen `container_name` vergeben, Port **nicht** nach außen exposen.
2. Im `Caddyfile` dieses Repos einen neuen Site-Block ergänzen:
   ```caddyfile
   app.hdave.net {
       reverse_proxy my-app:3000
   }
   ```
3. Workflow ausführen (`Actions → Deploy Edge → Run workflow`). Caddy lädt die Config live per `caddy reload` nach — keine Downtime für die anderen Sites.

## Lokale Validierung

```bash
docker run --rm \
  -v "$(pwd)/Caddyfile:/etc/caddy/Caddyfile" \
  -e ACME_EMAIL=test@example.com \
  -e FIFTHSEASON_DOMAIN=example.com \
  caddy:2-alpine caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```
