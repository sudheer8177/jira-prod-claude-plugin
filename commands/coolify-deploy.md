---
description: Deploy feature branches to Coolify dev environment and return preview URLs
argument-hint: <branch-name>  e.g. feat/PW-123-my-feature
allowed-tools: Bash
---

You are deploying PossibleWorks feature branches to the Coolify **dev** environment.

Input: $ARGUMENTS
Parse as a **single branch name** — e.g. `feat/PW-2252-replace-initiatives-with-tasks`.

The skill auto-detects which repos contain this branch and only deploys those.

All Coolify operations use **curl via Bash**. Never use MCP tools.

---

## ENVIRONMENT

```
BASE_URL = https://infra.erwrds.com
AUTH     = Authorization: Bearer $COOLIFY_ACCESS_TOKEN
```

All curl commands must include:
```
-H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN"
-H "Content-Type: application/json"
```

All JSON payloads **must be built with `jq`** to handle special characters safely:
```bash
jq -n --arg key "KEY" --arg val "VALUE" '{key: $key, value: $val, is_preview: false}'
```
Never construct JSON with raw string interpolation — values in SECRETS.md contain `+`, `=`, `/`, quotes and backslashes that will break raw JSON.

---

## REPO CONFIG TABLE

| Role | GitHub Repo | Port | App Prefix | Default Dev URL |
|------|-------------|------|------------|-----------------|
| Backend | PossibleWorks/pw-server-v3 | 3000 | BE | https://bev3-dev.lykkebook.com |
| Frontend | PossibleWorks/pw-react-client-v3 | 80 | FE | https://pw.lykkebook.com |
| AI Server | PossibleWorks/pw-ai-server | 3000 | AI | https://aibev3-dev.lykkebook.com |
| Notifications | PossibleWorks/pw-notifications | 80 | NOTIF | https://dev-notifications.lykkebook.com |
| AI Cron | PossibleWorks/ai-cron-server | 3000 | AIC | (internal only) |
| Cron Jobs | PossibleWorks/pw-cron-jobs | 3000 | CRON | (internal only) |

**App naming:** `<PREFIX>-<branch-slug>` where slug = branch with `/` replaced by `-`.
Example: `feat/PW-123-my-feature` → slug = `feat-PW-123-my-feature` → app name = `BE-feat-PW-123-my-feature`

If the resulting app name exceeds 50 characters, truncate the slug from the right to fit.

**FQDN:** `https://<app-name>.erwrds.com`

---

## REFERENCE IDs

```
PROJECT_UUID    = cc8gcwkk8go4gsco4cskwkos
DEV_ENV_UUID    = tscgoggowo8cwwgko4gooo4k
SERVER_UUID     = f8w0gwg4coc80s4cg0cco4og
GITHUB_APP_UUID = xgwsk8w8440wswo4g0kk80k0
FRONTEND_PARENT = qosws0k84g44kkcooo08owgo   <- NEVER deploy/restart
BACKEND_PARENT  = m4wkosg48c0okgw4coos80wo    <- NEVER deploy/restart
```

---

## STEP A — Auto-detect which repos have this branch

For each of the 6 repos, check if the branch exists on GitHub:

```bash
gh api "repos/<GITHUB_REPO>/git/refs/heads/<BRANCH_NAME>" 2>&1
```

Note: branch name slashes are fine in this URL — GitHub API handles them correctly.

- Response contains `"ref"` → branch **EXISTS** → ACTIVE, will deploy
- Response contains `"Not Found"` or non-zero exit → branch **missing** → SKIPPED, use default dev URL

Run all 6 checks. Record ACTIVE vs SKIPPED per repo.

Display result before proceeding:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Branch: feat/PW-xxx-slug
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ Backend       — branch found, will deploy
  ✅ Frontend      — branch found, will deploy
  ⬜ AI Server     — branch not found, using default
  ⬜ Notifications — branch not found, using default
  ⬜ AI Cron       — branch not found, using default
  ⬜ Cron Jobs     — branch not found, using default
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## STEP B — Resolve FQDNs for all services

For each ACTIVE repo:
- `FQDN = https://<PREFIX-branch-slug>.erwrds.com`

For each SKIPPED repo:
- `FQDN = Default Dev URL` from the table above

Record these variables — used for all inter-service env var overrides:
```
BE_URL    = <resolved backend FQDN>
FE_URL    = <resolved frontend FQDN>
AI_URL    = <resolved AI server FQDN>
NOTIF_URL = <resolved notifications FQDN>
```

---

## STEP C — Check for existing Coolify apps

```bash
curl -s -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  https://infra.erwrds.com/api/v1/applications
```

