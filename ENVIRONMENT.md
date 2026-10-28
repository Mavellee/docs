# Environment variables (API)

Copy `server/.env.example` to `server/.env` for development. In production these come
from `deploy/.env.production` (see [DEPLOYMENT.md](DEPLOYMENT.md)). The web app has **no**
secrets; it talks only to its own origin.

| Variable | Default | Purpose |
|---|---|---|
| `NODE_ENV` | `development` | `production` enables secure cookies and skips dev seed data |
| `PORT` | `4000` | API port |
| `APP_ORIGIN` | `http://localhost:5180` | Public web origin. Used for CORS, the CSRF origin check and links in emails |
| `DATABASE_URL` | — | PostgreSQL connection string (required) |
| `TEST_DATABASE_URL` | `…/knot_test` | Used only by `npm test` |
| `SESSION_TTL_DAYS` | `30` | Session lifetime |
| `COOKIE_SECURE` | `false` | Force `Secure` cookies (always on in production) |
| `TRUST_PROXY` | `0` | Number of proxies in front of the API (`1` behind Caddy) |
| `UPLOAD_DIR` | `./uploads` | Where photos are stored (a Docker volume in production) |
| `MAX_UPLOAD_MB` | `5` | Upload size limit |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM` | — | Outgoing email. Without `SMTP_HOST`, emails are logged to the console |
| `AI_PROVIDER` | `none` | `none`, `ollama` or `openai-compatible` |
| `AI_MODEL` | `llama3.2:3b` | Model name sent to the provider |
| `OLLAMA_URL` | `http://localhost:11434` | Ollama base URL (keep it private) |
| `AI_API_URL`, `AI_API_KEY` | — | Hosted OpenAI-compatible endpoint (server-side only) |
| `AI_TIMEOUT_MS` | `12000` | Per-request timeout before falling back |
| `AI_MAX_OUTPUT_CHARS` | `600` | Hard cap on returned text |
| `AI_MAX_CONCURRENCY` | `2` | Requests beyond this fall back to curated content |
| `AI_DAILY_USER_LIMIT` | `20` | AI calls per member per UTC day |
| `AI_DAILY_GLOBAL_LIMIT` | `2000` | AI calls across all members per UTC day |
| `COMPAT_WEIGHT_*` | 5/3/3/2/4 | Weights for relationship goals, communication, lifestyle, interests, values (0–10) |
| `LOG_LEVEL` | `info` | pino log level |
