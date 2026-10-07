# Social Media Chat Agent — n8n Workflow

An n8n chat-based agent that reads a blog post URL, drafts promotional posts for **LinkedIn, X (Twitter), Facebook, and Instagram**, gets your approval, and publishes them — all from a single chat interface.

---

## Architecture

```
User ──► n8n Chat Trigger ──► Gemini LLM Agent ──► Validate Output (Code Node)
                                   │                        │
                            ┌──────┘                        │
                            ▼                               ▼
                   Fetch Blog Content               ┌─ Is Approved? ─┐
                   Fetch Blog Meta                  │                │
                                              YES ──┤           NO ──┤
                                                    ▼                ▼
                                        Publish to 4 platforms   Reply to User
                                        Log to Google Sheets
```

### Nodes Overview

| Node | Purpose |
|---|---|
| **When chat message received** | Webhook chat trigger (entry point) |
| **Social Media Agent** | LangChain agent powered by Gemini 3.5 Flash Lite |
| **Fetch Blog Content / Meta** | HTTP tools that scrape blog text and `og:image` meta |
| **Structured Output Parser** | Enforces JSON schema on LLM output |
| **Validate Output** | Code node — security gate, text cleanup, anti-code-gen guard |
| **Is Approved?** | Routes `approved` status to publish, else replies to user |
| **Publish to LinkedIn / Facebook / Instagram / X** | Platform API calls |
| **Log \<Platform\>** | Appends publish status to Google Sheets |
| **Build Summary Reply** | Aggregates publish results into a single chat reply |

---

## Security Hardening

This workflow includes multiple layers of security:

### 1. Deterministic Approval Gate (Anti Prompt-Injection)
The `Validate Output` code node tracks session state using n8n's `$getWorkflowStaticData('global')`. Even if the LLM is tricked into returning `status: "approved"`:
- **First-turn approval is impossible** — `turnCount` must be > 1
- **Previous turn must have been `draft_ready`** — no skipping the review step
- Violations are silently downgraded to `draft_ready`

### 2. Anti-Code Generation Hard Gate
Two layers prevent the agent from ever generating code:
- **System prompt**: Absolute rule at the top of the prompt explicitly banning all code output
- **Regex scanner**: The `containsCode()` function in Validate Output scans `message_to_user` and all post texts for code patterns (code blocks, `<script>` tags, function declarations, SQL, shell commands, etc.). If any match is found, all output is wiped and the user gets a refusal message

### 3. Content Sanitization
All output text is cleaned:
- Literal `\n` → real line breaks
- Markdown symbols stripped (`**`, `#`, etc.)
- Emoji removed
- Hashtags removed
- Trailing whitespace and excess blank lines collapsed

---

## Prerequisites

