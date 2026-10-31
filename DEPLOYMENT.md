# Deploying Mavelle to AWS Lightsail

> Nothing in this repository provisions cloud resources. These are the steps a person
> runs; deployment is complete only once you've done them and the health check passes.

## 1. Choose an instance

Budget target: **₹1,000–₹3,000 / month** (roughly US$12–36). Prices below are Lightsail
Linux bundle list prices as last checked by the author — **confirm on the
[Lightsail pricing page](https://aws.amazon.com/lightsail/pricing/) before buying**, and add
GST and data transfer beyond the bundle allowance.

| Bundle | RAM / vCPU | Fits | Approx. |
|---|---|---|---|
| 2 GB | 2 GB / 2 | PostgreSQL + API + Caddy for an early beta (hundreds of active users). **No local AI.** | ~US$12 |
| 4 GB | 4 GB / 2 | Comfortable headroom for launch; still **no local LLM** | ~US$24 |
| 8 GB+ | 8 GB / 2 | A small CPU-only model (e.g. 3B) *can* run but will be slow (seconds per reply) and compete with the database | ~US$44+ |

**AI recommendation.** Don't run Ollama on the same small instance as PostgreSQL and
Node.js. CPU inference on 2 vCPUs is slow and memory-hungry and will degrade chat and
discovery. Either keep `AI_PROVIDER=none` (curated suggestions still work), point
`OLLAMA_URL` at a separate machine (ideally with a GPU or ≥16 GB RAM, on a private
network/VPN only), or use a hosted model with `AI_PROVIDER=openai-compatible`. The
daily limits (`AI_DAILY_USER_LIMIT`, `AI_DAILY_GLOBAL_LIMIT`) cap hosted-API spend.

Use the **Mumbai (ap-south-1)** region for Indian users. Attach a **static IP**.

## 2. Prepare the server (Ubuntu 22.04/24.04)

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl git ufw
```

```bash
curl -fsSL https://get.docker.com | sudo sh && sudo usermod -aG docker $USER
```

Lightsail firewall (networking tab): allow **22 (your IP only), 80, 443**. Nothing else.
PostgreSQL, the API (4000) and Ollama (11434) are never published by the production
compose file.

Add 2 GB swap on small instances so builds don't run out of memory:

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 3. Configure and deploy

Point your domain's A record at the static IP, then:

```bash
git clone <repo-url> knot && cd knot
```

```bash
cp deploy/.env.production.example deploy/.env.production && chmod 600 deploy/.env.production
```

Edit `deploy/.env.production`: `DOMAIN`, `ACME_EMAIL`, a strong `POSTGRES_PASSWORD`
(`openssl rand -base64 32 | tr -d '/+='`), SMTP settings (required for verification and
password reset emails), and AI settings.

```bash
./deploy/deploy.sh
```

`deploy.sh` builds images, starts PostgreSQL, **takes a backup**, applies Prisma
migrations with `migrate deploy` (never `migrate dev`/`reset` in production), loads
reference data (no fake members in production), starts the API and Caddy, and waits for
`/readyz`. Caddy obtains and renews HTTPS certificates automatically.

Create the first admin after signing up through the site:

```bash
docker compose -f deploy/docker-compose.prod.yml --env-file deploy/.env.production exec postgres psql -U knot -d knot -c "UPDATE \"User\" SET role='ADMIN' WHERE email='you@example.com';"
```

## 4. Operations

**Health checks.** `GET /healthz` (process up) and `GET /readyz` (database reachable). The
API container has a Docker `HEALTHCHECK`; all services use `restart: unless-stopped` and
memory limits.

**Logs.** Structured JSON (pino for the API, Caddy JSON access logs), rotated at 10 MB × 5.
Request bodies, cookies and message text are never logged.

```bash
docker compose -f deploy/docker-compose.prod.yml --env-file deploy/.env.production logs -f api
```

**Backups.** Daily at 02:30 UTC via cron, keeping 14 local copies. Each backup is a pair with
one timestamp: `knot-<time>.dump` (PostgreSQL) and `knot-<time>-uploads.tar.gz` (every photo,
video, poster and chat image). Restoring a database without its media leaves profiles pointing
at missing files, so always keep the pair together. Copy them off the instance (S3 or Lightsail
object storage) — a local-only backup does not survive losing the instance. Also enable
Lightsail **automatic snapshots** for the instance.

```bash
(crontab -l 2>/dev/null; echo "30 2 * * * $PWD/deploy/backup.sh >> $HOME/knot-backup.log 2>&1") | crontab -
```

**Restore** (stops the API, replaces the database, restores the matching uploads archive if it
sits next to the dump, restarts):

```bash
./deploy/restore.sh deploy/backups/knot-YYYYMMDDTHHMMSSZ.dump
```

Test a restore on a scratch instance at least once before launch.

**Account purge.** Accounts deleted more than 30 days ago are removed (including photos) by:

```bash
docker compose -f deploy/docker-compose.prod.yml --env-file deploy/.env.production exec api node dist/scripts/purgeDeleted.js
```

Schedule it daily with cron next to the backup job.

**Uploads.** Photos and videos live in the `uploads` Docker volume. `backup.sh` archives it
alongside each database dump; with 50 MB videos the archives grow quickly, so watch disk space
and prefer moving uploads to S3/Lightsail object storage as the member count grows.

## 5. Cost control and monitoring

- **Billing alerts:** AWS Billing → Budgets → create a monthly cost budget (e.g. ₹2,500)
  with email alerts at 80% and 100% of actual and forecasted spend.
- **Lightsail metric alarms:** CPU utilisation > 70% for 10 min, and status-check failures,
  notifying your email/SMS.
- **Disk:** `df -h` weekly or a cron alert when > 80%. Docker images: `docker system prune`
  after deployments.
- **AI spend:** keep the daily limits low until usage is understood. `AIUsage` records
  per-request metadata for monitoring.

## 6. Scaling up

1. **Bigger bundle:** create a snapshot → "Create new instance from snapshot" with a larger
   plan → move the static IP → verify → delete the old instance.
2. **Managed database:** move PostgreSQL to a Lightsail managed database (automatic
   backups, point-in-time restore) and set `DATABASE_URL` accordingly; remove the
   `postgres` service.
3. **Object storage:** move uploads to S3-compatible storage behind signed URLs.
4. **Multiple API instances:** add the Socket.IO Redis adapter and a shared rate-limit
   store before running more than one API container.

## 7. Before public launch

- Have the Terms, Privacy Policy and Community Guidelines reviewed by a lawyer (they're
  labelled as drafts in the app).
- Replace the placeholder support addresses on the Contact page.
- Configure SMTP with SPF/DKIM for the sending domain.
- Decide on a stronger age-assurance process if required in your jurisdiction (today:
  self-declared DOB + server age check + report-driven review).
- Add photo moderation (automated or manual) — photos are approved on upload in the MVP
  and can be rejected by moderators.
