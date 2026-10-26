# SkySend + Pocket ID — Setup-Dokumentation (VPS AlmaLinux 9.8)

**Server:** host.mojomail.me · 194.127.179.3 · 4 vCPU / 8 GB RAM · Centminmod (Nginx unter `/usr/local/nginx`), CSF-Firewall, acme.sh für Zertifikate
**Stand:** 26.10.2026 · SkySend 3.2.0 · Pocket ID 2.19.1

---

## Übersicht

```
┌─ https://share.mojobus.co ─────────────────────┐
│  SkySend — E2E-verschlüsseltes File/Note-Sharing│
│  Upload/Link-Erstellen: nur mit Pocket-ID-Login │
│  Download: frei für alle mit dem Link           │
│  Podman-Container, 127.0.0.1:3020               │
└──────────────┬──────────────────────────────────┘
               │ OIDC (Authorization Code + PKCE)
┌─ https://auth.mojobus.co ──────────────────────┐
│  Pocket ID — Passkey-Identity-Provider          │
│  6 Konten: 1 Admin + 5 User                     │
│  Podman-Container, 127.0.0.1:3011               │
└─────────────────────────────────────────────────┘
        beide hinter Nginx (TLS via Let's Encrypt)
```

**Sicherheitsmodell:**
- Dateien/Notizen werden im Browser mit AES-256-GCM verschlüsselt, Schlüssel liegt nur im URL-Fragment (#...) → Server sieht nur Chiffretext
- Bei passwortgeschützten Shares: Argon2id, Passwort verlässt den Browser nie
- Uploads nur für die eigene Runde (OIDC), Empfangen öffentlich per Link
- Keine externen Dienste in der Kette

---

## 1. SkySend

### 1.1 Dateien

- `/opt/skysend/docker-compose.yml` — Container-Definition
- `/opt/skysend/data/` — SQLite-Datenbank (Metadaten, im Klartext!)
- `/opt/skysend/uploads/` — verschlüsselte Dateien (Chiffretext)

### 1.2 docker-compose.yml (kompletter Stand)

```yaml
services:
  skysend:
    image: docker.io/skyfay/skysend:latest
    container_name: skysend
    restart: always
    ports:
      - "127.0.0.1:3020:3000"    # Host-Port 3020 (3000 ist durch analytics-Next.js belegt!)
    volumes:
      - /opt/skysend/data:/data
      - /opt/skysend/uploads:/uploads
    environment:
      - BASE_URL=https://share.mojobus.co
      - TRUST_PROXY=true
      # Ablaufzeiten (Sekunden): 10min, 1h, 6h, 1 Tag, 1 Woche, 1 Monat, 1 Jahr, 3 Jahre, 5 Jahre
      - FILE_EXPIRE_OPTIONS_SEC=600,3600,21600,86400,604800,2592000,31536000,94608000,157680000
      - FILE_DEFAULT_EXPIRE_SEC=604800
      - NOTE_EXPIRE_OPTIONS_SEC=600,3600,21600,86400,604800,2592000,31536000,94608000,157680000
      - NOTE_DEFAULT_EXPIRE_SEC=604800
      - FILE_REQUEST_EXPIRE_OPTIONS_SEC=86400,604800,2592000,31536000
      - FILE_REQUEST_DEFAULT_EXPIRE_SEC=604800
      # --- OIDC: Upload nur mit Login, Downloads bleiben frei ---
      - OIDC_PROVIDER=pocketid
      - OIDC_ISSUER=https://auth.mojobus.co
      - OIDC_CLIENT_ID=<aus Pocket ID, Client "skysend">
      - OIDC_CLIENT_SECRET=<aus Pocket ID, Client "skysend">
      - OIDC_REDIRECT_URI=https://share.mojobus.co/auth/callback
      - OIDC_SESSION_SECRET=<openssl rand -base64 48>
      - OIDC_PROTECT_FILES=true
      - OIDC_PROTECT_NOTES=true
      - OIDC_PROTECT_REQUESTS=true
    mem_limit: 512m     # Remotion auf derselben Kiste — hartes Limit
    cpus: '0.5'
```

**Wichtig:** Der Default-Ablaufwert muss in der Options-Liste enthalten sein, sonst startet der Server nicht sauber.

### 1.3 Nginx-VHost `/usr/local/nginx/conf/conf.d/share.mojobus.co.ssl.conf`

Wesentliche Änderungen gegenüber Centminmod-Default:
- `location /` → `proxy_pass http://127.0.0.1:3020` mit WebSocket-Support
- `php.conf` und `pre-staticfiles-*`/`staticfiles.conf` auskommentiert
- `cloudflare.conf` NICHT inkludiert (Cloudflare-Proxy ist off/DNS-only)
- Backup: `share.mojobus.co.ssl.conf.bak-20261026`

Kernblock:
```nginx
  location / {
  include /usr/local/nginx/conf/503include-only.conf;
  client_max_body_size 0;
  proxy_pass http://127.0.0.1:3020;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection $connection_upgrade;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_connect_timeout 60s;
  proxy_send_timeout    600s;
  proxy_read_timeout    600s;
  proxy_buffering off;
  proxy_request_buffering off;
  }
```

WebSocket-Map (einmalig, für alle VHosts): `/usr/local/nginx/conf/conf.d/map-connection-upgrade.conf`
```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

### 1.4 TLS

- Zertifikat: Let's Encrypt (ECC), via `~/.acme.sh/acme.sh` — **Pfad:** `/usr/local/nginx/conf/ssl/share.mojobus.co/share.mojobus.co.{crt,key}`
- Installation/Erneuerung:
  ```bash
  ~/.acme.sh/acme.sh --install-cert -d share.mojobus.co --ecc \
    --cert-file /usr/local/nginx/conf/ssl/share.mojobus.co/share.mojobus.co.crt \
    --key-file  /usr/local/nginx/conf/ssl/share.mojobus.co/share.mojobus.co.key \
    --fullchain-file /usr/local/nginx/conf/ssl/share.mojobus.co/share.mojobus.co.crt \
    --reloadcmd "nginx -s reload"
  ```
- Renewal automatisch (ARI-Fenster, ca. alle 60 Tage)
- **Achtung:** Erst versuchter Weg über Centminmod acmetool ergab ein Cloudflare-Origin-Cert (100 Jahre, nur CF-trusted) → Browser-Warnung. Richtiges LE-Cert kam via acme.sh (siehe oben). `--ecc` nicht vergessen!

### 1.5 Bedienung

```bash
cd /opt/skysend
podman-compose up -d          # starten
podman-compose down           # stoppen
podman-compose pull && podman-compose up -d   # Update
podman logs -f skysend        # Logs
curl -s https://share.mojobus.co/api/health   # Health-Check
```

### 1.6 Env-Referenz (alle Variablen)

Doku: https://docs.skysend.app/user-guide/configuration/environment-variables
Wichtige aus dem Code (`apps/server/src/lib/config.ts`): `FILE_EXPIRE_OPTIONS_SEC`, `FILE_DEFAULT_EXPIRE_SEC`, `NOTE_*` (gleiches Muster), `FILE_REQUEST_*`, `TRUST_PROXY`, `BASE_URL`, `MAX_UPLOAD_SIZE` (Limit regelt SkySend selbst; Nginx hat `client_max_body_size 0`).

---

## 2. Pocket ID

### 2.1 Dateien

- `/opt/pocketid/docker-compose.yml` — Container-Definition
- `/opt/pocketid/.env` — enthält **ENCRYPTION_KEY** (kritisch, sichern!)
- `/opt/pocketid/data/` — SQLite-DB + Uploads, gehört UID 1000

### 2.2 docker-compose.yml

```yaml
services:
  pocket-id:
    image: docker.io/pocketid/pocket-id:v2
    container_name: pocket-id
    restart: always
    ports:
      - "127.0.0.1:3011:1411"
    env_file: .env
    volumes:
      - /opt/pocketid/data:/app/data
    mem_limit: 256m
    cpus: '0.3'
```

### 2.3 .env

```
APP_URL=https://auth.mojobus.co     # Passkeys sind an diese Domain gebunden — nie ändern!
ENCRYPTION_KEY=<openssl rand -base64 32>   # verschlüsselt Signing-Keys — Backup der Datei machen!
TRUST_PROXY=true
PUID=1000
PGID=1000
```

### 2.4 Nginx-VHost `/usr/local/nginx/conf/conf.d/auth.mojobus.co.ssl.conf`

Wie share-VHost, aber:
- `proxy_pass http://127.0.0.1:3011`
- **`drop.conf` und `autoprotect` AUSKOMMENTIERT** — sie blocken SvelteKit-Routen (`__data.json`, `/_app/...`) und verursachen sonst „Etwas ist schiefgalaufen / Internal Error" in der Admin-UI (GitHub Issue pocket-id#1606/#1715 — gleiche Ursache)
- `php.conf` auskommentiert, keine WebSocket-Header nötig

### 2.5 TLS

Wie bei share: acme.sh, `--ecc`, Pfade `/usr/local/nginx/conf/ssl/auth.mojobus.co/auth.mojobus.co.{crt,key}`
Port-80-VHost mit `/.well-known/acme-challenge/`-Location war Voraussetzung für die Validierung.

### 2.6 Einrichtung (erledigt)

1. **Admin:** `https://auth.mojobus.co/setup` (nur bis erster Account existiert), Passkey registriert
2. **User:** 5 Konten angelegt; Login-Code pro Person (⋯-Menü → Login Code, kurz gültig, per Messenger geschickt); Person legt eigenen Passkey an
3. **OIDC-Client „skysend":**
   - Typ: **Vertraulich (Confidential)**
   - Callback-URL: `https://share.mojobus.co/auth/callback` (exakt, kein trailing slash)
   - Access: **All Users** (wichtig — sonst „You are not allowed to access this service"; Admins sind z. B. nicht in „Authenticated Users")
   - Client ID + Secret: nur einmal sichtbar; neu generieren unter Credentials, falls verloren

### 2.7 Login-Methoden (ohne Biometrie möglich)

- Passkey-Entsperrung per **Geräte-PIN** (kein Fingerabdruck nötig) — Admin-Setting: Application Configuration → Passkeys → **User verification: Preferred** (statt Required)
- Passkey im **Passwort-Manager** (Bitwarden/1Password) — Entsperrung mit Master-Passwort
- **„Sign in with another device"** — QR-Code, Freigabe am Handy (5 Min gültig)
- **Login-Code** (Admin-generiert, einmalig) — für Erst-Setup/Verlust-Fall
- Kein klassisches Passwort — bewusstes Design (passwordless)

### 2.8 Recovery

Admin-Passkey verloren:
```bash
podman exec pocket-id /app/pocket-id one-time-access-token
```
→ gibt Einmal-Login-Link (1 h gültig).

### 2.9 Bedienung

```bash
cd /opt/pocketid
podman-compose up -d / down
podman-compose pull && podman-compose up -d   # Update
podman logs -f pocket-id
```

---

## 3. Fehler, die wir hatten (und Lösungen)

| Problem | Ursache | Lösung |
|---|---|---|
| `acmetool.sh: command not found` | Centminmod-Werkzeug nicht im PATH | Direkt `~/.acme.sh/acme.sh` genutzt |
| Zertifikat 100 Jahre gültig, Browser-Warnung, curl 000 | acmetool hatte CF-Origin-Cert erzeugt (nur CF-trusted, Proxy war aber off) | LE-Cert via `acme.sh --issue` + `--install-cert --ecc` |
| `nginx: [emerg] unknown "connection_upgrade"` | Map fehlte | `conf.d/map-connection-upgrade.conf` angelegt |
| Pocket-ID-Admin-UI: „Etwas ist schiefgalaufen / Internal Error" bei Client-Erstellen/Editieren | Centminmod `drop.conf` + `autoprotect` blockten SvelteKit-Routen | Beide Includes im auth-VHost auskommentiert (Issue pocket-id#1606) |
| `You are not allowed to access this service` beim Login | OIDC-Client-Access nicht auf „All Users" (Admin nicht in Gruppe) | Access-Tab → All Users |
| Health-Check ok lokal, extern 000 | (vor Cert-Tausch:) Zertifikatsvertrauen | siehe LE-Cert-Fix |

---

## 4. Betrieb / Wartung

### Regelmäßig

```bash
# Updates (beide Dienste):
cd /opt/skysend && podman-compose pull && podman-compose up -d
cd /opt/pocketid && podman-compose pull && podman-compose up -d

# Disk im Blick (5-Jahres-Shares sammeln sich an):
du -sh /opt/skysend/uploads

# Health:
curl -s https://share.mojobus.co/api/health
curl -s -o /dev/null -w "%{http_code}\n" https://auth.mojobus.co
```

### Backup-Empfehlung

- **Sichern:** `/opt/pocketid/.env` (ENCRYPTION_KEY!), `/opt/skysend/docker-compose.yml` (OIDC-Client-Secret!), Nginx-VHosts beider Domains
- **SQLite-Backups** (beide DBs) nur mit Bewusstsein: SkySend-DB enthält Metadaten + Auth-Token-Hashes im Klartext; Dateiinhalte bleiben verschlüsselt
- Pocket-ID-DB: enthält nur Public Keys der Passkeys — DB-Diebstahl erlaubt keinen Login

### Architektur-Hinweise

- Port 3000 = analytics-Next.js (belegt!) → SkySend auf 3020, Pocket ID auf 3011, beide localhost-only
- CSF: keine extra Ports nötig (Nginx macht 80/443)
- Cloudflare: DNS-only für beide Subdomains. Falls später Proxy AN: Cloudflare.conf-Include im VHost aktivieren, SSL-Modus „Full (strict)", Upload-Limit 100 MB (Free) beachten
- Ressourcen: SkySend ~100–150 MB, Pocket ID ~50–80 MB; Limits (512m/256m) schützen Remotion-Renders

### Zukünftige Apps an Pocket ID anbinden

Nur noch: OIDC-Client in Pocket ID anlegen (Callback-URL der neuen App) → Client-ID/Secret in die App → 2 Minuten. Fast 100 Apps haben fertige Anleitungen: https://pocket-id.org/docs/setup/client-examples