| Service | What You Need |
|---|---|
| **Docker** | Docker Engine 20+ and Docker Compose v2+ |
| **Google Gemini API** | API key from [AI Studio](https://aistudio.google.com/apikey) |
| **LinkedIn** | OAuth2 app with `w_member_social` scope |
| **Facebook / Instagram** | Facebook App with Page publish permissions and Instagram Business account |
| **X (Twitter)** | OAuth2 app with tweet write access |
| **Google Sheets** *(optional)* | OAuth2 credentials for logging publishes |

---

## Quick Start

### 1. Clone and configure

```bash
git clone <your-repo-url> n8nworkflow
cd n8nworkflow

# Create your .env from the template
cp .env.example .env

# Generate a secure encryption key
openssl rand -hex 32
# Paste the output as N8N_ENCRYPTION_KEY in .env
```

Edit `.env` and fill in:
```env
N8N_ENCRYPTION_KEY=<your-generated-key>
N8N_GEMINI_API_KEY=<your-gemini-api-key>
```

### 2. Start n8n

```bash
docker compose up -d
```

Docker will automatically create the `n8n_data` named volume for persistent storage. Your workflows, credentials, and execution history are stored there.

### 3. Import the workflow

1. Open http://localhost:5678 in your browser
2. Go to **Workflows → Import from File**
3. Select `Social Media Chat Agent (fixed) (4).json`
4. Follow the **Credential & Account ID Setup Guide** below
5. Toggle the workflow **Active**

---

## 🔑 Credential & Account ID Setup Guide

To connect your social media accounts and AI models, you must configure credentials and IDs in **two places**:

1. **Inside the `Validate Output` Node** (for Account/Page IDs)
2. **Inside n8n Credentials Menu** (for API authentication)

---

### 1. Configure Account IDs (In `Validate Output` Node)

Open the `Validate Output` code node in n8n. At the very top of the script, update the `CONFIG` object:

```javascript
// ===== SETTINGS: Fill in your account details =====
const LIVE = {
  linkedin: true,   // Set to false for dry-run testing
  facebook: true,
  instagram: true,
  twitter: true,
};
const CONFIG = {
  linkedin_author: 'urn:li:person:YOUR_LINKEDIN_AUTHOR_ID', // Or 'urn:li:organization:YOUR_ORG_ID'
  fb_page_id: 'YOUR_FACEBOOK_PAGE_ID',
  ig_user_id: 'YOUR_INSTAGRAM_USER_ID',
};
// =============================================
```

#### How to find these IDs:

* **LinkedIn Author URN (`linkedin_author`)**:
  - For personal profiles: `urn:li:person:XXXXX` (Find your person ID using LinkedIn API `/v2/me` or inspecting API requests).
  - For company pages: `urn:li:organization:XXXXX` (Find your organization ID in your LinkedIn Page URL: `linkedin.com/company/12345678`).

* **Facebook Page ID (`fb_page_id`)**:
  - Open your Facebook Page → click **About** → scroll down to find **Page ID** (a numeric string like `132166854372449`).

* **Instagram User ID (`ig_user_id`)**:
  - Requires an **Instagram Professional / Business Account** linked to your Facebook Page.
  - Query Meta Graph API: `GET https://graph.facebook.com/v23.0/me/accounts?fields=instagram_business_account` to get your Instagram Business Account ID.

---

### 2. Configure API Credentials (n8n UI)

#### 🔵 A. LinkedIn Setup
1. In n8n, go to **Credentials → New → LinkedIn OAuth2 API**.
2. Go to [LinkedIn Developer Portal](https://www.linkedin.com/developers/) and create an app.
3. Under **Auth Settings**, add the OAuth2 Redirect URL provided by n8n.
4. Request `w_member_social` scope (for personal profile posts) or `w_organization_social` scope (for company pages).
5. Copy **Client ID** and **Client Secret** into n8n and click **Connect**.
6. Attach this credential to the **`Publish to LinkedIn`** node.

#### 📘 B. Facebook Page Setup
1. In n8n, go to **Credentials → New → Header Auth** (or **Query Auth**).
2. Go to [Meta for Developers](https://developers.facebook.com/) and create a Facebook App.
3. Generate a **Page Access Token** with `pages_manage_posts` and `pages_read_engagement` permissions.
4. Set parameter name `Authorization` with value `Bearer YOUR_PAGE_ACCESS_TOKEN`.
5. Attach this credential to the **`Publish to Facebook`** node.

#### 📸 C. Instagram Setup
1. Open the **`Create Instagram Media`** and **`Publish to Instagram`** HTTP nodes in n8n.
2. Select **Query Auth** (or **Header Auth**) with parameter `access_token` set to your **Meta Access Token** (same token used for Facebook Page if it has Instagram management permissions).

#### 🐤 D. X (Twitter) Setup
1. In n8n, go to **Credentials → New → X (Twitter) OAuth2 API**.
2. Go to [X Developer Portal](https://developer.x.com/) and create a Project & App.
3. Enable **OAuth 2.0 Settings** with **Read and Write** permissions and **Web App** client type.
4. Set Callback URL in X Developer Portal to your n8n redirect URL (`http://<your-n8n-host>/rest/oauth2-credential/callback`).
5. Copy **Client ID** and **Client Secret** into n8n and authorize.
6. Attach this credential to the **`Publish to X (Twitter)`** node.

#### 🤖 E. Google Gemini AI Setup
1. Go to [Google AI Studio](https://aistudio.google.com/app/apikey) and generate an API key.
2. In n8n, go to **Credentials → New → Google Gemini(PaLM) Api**.
3. Enter your API Key.
4. Attach this credential to the **`Google Gemini Chat Model`** node.

#### 📊 F. Google Sheets Logging Setup (Optional)
1. In n8n, create a **Google Sheets OAuth2 API** credential.
2. Open nodes: **`Log LinkedIn`**, **`Log Facebook`**, **`Log Instagram`**, and **`Log X`**.
3. In each node's parameters, replace `YOUR_SHEET_ID` with your Google Spreadsheet ID (found in your spreadsheet URL: `https://docs.google.com/spreadsheets/d/<YOUR_SHEET_ID>/edit`).

---

## Docker Compose Details

### Persistent Volume

```yaml
volumes:
  n8n_data:
    driver: local
```

The named volume `n8n_data` is mounted at `/home/node/.n8n` inside the container. This stores:
- Workflows
- Credentials (encrypted with `N8N_ENCRYPTION_KEY`)
- Execution history
- SQLite database

The volume persists across `docker compose down` / `up` cycles. It is only removed if you explicitly run `docker volume rm`.

### Managing the Volume

```bash
# Check volume exists
docker volume ls | grep n8n_data

# Inspect volume details
docker volume inspect n8nworkflow_n8n_data

# Backup (while container is stopped)
docker compose down
docker run --rm -v n8nworkflow_n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n_backup_$(date +%Y%m%d).tar.gz -C /data .

# Restore
docker run --rm -v n8nworkflow_n8n_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/n8n_backup_YYYYMMDD.tar.gz -C /data
docker compose up -d
```

### Common Commands

```bash
# Start in background
docker compose up -d

# View logs
docker compose logs -f n8n

# Restart
docker compose restart

# Stop (data is preserved)
docker compose down

# Stop and DELETE all data (⚠️ destructive)
docker compose down -v
```

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `N8N_ENCRYPTION_KEY` | **Yes** | — | Encrypts stored credentials. Generate with `openssl rand -hex 32`. Never change after first run. |
| `N8N_HOST` | No | `localhost` | Hostname for n8n |
| `N8N_PORT` | No | `5678` | Port to expose |
| `N8N_PROTOCOL` | No | `http` | `http` or `https` |
| `WEBHOOK_URL` | No | `http://localhost:5678/` | Public URL for webhooks (set this if behind a reverse proxy) |
| `GENERIC_TIMEZONE` | No | `Asia/Kolkata` | Timezone for cron and timestamps |
| `N8N_GEMINI_API_KEY` | No | — | Gemini API key (can also be set inside n8n credentials UI) |

---

## File Structure

```
n8nworkflow/
├── Social Media Chat Agent (fixed) (4).json   # n8n workflow (import this)
├── docker-compose.yml                          # Docker Compose config
├── .env.example                                # Environment variable template
├── .gitignore                                  # Keeps .env and volume data out of git
└── README.md                                   # Detailed documentation
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Container exits immediately | Check `docker compose logs n8n` — usually a missing `N8N_ENCRYPTION_KEY` |
| Webhook not reachable | Set `WEBHOOK_URL` to your public URL if behind a proxy |
| Credentials lost after restart | Ensure `N8N_ENCRYPTION_KEY` hasn't changed and `n8n_data` volume exists |
| Platform publish fails | Check credential expiry in n8n → Credentials; re-authenticate OAuth |
| Agent generates code | Both the prompt rule and the regex gate should block this. Check `sessionSecurity.codeBlocked` in execution data |

---

## License

Private — not for redistribution without permission.
