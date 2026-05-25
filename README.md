# AutoDoc Pipeline

An n8n-based automation that converts Bitbucket pull requests into user-facing documentation guides — powered by Claude AI, delivered to Google Docs, with Slack approvals at every step.

The team member pastes one or more Bitbucket PR URLs into Slack (team communication platform). The system reads the actual code changes, uses Claude to understand what changed from a user perspective, generates a complete guide in plain English, and posts the Google Doc link back to Slack — in under 5 minutes, with zero writing required.

---

## How It Works

```
/createguide <bitbucket-pr-url(s)>  ←  typed in Slack
          │
          ▼
  Fetch PR data from Bitbucket
  (metadata · changed files · commits · raw diff)
          │
          ▼
  Claude analyses the PR
  "What did this change do for the user?"
          │
          ▼
  Gate 2 — Human reviews analysis in Slack
  Clicks "Approve — Generate Guide"
          │
          ▼
  Claude writes the full user guide
  (Overview · How to Use It · What You Can Do · Things to Know)
          │
          ▼
  Google Doc created in Documentation Drafts folder
          │
          ▼
  Slack notification with direct doc link
```

**Multiple PRs supported** — paste 2 or 3 URLs together if a feature spans multiple repos. The system groups them and generates a single unified guide.

**Context hints supported** — add free text alongside the URLs:
```
/createguide https://bitbucket.org/your-workspace/repo/pull-requests/123
This is the new bulk discount feature for enterprise accounts
```

---

## Prerequisites

| Service | What You Need |
|---|---|
| **n8n** | Cloud or self-hosted instance |
| **Bitbucket Cloud** | Workspace with repos; OAuth2 consumer for API access |
| **Anthropic** | API key for Claude claude-sonnet-4-6 |
| **Google Docs** | OAuth2 credential connected to a Google account |
| **Google Sheets** | One sheet for PR staging buffer (used by the cron path) |
| **Slack** | Incoming webhook URL; slash command `/createguide` configured |

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/khusroo7AutoDocs-AI-PR-Based-Documentation-Engine-.git
cd AutoDocs-AI-PR-Based-Documentation-Engine-
```

### 2. Set up credentials in n8n

Create the following credentials in your n8n instance:

| Credential Name | Type | Fields Required |
|---|---|---|
| `Bitbucket OAuth2` | OAuth2 API | Authorization URL, Token URL, Client ID, Client Secret |
| `Google Docs OAuth` | Google Docs OAuth2 API | Connect via Google account |
| `Google Sheets account` | Google Sheets OAuth2 API | Connect via Google account |

**Bitbucket OAuth2 settings:**
- Grant Type: `Authorization Code`
- Authorization URL: `https://bitbucket.org/site/oauth2/authorize`
- Access Token URL: `https://bitbucket.org/site/oauth2/access_token`
- Scope: `repository`
- Authentication: `Header`
- Callback URL to set in Bitbucket: `https://oauth.n8n.cloud/oauth2/callback` *(n8n Cloud)* or `https://your-instance/rest/oauth2-credential/callback` *(self-hosted)*

### 3. Create the Bitbucket OAuth consumer

In Bitbucket workspace settings → **OAuth consumers** → **Add consumer**:
- Callback URL: *(from n8n credential form above)*
- Permissions: `Repositories → Read`, `Pull requests → Read`

### 4. Set up the Slack slash command

In your Slack app settings:
- Create a slash command `/createguide`
- Request URL: `https://your-n8n-instance/webhook/slack-createguide-v2`
- Method: POST

### 5. Import the workflow

1. Open `workflow-template.json`
2. Replace all placeholder values (see table below)
3. Import into n8n via **Workflows → Import from file**
4. Update each node's credential references to match your n8n credential names

### 6. Configure placeholder values

Replace these in `workflow-template.json` before importing:

