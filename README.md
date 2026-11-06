# SBC Provisioning Tool

A web app for managing customers on a pair of AudioCodes Mediant SBCs (7.4). It keeps a customer database, allocates slots and table indexes, and audits the two SBCs for drift against each other and against the database.

This repository holds only the install files (`install.sh`, `docker-compose.yml`, `Caddyfile`, `env.example`), published as [releases](../../releases). The application image is private.

> **Current version is read-only on the SBCs.** It reads their status and running configuration and never changes anything on them.

---

## Installation

### Requirements

- A Linux server running **Ubuntu 22.04 / 24.04 or Debian 12**, with root access (sudo).
- HTTPS (port 443) from the server to the management interface of every SBC you'll add.
- A DNS name for the server that users will browse to (e.g. `sbc-provisioning.example.internal`).
- The **GitHub username and `read:packages` token** for pulling the image. Both are stored in **Bitwarden**. (If you ever need to replace the token: it must be a *classic* personal access token with only the `read:packages` scope. Fine-grained tokens don't work with `ghcr.io`.)
- An AudioCodes **Administrator** account on both SBCs of each pair you'll add.

Docker Engine and the Compose plugin are installed automatically if they're missing.

### Install

```bash
curl -fsSL https://github.com/dangilroy/sbc-provisioning-release/releases/latest/download/install.sh | sudo bash
```

The installer asks for:

1. **Hostname**: the name users will browse to.
2. **TLS**:
   - `internal`: Caddy issues its own certificate. Browsers warn until Caddy's root CA is trusted.
   - `supplied`: use your own certificate. Put `cert.pem` and `key.pem` in `/opt/sbc-provisioning/certs/` (full chain in `cert.pem`).
3. **GitHub username and `read:packages` token**, to pull the image. Copy both from Bitwarden.

It then generates the secrets, starts the containers and prints a **first-time setup code** such as `ABCD-EFGH-JKLM`. Keep it handy for the next step.

Everything is installed in `/opt/sbc-provisioning`.

### First-time setup

1. Browse to `https://<hostname>/`. With no SBC pairs configured, the app opens the **First-time setup** page.
2. Enter the **setup code**.
3. Describe the first SBC pair:
   - **Pair name and colour**, e.g. *Production* / red. The colour is shown in the header so it's obvious which pair you're working on.
   - **SBC 1 / SBC 2 name and URL**, e.g. `https://sbc1.example.com`. Use the hostname on the SBC's certificate, not its IP address.
   - **Internal CA certificate** (optional): if the SBC certificates are issued by an internal CA, paste that CA's PEM here. Leave it blank for public CAs.
4. Enter your AudioCodes username and password. They must be **Administrator on both SBCs**.
5. Click **Create pair and log in**.

### Managing the installation

```bash
sudo /opt/sbc-provisioning/install.sh update            # latest release (takes a backup first)
sudo /opt/sbc-provisioning/install.sh update v0.3.4     # a specific release
sudo /opt/sbc-provisioning/install.sh backup            # DB dump + .env + certs -> /opt/sbc-provisioning/backups/
sudo /opt/sbc-provisioning/install.sh setup-code        # show the first-time setup code again
sudo /opt/sbc-provisioning/install.sh status            # container status
sudo /opt/sbc-provisioning/install.sh uninstall         # stop; optionally back up and delete data
```

Database migrations run automatically when the app starts after an update.

Backups contain `APP_SECRET_KEY` and the database password. Store them somewhere safe. To restore a database dump:

```bash
gunzip -c db.sql.gz | docker compose -f /opt/sbc-provisioning/docker-compose.yml exec -T db psql -U sbcprov -d sbcprov
```

If the token stops working (for example, it has expired), `update` asks for a new one. Get the current token from Bitwarden.

If the install stops part-way (for example, the registry token is rejected), fix the problem and run `sudo /opt/sbc-provisioning/install.sh` again. It resumes with the settings already entered.

#### Settings (`/opt/sbc-provisioning/.env`)

The installer fills this in. You rarely need to edit it; run `docker compose up -d` in `/opt/sbc-provisioning` afterwards.

| Setting | Meaning |
| --- | --- |
| `APP_HOSTNAME` | Hostname users browse to |
| `CADDY_TLS` | `internal`, or `/certs/cert.pem /certs/key.pem` |
| `APP_VERSION` | Image tag to run (`latest` or e.g. `v0.3.4`) |
| `APP_SECRET_KEY` | Encryption key for stored passwords. **Don't lose or change it**: stored service-account passwords would become unreadable. |
| `POSTGRES_PASSWORD` | Database password |
| `SESSION_IDLE_MINUTES` | Log out after this many idle minutes (default 30) |
| `ALLOW_DEGRADED_LOGIN` | Allow a read-only login when one SBC is unreachable (default `true`) |
| `LOG_LEVEL` | `INFO` by default |

SBC pairs, certificates and service accounts are **not** in `.env`. They're configured in the app.

---

## Using the tool

### Logging in

Log in with your **existing AudioCodes SBC account**. The app has no user accounts of its own.

1. Pick the **SBC pair** (e.g. Production, Dev).
2. Enter your SBC username and password.

The app checks your credentials against **both** SBCs in the pair:

- **Administrator on both** → full access to that pair.
- **Monitor on either** → read-only access.
- **One SBC unreachable** → read-only session with a banner. Log in again once both are back to get admin access.

Your role applies only to the selected pair. To change pairs, use **Switch** in the header. Your credentials are re-checked against the new pair.

Failed logins are rate-limited (5 per username per 15 minutes) so the tool can't lock out your SBC account.

Sessions end after 30 minutes idle or when you click **Log out**.

### SBC pairs

Each pair (Production, Dev, QA…) is separate: it has its own customers, slots and audits. Manage pairs under **SBC pairs**.

- **Add pair**: enter the details as in first-time setup. You must be Administrator on both new SBCs.
- **Edit / delete pair**: you must be Administrator on the existing pair's SBCs. If you change addresses or certificates, you must also be Administrator on the new SBCs.

These checks stop anyone from pointing a pair at an SBC they don't administer.

### Customers

**Customers** lists the customers on the current pair, with their slot, services, tags and status. Administrators can:

- **Add customer**: enter a name, tags and notes, and optionally a first service. The tool assigns the **lowest free slot** (00–99), checking both the database and the live config on both SBCs.
- **Add service** on a customer page: pick a platform (Teams, Mitel, Five9, Zoom, Other…), plus FQDN, channels and CLI as needed.

You never type an index. It's calculated as `platform prefix × 100 + slot`:

| Prefix | Platform |
| --- | --- |
| 1 | Teams |
| 2 | Mitel |
| 3 | Five9 |
| 4 | Other |
| 5 | Zoom |
| 6 | NetTranslate |
| 8 | Temp |

A customer in slot 05 gets index 105 for Teams, 205 for Mitel, and so on. A slot belongs to one customer across every prefix.

### Importing an existing spreadsheet

Use **Import** to load an existing customer list (Administrators only).

1. Upload a `.xlsx` or `.csv` file with these columns, one row per customer-platform:
   `Index, Platform, Name, FQDN, Tags, Notes, Channels Assigned, CLI`
2. Review the **preview**, which lists the customers to be created, any conflicts (such as a slot already in use, or a platform that doesn't match the index) and any skipped rows.
3. Click **Import** to commit. Nothing is written before that.

Imported customers are marked **unmanaged**.

### Audit

**Audit** compares the running config of both SBCs with each other and with the customer database. It's read-only.

- **Run audit now** (Administrators) reads both SBCs and lists the findings.
- **Scheduled audit**: under *SBC pairs → Edit*, tick *Audit this pair every day*, set a time (UTC) and enter a **service account**. The service account must be Administrator on both SBCs, because nobody is logged in when the scheduled audit runs.

The audit checks:

1. **SBC 1 vs SBC 2**: objects that exist on only one SBC, and fields or enabled states that differ. This covers proxy sets (including their proxy IPs), IP groups, SIP accounts, routing, message manipulations and similar tables.
2. **Database → SBCs**: every service has its proxy set and IP group on both SBCs.
3. **SBCs → database**: every proxy set or IP group in the managed range (100–899) belongs to a service.
4. **Index scheme**: indexes that don't fit the scheme.

Findings are grouped by area and rated **high / medium / low**. **New** marks a finding that wasn't in the previous run.

- Open a finding to see each SBC's value side by side, plus its history.
- **Acknowledge** a finding (individually or in bulk) to mark it as known. It stays acknowledged until the values change; then it reappears.

Device-encrypted secrets (values ending in `obscured`) are never compared or shown.

### Activity log

Logins, pair switches, pair changes, customer changes, imports, audits and acknowledgements are recorded in the database with who did what and when. A customer's page shows that customer's activity.

---

## How it works

### Components

The installer runs three Docker containers on one server:

| Container | Role |
| --- | --- |
| `caddy` | HTTPS front end (ports 80/443). Terminates TLS and forwards requests to the app. |
| `app` | The application (Python / FastAPI). Talks to the SBCs and the database. |
| `db` | PostgreSQL 16. |

Only Caddy is exposed. The app and database are reachable only on the internal Docker network.

### Authentication

- **No separate user accounts.** You log in with your AudioCodes SBC username and password. The app sends them to both SBCs in the chosen pair using HTTP Basic authentication over HTTPS.
  - Access is granted if `GET /api/v1/status` succeeds on both.
  - The app then calls `GET /api/v1/files`, which only Administrators can read: success means Administrator, a refusal means Monitor.
  - Your role is the lower of the two SBC results.
- **Sessions.** After login the browser gets a random, opaque session cookie (Secure, HTTP-only). The server stores only a hash of it. Your SBC password is stored **encrypted** (Fernet, keyed by `APP_SECRET_KEY`) in the session record. It's decrypted only at the moment the app needs to call an SBC on your behalf, and deleted when the session ends or expires.
- **Service accounts.** For scheduled audits, each pair can store a service-account password, also encrypted with `APP_SECRET_KEY`.
- **Setup code.** The first pair requires the setup code, which is derived from `APP_SECRET_KEY`. Only someone with access to the server can obtain it.
- Passwords and `Authorization` headers are never written to logs.

### Where data is stored

Everything lives in the PostgreSQL database (Docker volume `pgdata`) on the app server:

- SBC pair definitions (names, URLs, CA certificate, encrypted service-account password)
- Customers, slots and services
- Login sessions (encrypted SBC password, hashed token)
- Audit runs and findings, plus **compressed snapshots of each SBC's config** taken at every audit. Snapshots are kept for 90 days; audit runs for a year.
- Activity log

Other data on the server:

- `/opt/sbc-provisioning/.env`: startup settings and secrets (mode 600)
- `/opt/sbc-provisioning/certs/`: your TLS certificate, if supplied
- `/opt/sbc-provisioning/backups/`: backups taken by `install.sh backup` / `update`
- Docker volumes `caddy_data` / `caddy_config`: Caddy's certificates and state

The SBCs remain the source of truth for configuration. The database is the source of truth for intent: which customer owns which slot and which services they should have.

### How it reads the SBCs

The app uses the **AudioCodes REST API (`/api/v1`)** over HTTPS:

| Call | Used for |
| --- | --- |
| `GET /api/v1/status` | Check credentials and reachability |
| `GET /api/v1/files` | Detect Administrator vs Monitor |
| `GET /api/v1/files/cliScript` | Download the full running configuration as CLI text |

The cliScript is parsed into tables and rows (proxy sets with their proxy IPs, IP groups, routing rules…). These are used to:

- find indexes already in use when allocating a slot
- compare the two SBCs and the database during an audit

TLS certificates are verified against public CAs, or against the internal CA you supply for the pair. This is why SBCs are addressed by the hostname on their certificate.

This version makes **no write calls** to the SBCs.