For each ACTIVE repo, scan the JSON response for an app where BOTH match:
- `git_repository` = the repo's GitHub path (e.g. `PossibleWorks/pw-server-v3`)
- `git_branch` = the branch name

- Match found → **EXISTING** — record `uuid`, go to STEP D2 (skip creation, still PATCH dockerfile)
- No match → **NEW** — proceed to STEP D1

---

## STEP D — Create apps + set Dockerfile location

**Strict creation order: BE → AI → NOTIF → AIC → CRON → FE**

### D1 — Create new app (NEW apps only)

Build the payload with jq, then POST:

```bash
PAYLOAD=$(jq -n \
  --arg name "<PREFIX-branch-slug>" \
  --arg git_repository "<GitHub Repo Path>" \
  --arg git_branch "<branch-name>" \
  --arg ports_exposes "<port>" \
  '{
    name: $name,
    project_uuid: "cc8gcwkk8go4gsco4cskwkos",
    server_uuid: "f8w0gwg4coc80s4cg0cco4og",
    environment_name: "dev",
    github_app_uuid: "xgwsk8w8440wswo4g0kk80k0",
    git_repository: $git_repository,
    git_branch: $git_branch,
    build_pack: "dockerfile",
    dockerfile_location: "/Dockerfile",
    ports_exposes: $ports_exposes
  }')

curl -s -X POST https://infra.erwrds.com/api/v1/applications/private-github-app \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD"
```

Extract `uuid` from response: `echo "$RESPONSE" | jq -r '.uuid'`

### D2 — PATCH dockerfile_location (ALL apps — new and existing)

Immediately after creation (new), or right away (existing), PATCH to guarantee `/Dockerfile` is set — existing apps may have been created before this was enforced:

```bash
curl -s -X PATCH "https://infra.erwrds.com/api/v1/applications/<APP_UUID>" \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dockerfile_location": "/Dockerfile"}'
```

No user confirmation needed — fully automatic.

---

## STEP E — Set domains

For each ACTIVE app (new or existing):

```bash
curl -s -X PATCH "https://infra.erwrds.com/api/v1/applications/<APP_UUID>" \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg fqdn 'https://<app-name>.erwrds.com' '{fqdn: $fqdn}')"
```

---

## STEP F — Set env vars (two phases per app)

### Phase 1 — Base vars from ~/.claude/SECRETS.md

Read `~/.claude/SECRETS.md`. Find the section for this repo. Extract every `KEY=VALUE` line, stripping surrounding quotes and whitespace from both key and value.

First, GET the app's current env vars to find existing IDs:

```bash
EXISTING=$(curl -s -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  "https://infra.erwrds.com/api/v1/applications/<APP_UUID>/envs")
```

For each key-value pair from SECRETS.md:

```bash
# Check if key has an existing id:
EXISTING_ID=$(echo "$EXISTING" | jq -r --arg k "KEY" '.[] | select(.key == $k) | .id // empty')

if [ -n "$EXISTING_ID" ]; then
  # PATCH existing var (endpoint is /envs/<id>):
  curl -s -X PATCH "https://infra.erwrds.com/api/v1/applications/<APP_UUID>/envs/$EXISTING_ID" \
    -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg key "KEY" --arg val "VALUE" '{key: $key, value: $val}')"
else
  # POST new var:
  curl -s -X POST "https://infra.erwrds.com/api/v1/applications/<APP_UUID>/envs" \
    -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg key "KEY" --arg val "VALUE" '{key: $key, value: $val, is_preview: false}')"
fi
```

### Phase 2 — Inter-service URL overrides

Apply AFTER Phase 1 — same PATCH-if-exists / POST-if-new logic using FQDNs from STEP B.

Re-fetch existing IDs before this phase if needed.

**Backend:**
| Key | Value |
|-----|-------|
| `DOMAIN` | `BE_URL` |
| `CLIENT_HOST` | `FE_URL` |
| `NOTIFICATION_SERVER_URL` | `NOTIF_URL/api/notification` |

**Frontend:**
| Key | Value |
|-----|-------|
| `VITE_API_URL` | `BE_URL/api` |
| `VITE_SOCKET_URL` | `BE_URL` |
| `VITE_AI_API_URL` | `AI_URL/api` |
| `VITE_NOTICATION_URL` | `NOTIF_URL/api` |
| `VITE_ENV` | `QA` |

**AI Server:**
| Key | Value |
|-----|-------|
| `BACKEND_SERVER_URL` | `BE_URL/api` |

