# 🎯 AI Job Matching Agent – MVP

An AI-powered job matching system where each user has a personal job agent that finds jobs posted within the last 24 hours, ranks them against their resume and preferences, and sends daily summaries via SMS or email.

## Architecture

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│   Next.js    │───▶│  Express API │───▶│   Supabase   │
│  Frontend    │◀───│  (Railway)   │◀───│  PostgreSQL  │
└─────────────┘    └──────┬───────┘    └──────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌────────┐ ┌─────────┐
        │  JSearch  │ │ OpenAI │ │ Twilio/ │
        │   API    │ │  GPT-4o│ │ Resend  │
        │ (jobs)   │ │  mini  │ │ (notify)│
        └──────────┘ └────────┘ └─────────┘
```

## Data Flow

```
User signs up → uploads resume → AI parses resume (once) →
structured profile saved → scheduled worker searches jobs →
jobs stored in DB → algorithm matches jobs to users →
AI summarizes top 3 → summary sent via SMS/email
```

## Cost Control

| Resource     | Monthly Cost |
|-------------|-------------|
| Supabase    | Free tier   |
| Railway     | ~$5         |
| AI (OpenAI) | <$1         |
| SMS/Email   | $1–5        |
| **Total**   | **$6–10**   |

AI is used ONLY for:
1. Resume extraction (once per user)
2. Summarizing top 3 matched jobs (3x daily per user)

All filtering and matching uses a scoring algorithm — no AI.

---

## Quick Start

### 1. Prerequisites

- Node.js 18+
- Supabase project (free tier)
- OpenAI API key
- Twilio or Resend account (for notifications)
- Optional: RapidAPI key for JSearch (job data)

### 2. Supabase Setup

1. Create a new Supabase project at [supabase.com](https://supabase.com)
2. Go to **SQL Editor** and run the schema:

```bash
# Copy contents of sql/schema.sql into the SQL Editor and execute
```

3. Copy your project URL, anon key, and service role key from **Settings → API**

### 3. Backend Setup

```bash
cd backend
cp .env.example .env
# Edit .env with your actual keys

npm install
npm run dev
```

The API starts on `http://localhost:3001`.

### 4. Test the Worker

```bash
# Run the job worker manually (uses sample jobs in dev mode)
npm run worker
```

### 5. Frontend

The React dashboard (`jobagent-dashboard.jsx`) connects to the API at `http://localhost:3001/api`. For production, update the `API` constant or use environment variables.

---

## API Endpoints

### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/signup` | Create account `{email, password, phone?}` |
| POST | `/api/auth/login` | Sign in `{email, password}` → `{user, token}` |

### Profile
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/profile` | Get user profile (auth required) |
| PATCH | `/api/profile` | Update profile fields |
| POST | `/api/profile/resume` | Submit resume text → AI extraction |
| POST | `/api/profile/resume/upload` | Upload .txt or .pdf file |

### Jobs
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/jobs` | List recent jobs `?limit=20&offset=0` |
| GET | `/api/jobs/search` | Search `?q=engineer&location=remote` |
| GET | `/api/jobs/:id` | Get single job |

### Matches
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/matches` | User's job matches with scores |
| GET | `/api/matches/stats` | Match statistics |

### Admin
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/admin/run-worker` | Trigger worker manually |
| GET | `/api/health` | Health check |

---

## Matching Algorithm

The scoring system runs entirely in code (no AI):

| Criteria | Points | Logic |
|----------|--------|-------|
| Role match | +50 | Job title contains user's target roles |
| Partial role | +25 | 70%+ role words found in description |
| Skills match | up to +30 | Scaled by % of skills found in description |
| Location match | +10 | Job location matches preferences |
| Remote match | +10 | Job is remote & user prefers remote |
| Salary match | +10 | Job salary ≥ 85% of user expectation |

**Threshold:** Only matches scoring ≥ 70 are kept.  
**Limit:** Top 3 per user per run.

---

## Worker Schedule

Runs 3x daily via `node-cron`:

| Time (UTC) | Purpose |
|------------|---------|
| 08:00 | Morning scan |
| 13:00 | Midday scan |
| 18:00 | Evening scan |

Each run:
1. Fetch new jobs from JSearch API
2. Store in database (dedup by URL)
3. For each user: score all recent jobs
4. AI summarize top 3 new matches
5. Send notification (email/SMS)
6. Mark matches as sent

---

## Deployment (Railway)

```bash
# Install Railway CLI
npm i -g @railway/cli

# Login and init
railway login
railway init

# Set environment variables
railway variables set SUPABASE_URL=...
railway variables set SUPABASE_SERVICE_KEY=...
railway variables set OPENAI_API_KEY=...
# ... set all vars from .env.example

# Deploy
railway up
```

The `railway.json` config handles build and health checks automatically.

---

## File Structure

```
jobagent/
├── sql/
│   └── schema.sql              # Supabase database schema
├── backend/
│   ├── server.js               # Express app + cron setup
│   ├── package.json
│   ├── railway.json            # Railway deployment config
│   ├── .env.example            # Environment template
│   ├── routes/
│   │   ├── auth.js             # Signup/login
│   │   ├── profile.js          # Profile + resume upload
│   │   ├── jobs.js             # Job browsing/search
│   │   └── matches.js          # Match history + stats
│   ├── services/
│   │   ├── ai.js               # OpenAI resume extraction + summarization
│   │   ├── jobSearch.js        # JSearch API integration
│   │   ├── matching.js         # Scoring algorithm
│   │   └── notifications.js    # Twilio SMS + Resend email
│   ├── workers/
│   │   └── jobWorker.js        # Main pipeline orchestrator
│   └── utils/
│       ├── supabase.js         # Database client
│       └── auth.js             # JWT middleware
└── frontend/
    └── jobagent-dashboard.jsx  # React dashboard (artifact)
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `SUPABASE_URL` | Yes | Supabase project URL |
| `SUPABASE_SERVICE_KEY` | Yes | Service role key (backend only) |
| `SUPABASE_ANON_KEY` | Yes | Anon key (for auth) |
| `JWT_SECRET` | Yes | 32+ char secret for JWT signing |
| `OPENAI_API_KEY` | Yes | OpenAI API key |
| `AI_MODEL` | No | Default: `gpt-4o-mini` |
| `RAPIDAPI_KEY` | No | For JSearch job API |
| `TWILIO_ACCOUNT_SID` | No* | Twilio SMS |
| `TWILIO_AUTH_TOKEN` | No* | Twilio SMS |
| `TWILIO_PHONE_NUMBER` | No* | Twilio sender number |
| `RESEND_API_KEY` | No* | Resend email |
| `RESEND_FROM_EMAIL` | No* | Resend sender address |

*At least one notification provider required.

---

## Future Improvements (Not Implemented)

- Vector embeddings for semantic job matching
- AI cover letter generation
- Resume improvement suggestions
- Weekly job market reports
- Personalized career insights
- Multi-provider job aggregation