| Placeholder | Where | What to put |
|---|---|---|
| `YOUR_ANTHROPIC_API_KEY` | Claude nodes (headers) | Your Anthropic API key |
| `YOUR_SLACK_WEBHOOK_URL` | Gate 1, Gate 2, Gate 3 nodes | Your Slack incoming webhook URL |
| `YOUR_GOOGLE_DRIVE_FOLDER_ID` | Create Google Doc node | ID of your Documentation Drafts folder |
| `YOUR_BITBUCKET_WORKSPACE` | All Fetch nodes | Your Bitbucket workspace slug |
| `YOUR_GOOGLE_SHEETS_DOC_ID` | Google Sheets nodes | ID of your staging spreadsheet |

---

## Usage

In your Slack channel, type:

```
/createguide https://bitbucket.org/your-workspace/repo/pull-requests/123
```

For multiple PRs (feature spanning several repos):

```
/createguide https://bitbucket.org/your-workspace/repo-a/pull-requests/45
             https://bitbucket.org/your-workspace/repo-b/pull-requests/67
```

With a context hint:

```
/createguide https://bitbucket.org/your-workspace/repo/pull-requests/123
This is the new checkout redesign — focus on the tenant admin settings
```

The workflow will:
1. Post the Claude analysis to your `#user-guide` channel for review
2. Wait for you to click **"Approve — Generate Guide"**
3. Generate the full guide and post the Google Doc link

---

## Workflow Nodes (Manual Path)

| # | Node | Type | What it does |
|---|---|---|---|
| 1 | Slack /createguide Webhook | Webhook | Receives the slash command from Slack |
| 2 | Parse Slack Command | Code | Extracts PR URLs and context hint from message |
| 3 | Split PRs | Code | Creates one item per PR for sequential fetching |
| 4 | Fetch PR Metadata | HTTP Request | Calls Bitbucket API — title, description, author |
| 5 | Fetch Diffstat | HTTP Request | Calls Bitbucket API — changed files and line counts |
| 6 | Fetch Commits | HTTP Request | Calls Bitbucket API — commit messages |
| 7 | Fetch Diff | HTTP Request | Calls Bitbucket API — raw code diff (text) |
| 8 | Aggregate PR Data | Code | Merges all fetched data into a single pr_group object |
| 9 | Build Analysis Payload | Code | Assembles the prompt for Claude PR analysis |
| 10 | Claude — PR Analysis | HTTP Request | Calls Anthropic API — returns structured JSON analysis |
| 11 | Parse Analysis | Code | Extracts and parses Claude's JSON response |
| 12 | Gate 2 — Slack Notification | HTTP Request | Posts analysis summary to Slack with Approve button |
| 13 | Gate 2 — Wait for Approval | Wait | Pauses until a human clicks Approve in Slack |
| 14 | Build Guide Payload | Code | Assembles the guide writing prompt for Claude |
| 15 | Claude — Generate Guide | HTTP Request | Calls Anthropic API — returns full Markdown guide |
| 16 | Parse Guide | Code | Extracts guide text, sets document title |
| 17 | Create Google Doc Draft | Google Docs | Creates new doc in Documentation Drafts folder |
| 18 | Insert Guide Content | HTTP Request | Inserts guide text via Google Docs batchUpdate API |
| 19 | Mark Staging Rows Done | Google Sheets | Updates staging sheet status to done |
| 20 | Gate 3 — Draft Ready | HTTP Request | Posts final Slack notification with Google Doc link |

---

## AI Model

Both AI steps use **Claude claude-sonnet-4-6** via the Anthropic API (`https://api.anthropic.com/v1/messages`).

| Step | Purpose | Max Output |
|---|---|---|
| PR Analysis | Reads code diff, returns structured JSON on user impact | 2,048 tokens |
| Guide Generation | Writes full user-facing Markdown guide | 4,096 tokens |

The guide always follows this structure:
```
# Guide Title

## Overview
## How to Use It
## What You Can Do
## Things to Know
```

Rules enforced via system prompt: no technical terms, second person, clean Markdown only.

---

## Cost

Claude API pricing (claude-sonnet-4-6): $3/M input tokens · $15/M output tokens

| Scenario | Approx. Cost Per Run |
|---|---|
| Single small PR | ~$0.05 |
| Single medium PR | ~$0.07 |
| 3 PRs, large feature | ~$0.16 |

100 guides/month ≈ $5–15 in Claude API fees.