**AI Cron:**
| Key | Value |
|-----|-------|
| `BACKEND_SERVER_URL` | `BE_URL/api` |
| `CLIENT_HOST` | `FE_URL` |
| `NOTIFICATION_SERVER_URL` | `NOTIF_URL/api/notification` |

**Cron Jobs:**
| Key | Value |
|-----|-------|
| `BACKEND_SERVER_URL` | `BE_URL/api` |
| `CLIENT_HOST` | `FE_URL` |
| `NOTIFICATION_SERVER_URL` | `NOTIF_URL/api/notification` |
| `AI_API_BASE_URL` | `AI_URL` |

---

## STEP G — Deploy in order

Deploy ACTIVE apps only. Strict order — wait for each to reach `finished` before starting the next:

**BE → AI → NOTIF → AIC → CRON → FE**

```bash
DEPLOY_RESPONSE=$(curl -s \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  "https://infra.erwrds.com/api/v1/deploy?uuid=<APP_UUID>&force=true")

DEPLOYMENT_UUID=$(echo "$DEPLOY_RESPONSE" | jq -r '.deployment_uuid // .deploymentUuid // empty')
```

If `DEPLOYMENT_UUID` is empty, check the full response for errors before proceeding.

---

## STEP H — Poll until finished

Poll every 15 seconds. Use a Bash timeout of **600000ms** (10 min) on the polling block to avoid the default 2-minute cutoff.

```bash
for i in $(seq 1 30); do
  sleep 15
  STATUS=$(curl -s \
    -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
    "https://infra.erwrds.com/api/v1/deployments/<DEPLOYMENT_UUID>" \
    | jq -r '.status // empty')

  echo "Poll $i/30 — status: $STATUS"

  if [ "$STATUS" = "finished" ]; then
    echo "Deployment finished."
    break
  elif [ "$STATUS" = "failed" ] || [ "$STATUS" = "error" ]; then
    echo "Deployment FAILED."
    break
  fi
done
```

On `finished` — verify app health:

```bash
APP_STATUS=$(curl -s \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  "https://infra.erwrds.com/api/v1/applications/<APP_UUID>" \
  | jq -r '.status // empty')
```

Interpret `status` field:
- `running:healthy` → success
- `running:unhealthy` → acceptable for frontend (nginx has no health check)
- `exited:unhealthy` → **failure** — fetch logs

On `failed` / `error` / `exited:unhealthy` — fetch logs:

```bash
curl -s \
  -H "Authorization: Bearer $COOLIFY_ACCESS_TOKEN" \
  "https://infra.erwrds.com/api/v1/applications/<APP_UUID>/logs" \
  | jq -r '.logs // .'
```

Report the error in full. Do NOT retry. Do NOT delete the app. Ask user how to proceed.

---

## STEP I — Output preview URLs

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Coolify Preview Deployment
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Backend      → https://BE-<slug>.erwrds.com     [status]
  AI Server    → https://AI-<slug>.erwrds.com     [status]
  Notifications→ https://NOTIF-<slug>.erwrds.com  [status]
  AI Cron      → https://AIC-<slug>.erwrds.com    [status]
  Cron Jobs    → https://CRON-<slug>.erwrds.com   [status]
  Frontend     → https://FE-<slug>.erwrds.com     [status]

  (only shows repos that were actually deployed)

  Coolify dashboard:
  https://infra.erwrds.com/project/cc8gcwkk8go4gsco4cskwkos/environment/tscgoggowo8cwwgko4gooo4k
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## HARD RULES

- **NEVER touch prod** (`os4ok8oco488gg4c8s8kgc4k`) **or UAT** (`qoocsssso88g0ooss0ggkco8`)
- **NEVER restart/redeploy** `FRONTEND_PARENT` (`qosws0k84g44kkcooo08owgo`) or `BACKEND_PARENT` (`m4wkosg48c0okgw4coos80wo`)
- **Always** `POST /api/v1/applications/private-github-app` — never public endpoint
- **Deploy order is strict**: BE → AI → NOTIF → AIC → CRON → FE
- **Dockerfile**: set in POST body AND PATCH for every app (new and existing)
- **Never retry a failed deploy** — report and ask user
- **Never delete a failed app**
- **All JSON payloads via `jq`** — never raw string interpolation (special chars in secret values will break it)
- **Env var update**: GET existing → `PATCH /envs/<id>` if found, `POST /envs` if new
- **Base env vars from `~/.claude/SECRETS.md`** — Coolify API returns null for values
- **Inter-service URLs**: use deployed FQDN if branch active, default dev URL if skipped
- **Polling Bash timeout**: always set to 600000ms to avoid the 2-min default cutoff
- **All ops via curl/Bash** — no MCP tools
