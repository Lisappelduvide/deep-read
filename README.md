# DeepRead

DeepRead is a self-hosted, account-scoped workspace for reading English articles. It combines a block-based article editor, a paginated reader with highlights and word lookup, a vocabulary notebook, and optional AI-assisted translation and analysis. Content is stored locally in SQLite; accounts are created by an administrator, with an optional shared guest account.

The application is a React single-page frontend served by an Express API. 

> This document describes behavior verified against the current source code. For the full production runbook, see [DEPLOYMENT.md](./DEPLOYMENT.md).

English | [简体中文](./README.zh-CN.md)

## Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Local Development](#local-development)
- [Production Deployment](#production-deployment)
- [Configuration](#configuration)
- [User Roles and Permissions](#user-roles-and-permissions)
- [Data Storage](#data-storage)
- [FAQ and Known Limitations](#faq-and-known-limitations)

## Features

| Area | What the code implements |
|---|---|
| Article library | Create and edit articles in a block editor (`text` / `image` blocks). Paste HTML, drag or paste images, import `.txt` files, and record source URL/author metadata. Browse, search by title or tag, sort, and tag articles (10 per page). |
| Reader | Paginates text at 1,800 characters per page, preserves image blocks, restores the last page per article, adjusts font size (12–32 px), and switches between light and dark themes. |
| Annotations | Click a word to look it up; select text to save highlights/underlines with notes; automatically emphasize words already in the vocabulary notebook. Highlights are aggregated in global and per-article views. |
| Vocabulary | Save words with definition, phonetic, part of speech, example, article context, and spaced-repetition scheduling fields. Search, sort, and group by article. Administrator-only Eudic sync endpoint is available. |
| Translation and analysis | Per-page translation through Baidu AI Text Translation and sentence-level analysis through OpenAI, Gemini, or a local fallback. Results are cached in SQLite and can be cleared per article. External API credentials are required. |
| Media | Uploads are converted to WebP with generated thumbnails, stored under per-account directories, and served only through ownership-checked routes. Remote image URLs are validated (http/https only, private addresses blocked, 10 MB limit). |
| Administration | Overview statistics, create/enable/disable/delete accounts, reset passwords, export account data, and toggle guest access. |
| Portability | Users can export their own data as JSON. Database backup, restore, import, and migration scripts are provided. |

## Tech Stack

| Layer | Components |
|---|---|
| Frontend | React 19, React Router 7, Vite 8, lucide-react, axios, oxlint |
| Backend | Node.js (deployment requires 22+), Express 5, better-sqlite3, sharp, dotenv, cors |
| Database | SQLite in WAL mode; separate Admin and Guest database files |
| Operations | PM2, Nginx, systemd, Ubuntu; no container image is provided |

## Project Structure

```text
deep-read/
├── src/                        # React application
│   ├── api/                    # Frontend API wrappers
│   ├── components/             # article/, reader/, and shared UI components
│   ├── pages/                  # Route pages (library, reader, vocabulary, admin, ...)
│   ├── routing/access.js       # Role-based navigation and redirects
│   └── styles/                 # Global and page styles
├── backend/
│   ├── server.js               # Express API, auth, media routes, static SPA hosting
│   ├── db.js                   # SQLite connections, schema, and migration helpers
│   ├── scripts/                # migrate, bootstrap administrator, backup, restore
│   └── .env.example            # Development configuration template
├── deploy/
│   ├── install.sh              # First-time production installation
│   ├── update.sh               # Production update
│   ├── backup-db.sh / restore-db.sh / import-db.sh / migrate.sh / check.sh
│   ├── migrations/             # Account-ownership SQL migrations
│   ├── ecosystem.config.cjs    # PM2 process definition
│   └── nginx.conf              # Reverse proxy configuration
├── data/                       # Local SQLite databases (gitignored)
├── uploads/                    # Local media files (gitignored)
├── .env.example                # Production-oriented configuration template
├── package.json                # Frontend scripts and shared dependencies
├── vite.config.js              # Dev server proxy for /api and /uploads
└── DEPLOYMENT.md               # Detailed deployment and operations guide
```

## Requirements

- **Node.js 22 or newer.** `deploy/install.sh` rejects older versions, and Vite 8 requires Node 22.13+ or 24+.
- **npm**, **Git**, and the SQLite CLI (`sqlite3`) for the deployment and migration scripts.
- For production: an Ubuntu host with `sudo`, PM2, Nginx, and systemd. `deploy/install.sh` installs the system packages and PM2 itself and expects a non-root `ubuntu` user.
- Optional external accounts for translation, AI analysis, dictionary lookups, and Eudic sync. These features are unavailable without their respective credentials.

## Installation

```bash
git clone https://github.com/After2thougt/deep-read.git
cd deep-read

npm ci
(cd backend && npm ci)

cp backend/.env.example .env
```

Edit `.env` before starting. The server loads the repository-root `.env` first and then `backend/.env`; values already set are not overwritten. See [Configuration](#configuration) for the variables actually read by the code.

Prepare the schema and the first administrator:

```bash
npm --prefix backend run db:migrate
npm --prefix backend run db:bootstrap-administrator
```

`db:migrate` applies the account-ownership migrations to the configured Admin database. `db:bootstrap-administrator` is interactive and asks for a username and password; it refuses to run if an administrator already exists. The `AUTH_USERNAME` / `AUTH_PASSWORD` pair in `.env` creates a **regular** account on startup, not an administrator.

## Local Development

Run the API and the Vite dev server in separate terminals:

```bash
# Terminal 1 — API on http://127.0.0.1:3000
npm --prefix backend run dev
```

```bash
# Terminal 2 — frontend on http://localhost:5173
npm run dev
```

Vite proxies `/api` and `/uploads` to `http://127.0.0.1:3000`. Useful checks:

```bash
curl http://127.0.0.1:3000/api/health    # {"status":"ok"}
npm run lint                             # oxlint
npm run build                            # production build into dist/
npm run preview                          # preview the built frontend
```

Use `npm start` instead of the watch command to run the API without auto-restart. The API binds to `127.0.0.1:3000` by default and serves `dist/` with an SPA fallback in production mode.

## Production Deployment

All deployment scripts must be run as a non-root user from `/opt/deepread/app`. Paths are defined in the scripts and are not configurable via command-line flags.

```bash
# First-time installation (Ubuntu host)
bash deploy/install.sh

# Apply an update from a branch (default: main)
bash deploy/update.sh [branch]

# Operations
bash deploy/check.sh
bash deploy/backup-db.sh
bash deploy/restore-db.sh [backup-file]
bash deploy/migrate.sh

# Package-script equivalents
npm run db:backup
npm run db:restore -- /path/to/backup.db
```

`install.sh` installs system packages and PM2, configures Nginx and systemd, prepares `.env`, installs both dependency trees with `npm ci`, backs up the database, runs migrations, builds the frontend, starts the `deepread` PM2 process, and performs health checks. It **requires the Admin database to already exist** at `/data/deepread/app.db`; it will not create or replace it.

`update.sh` verifies the environment, requires a clean working tree, pulls with `git pull --ff-only`, reinstalls dependencies, backs up the database, runs migrations, rebuilds the frontend, restarts PM2 with the updated environment, and checks health endpoints. There is no automatic rollback: restore a database backup manually if an update fails after migration.

A `deepread-deploy.timer` unit may be installed to check GitHub every five minutes and invoke `update.sh main`. Check `systemctl status deepread-deploy.timer` on the host; if it is disabled or stopped, run `update.sh` manually.

| Runtime path | Purpose |
|---|---|
| `/opt/deepread/app` | Application checkout |
| `/data/deepread/app.db` | Admin database |
| `/data/deepread/guest.db` | Guest database |
| `/data/deepread/backups` | Database backups (30 most recent kept) |
| `/data/deepread/uploads` | Uploaded media |
| `/data/deepread/logs` | PM2 logs (`deepread-out.log`, `deepread-error.log`) |

Nginx listens on port 80 and proxies `/api/`, `/uploads/`, and all SPA routes to `127.0.0.1:3000`. PM2 runs the process named `deepread` from `deploy/ecosystem.config.cjs`.

## Configuration

No secret values are included here. Copy a template and fill in your own values.

| Variable | Purpose | Required / default |
|---|---|---|
| `NODE_ENV` | Runtime mode; enables production guards and restricts CORS | `development` or `production` |
| `HOST`, `PORT` | API bind address and port | `127.0.0.1`, `3000` |
| `DATABASE_PATH` | Admin SQLite database | Defaults to `<repo>/data/app.db`; production scripts require `/data/deepread/app.db` |
| `GUEST_DATABASE_PATH` | Guest SQLite database | Defaults to `guest.db` beside the Admin DB; production scripts require `/data/deepread/guest.db` |
| `AUTH_USERNAME`, `AUTH_PASSWORD` | Bootstrap a regular account on startup | Required when `NODE_ENV=production` |
| `AUTH_SESSION_SECRET` | Signs session cookies | Required when `NODE_ENV=production` |
| `GUEST_USERNAME`, `GUEST_PASSWORD` | Shared guest login | Needed to enable guest sign-in |
| `CORS_ORIGIN` | Allowed cross-origin origin | Empty by default; cross-origin requests are disabled in production |
| `UPLOADS_PATH` | Media root directory | Defaults to `<repo>/uploads` |
| `BAIDU_TRANSLATE_APPID`, `BAIDU_AI_TRANSLATE_API_KEY` | Translation service | Needed for `/api/translate` |
| `ANALYSIS_PROVIDER`, `ANALYSIS_PROMPT_VERSION` | Analysis provider (`openai`, `gemini`, `fallback`) and prompt version | Default prompt version `v4` |
| `OPENAI_API_KEY`, `OPENAI_API_BASE`, `OPENAI_API_MODE`, `OPENAI_MODEL` | OpenAI-compatible analysis | Optional if another provider is configured |
| `GOOGLE_GEMINI_API_KEY`, `GOOGLE_GEMINI_MODEL` | Gemini analysis | Optional if another provider is configured |
| `MERRIAM_WEBSTER_LEARNERS_KEY` | Dictionary lookups | Optional |
| `EUDIC_TOKEN` | Eudic vocabulary sync | Optional |
| `IMAGE_DOWNLOAD_PROXY` | Proxy for downloading remote images | Optional |
| `LEGACY_OWNER` | Legacy account used as owner by the ownership migration script | Defaults to `carolc` in `deploy/migrate.sh` |

Template entries `ADMIN_KEY`, `TRANSLATE_PROVIDER`, and `TRANSLATE_API_KEY` are **not read** by the current server. The translation route always uses Baidu; Tencent configuration is present in the code but is not selected by that route.

Sessions are stored in process memory in a signed, `HttpOnly`, `SameSite=Lax` cookie (`deepread_session`) with a 14-day lifetime. They do not survive an API restart and are not shared between processes. Login attempts are rate limited to 10 per 15 minutes per IP.

## User Roles and Permissions

| Role | How it is created | Capabilities |
|---|---|---|
| Administrator | Interactive `db:bootstrap-administrator` CLI | Admin overview, create/enable/disable/reset/delete/export accounts, toggle guest access, Eudic sync, plus normal reading features. The UI restricts admins to Home and User Management. |
| User | Created by an administrator through `/api/admin/users` | Own articles, tags, vocabulary, highlights, theme/font settings, self-service password change, and JSON export. |
| Guest | Optional shared login from the separate Guest database when guest access is enabled | Access to the shared guest data space; no self-service password change. Disabling guest access invalidates all guest sessions. |

Ownership filtering is enforced in the backend: regular and administrator requests are scoped by their account ID and database, while guest sessions use the Guest database. Frontend role checks only control navigation; the `/api/admin/*` endpoints enforce administrator access independently. Passwords are stored as salted `scrypt` hashes. Self-service password changes require at least 8 characters; administrator-created or reset passwords require at least 12.

## Data Storage

- **Databases:** SQLite with WAL mode and foreign keys enabled. The Admin database holds accounts and account-owned content; the Guest database holds the shared guest space. The required schema and ownership migrations are checked at startup.
- **Tables:** `auth_users`, `articles`, `article_blocks`, `tags`, `article_tags`, `vocabulary`, `article_analyses`, `article_translation_cache`, `guest_access_settings`, and `migrations`. Ownership columns and constraints are applied by the two migrations in `deploy/migrations/`.
- **Media:** `UPLOADS_PATH` (default `<repo>/uploads`) contains `users/<userId>/` and `guest/<userId>/` trees, with temporary uploads and per-article WebP images plus thumbnails. Media reads require a valid session, an owned path, and a database reference.
- **Browser state:** theme and reader font size are stored in `localStorage`; drafts and per-article reading state are namespaced by database and account ID.
- **Backups:** `deploy/backup-db.sh` takes a WAL-safe snapshot with an integrity check and keeps the latest 30 files. `deploy/restore-db.sh` stops PM2, creates a pre-restore backup, replaces the database, clears WAL/SHM files, and reruns health checks.

## FAQ and Known Limitations

**Startup fails with "Admin database schema is not ready".** Back up the Admin database, then run `npm --prefix backend run db:migrate`. The API refuses to listen until both ownership migrations and the `role` / `is_active` columns are present.

**The `AUTH_USERNAME` account cannot open administration pages.** It is a regular account. Create an administrator with `npm --prefix backend run db:bootstrap-administrator`.

**Guest login is unavailable.** Set `GUEST_USERNAME` and `GUEST_PASSWORD`, ensure the guest account exists, and verify that guest access is enabled in the admin user-management page.

**Translation or analysis does nothing.** These features call external services. Configure the Baidu keys for translation and one of the OpenAI/Gemini providers for analysis. Results are cached per article and page.

**An update failed.** Deployment scripts do not roll back automatically. Restore the latest backup with `bash deploy/restore-db.sh [backup-file]`, then re-run `bash deploy/update.sh main`.

**Current limitations.** There is no self-registration or password recovery flow; the "Forgot password?" link is not wired to a route. This repository has no URL fetching or article-extraction endpoint — source URLs are stored as metadata only. There is no article-level text-to-speech; only individual word pronunciation uses the browser speech API. No test script is defined in `package.json`; use `npm run lint` and `npm run build` for verification.
