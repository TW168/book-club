# Path of Awakening Club — CLAUDE.md

## Project Overview

**Path of Awakening Club** is a web-based book discussion platform built around Shih Cheng Yen's *The Path of Awakening*. Members paste book passages (hidden from public view), share their personal thoughts, and receive AI-generated interpretations powered by Qwen 2.5 via Hugging Face. Readers engage through comments using display names — no registration required.

The app serves an existing book club community. Every member already owns the book. The book text is stored privately for LLM context only and never displayed publicly. Posts reference chapters/sections so readers follow along in their own copy.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | FastAPI (Python 3.11+) |
| Frontend | Bootstrap 5, Jinja2 templates |
| Database | PostgreSQL (shared instance, dedicated `awakening_club` database) |
| LLM | Qwen 2.5 via Hugging Face Inference API (fallback: Ollama local) |
| Package Manager | UV |
| Deployment | Docker → Dokploy on homelab |
| ORM | SQLAlchemy 2.0 + Alembic migrations |

---

## Design Language — Buddhist Peaceful Aesthetic

### Color Palette

```
--color-bg:           #FAF6F0    /* Warm ivory / parchment */
--color-bg-alt:       #F0EBE3    /* Soft sand */
--color-surface:      #FFFFFF    /* Clean white cards */
--color-primary:      #8B7355    /* Muted gold-brown / sandalwood */
--color-primary-dark: #6B563F    /* Deeper sandalwood */
--color-accent:       #7A8B6F    /* Sage green / bamboo */
--color-accent-light: #D4DDD0    /* Light sage wash */
--color-calm:         #9B8EC4    /* Soft meditation purple */
--color-calm-light:   #E8E4F0    /* Lavender mist */
--color-text:         #3D3529    /* Deep warm brown */
--color-text-light:   #7A7062    /* Muted brown */
--color-border:       #E0D8CE    /* Gentle border */
--color-danger:       #C47A6A    /* Soft terracotta for warnings */
```

### Typography

- **Headings**: Crimson Pro (serif, contemplative, readable)
- **Body**: Source Sans 3 (clean, bilingual-friendly)
- **Chinese fallback**: Noto Serif TC for Traditional Chinese rendering
- **Monospace**: Not needed (no code display)

### UI Principles

- Generous whitespace — let content breathe like a meditation space
- No harsh borders — use soft shadows and subtle dividers
- Rounded corners (8px–12px) — gentle, approachable
- Minimal animations — slow, intentional fades only (300ms+)
- Lotus or Dharma wheel as subtle decorative motif (SVG, not heavy imagery)
- Side-by-side layout for "My Thoughts" vs "AI Interpretation" on desktop
- Stacked layout on mobile (thoughts first, AI below)
- Chapter navigation feels like turning pages, not clicking buttons

---

## Data Model

### Tables

#### `chapters`
```
id              SERIAL PRIMARY KEY
chapter_number  INTEGER NOT NULL
title           VARCHAR(255) NOT NULL
description     TEXT                    -- optional chapter summary
sort_order      INTEGER DEFAULT 0
created_at      TIMESTAMP DEFAULT NOW()
```

#### `posts`
```
id              SERIAL PRIMARY KEY
chapter_id      INTEGER FK → chapters(id)
author_name     VARCHAR(100) NOT NULL
book_text       TEXT NOT NULL            -- HIDDEN: never sent to frontend
thoughts        TEXT NOT NULL            -- member's personal interpretation
ai_response     TEXT                     -- Qwen 2.5 generated interpretation
ai_prompt_used  TEXT                     -- stored for debugging/tuning
created_at      TIMESTAMP DEFAULT NOW()
updated_at      TIMESTAMP DEFAULT NOW()
```

#### `comments`
```
id              SERIAL PRIMARY KEY
post_id         INTEGER FK → posts(id)
parent_id       INTEGER FK → comments(id) NULL  -- threaded replies
author_name     VARCHAR(100) NOT NULL
content         TEXT NOT NULL
created_at      TIMESTAMP DEFAULT NOW()
```

#### `admin_users`
```
id              SERIAL PRIMARY KEY
username        VARCHAR(100) UNIQUE NOT NULL
password_hash   VARCHAR(255) NOT NULL
created_at      TIMESTAMP DEFAULT NOW()
```

### Key Constraints

- `book_text` is NEVER included in any public API response or template context
- Comments are flat with optional `parent_id` for one-level threading
- No user accounts for members — `author_name` is freeform text per action
- Admin auth is separate, simple session-based login for admin panel

---

## Application Structure

```
path-of-awakening-club/
├── CLAUDE.md
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── alembic.ini
├── alembic/
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app entry
│   ├── config.py                # Settings (DB, HF API key, etc.)
│   ├── database.py              # SQLAlchemy engine + session
│   ├── models/
│   │   ├── __init__.py
│   │   ├── chapter.py
│   │   ├── post.py
│   │   ├── comment.py
│   │   └── admin.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── chapter.py
│   │   ├── post.py
│   │   └── comment.py
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── pages.py             # Public page routes (Jinja2)
│   │   ├── posts.py             # Post CRUD API
│   │   ├── comments.py          # Comment API
│   │   ├── chapters.py          # Chapter management
│   │   ├── admin.py             # Admin panel routes
│   │   └── ai.py                # Hugging Face integration
│   ├── services/
│   │   ├── __init__.py
│   │   ├── ai_service.py        # Qwen 2.5 prompt + API call
│   │   └── spam_service.py      # Honeypot + rate limiting
│   ├── templates/
│   │   ├── base.html            # Master layout with Buddhist theme
│   │   ├── index.html           # Home — chapter list
│   │   ├── chapter.html         # Chapter view — list of posts
│   │   ├── post.html            # Single post — side-by-side view
│   │   ├── post_create.html     # New post form
│   │   ├── docs.html            # In-app Docs: architecture, credits, member guide
│   │   ├── admin/
│   │   │   ├── login.html
│   │   │   ├── dashboard.html
│   │   │   ├── chapters.html
│   │   │   └── comments.html
│   │   └── components/
│   │       ├── navbar.html
│   │       ├── comment_thread.html
│   │       └── footer.html
│   └── static/
│       ├── css/
│       │   └── style.css        # Buddhist theme styles
│       ├── js/
│       │   └── app.js           # Comment submission, interactions
│       └── img/
│           └── lotus.svg        # Decorative motif
└── scripts/
    └── seed_chapters.py         # Pre-populate chapter list
```

---

## LLM Integration — Qwen 2.5 via Hugging Face

### System Prompt

```
You are a thoughtful reader and interpreter of Buddhist philosophical texts,
specifically the works of Dharma Master Shih Cheng Yen (證嚴法師), founder of
the Tzu Chi Foundation.

When given a passage from "The Path of Awakening," provide an interpretation that:

1. Explains the deeper dharma meaning behind the everyday language
2. Connects the teaching to practical daily life application
3. Identifies the core Buddhist concepts being conveyed (impermanence,
   compassion, cause and condition, mindfulness, etc.)
4. Respects the Tzu Chi tradition of combining spiritual practice with
   humanitarian action
5. Keeps the tone warm, contemplative, and accessible — not academic

Write in clear English. Keep the interpretation concise (2-4 paragraphs).
Do not summarize — interpret and illuminate.
```

### API Flow

1. Member pastes book text + writes their thoughts
2. Backend sends ONLY the book text to Qwen 2.5 (member thoughts are independent)
3. AI response is saved to `posts.ai_response`
4. Member can preview AI interpretation before publishing
5. Member can regenerate if the AI response is weak
6. Published post shows: chapter reference, member thoughts (left), AI interpretation (right)

### Fallback Strategy

- Primary: Hugging Face Inference API (Qwen 2.5 72B or 14B)
- Fallback: Local Ollama instance running Qwen 2.5 7B
- If both fail: Post publishes without AI interpretation, flagged for retry

---

## Page Routes

| Route | Purpose |
|-------|---------|
| `GET /` | Home — chapter list with post counts |
| `GET /chapter/{id}` | Chapter view — all posts for this chapter |
| `GET /post/{id}` | Single post — side-by-side thoughts + AI, comments |
| `GET /post/new` | New post form (select chapter, paste text, write thoughts) |
| `POST /post/create` | Submit new post, trigger AI generation |
| `POST /post/{id}/regenerate` | Regenerate AI interpretation |
| `POST /post/{id}/comment` | Add comment to a post |
| `GET /docs` | In-app documentation: architecture, member guide, credits |
| `GET /admin/login` | Admin login |
| `GET /admin/dashboard` | Admin panel — manage chapters, delete comments |

---

## Docs Page (`/docs`)

The Docs page is an in-app page rendered at `/docs` using the same Buddhist theme as the rest of the application. It is linked from the navbar and accessible to all visitors. The page has three sections presented as anchor-linked tabs or scrollable sections.

### Section 1: How to Use This App (Member Guide)

Friendly walkthrough for book club members. Written in warm, simple language.

**Content:**
- What this app is (a companion tool for our book club reading of *The Path of Awakening*)
- How to read a discussion (chapter list → post → side-by-side thoughts + AI interpretation)
- How to start a new discussion post:
  1. Click "New Discussion"
  2. Select the chapter you want to discuss
  3. Copy and paste the passage from your book (this stays private, only the AI reads it)
  4. Write your personal thoughts and reflections
  5. Submit — the AI will generate its own interpretation
  6. Preview both sides, then publish
- How to comment (type your name, write your thoughts, submit — supports English, Chinese, or mixed)
- How the AI interpretation works (it reads the passage independently and offers a Buddhist philosophical perspective — it doesn't see your thoughts, so you get two genuine viewpoints)
- Note: everyone needs their own copy of the book — the app references chapters but doesn't show the book text

### Section 2: How This App Is Built (Architecture)

Technical overview for anyone curious. Rendered from the same content as the standalone ARCHITECTURE.md file. Includes:

- System architecture diagram (ASCII or simple visual)
- Discussion flow diagram (paste → thoughts → AI → publish → comments)
- Technology stack table (FastAPI, PostgreSQL, Qwen 2.5, Bootstrap 5, Docker)
- Why each technology was chosen (short rationale)
- AI interpretation strategy (independent from member thoughts, Buddhist-tuned prompt)
- Comment system design (threaded, bilingual, no registration)

### Section 3: Credits & Acknowledgments

- **The Book**: *The Path of Awakening* by Dharma Master Shih Cheng Yen (證嚴法師). This book was personally purchased by the application creator. No copyrighted content is displayed publicly.
- **Shih Cheng Yen**: Brief respectful note about the author and founder of Tzu Chi Foundation, with link to [tzuchi.org](https://www.tzuchi.org)
- **AI Interpretation**: Powered by Qwen 2.5 (Alibaba Cloud) via Hugging Face Inference API
- **Built with**: FastAPI, PostgreSQL, Bootstrap 5, SQLAlchemy, Docker
- **Open source libraries**: list key dependencies with links

### Docs Page Design Notes

- Same Buddhist peaceful theme as the rest of the app
- In-page navigation (anchor links or tab-style switcher) for the three sections
- Keep the member guide section warm and non-technical
- Keep the architecture section informative but not overwhelming — it's for the curious, not a developer onboarding doc
- Credits section should feel respectful and grateful, not just a list

---

## Anti-Spam Strategy

Since there is no registration and comments publish immediately:

1. **Honeypot field** — hidden form field; bots fill it, humans don't. Reject if filled.
2. **Rate limiting** — max 5 comments per IP per 10 minutes via FastAPI middleware.
3. **Minimum content length** — comments must be at least 2 characters.
4. **Admin delete** — admin panel provides quick-delete for any comment.
5. **No links in first comment** — optional: strip or flag comments with URLs from new names.

---

## Build Phases

### Phase 1 — Foundation (MVP)
- [ ] Project scaffolding with FastAPI + SQLAlchemy + Alembic
- [ ] PostgreSQL database setup (`awakening_club` on shared instance)
- [ ] Data models: chapters, posts, comments
- [ ] Buddhist-themed base template with Bootstrap 5
- [ ] Chapter list page (home)
- [ ] Post creation form (paste book text, write thoughts)
- [ ] Post display with side-by-side layout
- [ ] Comment system with display name (threaded)
- [ ] Seed script for chapter list from The Path of Awakening

### Phase 2 — AI Integration
- [ ] Hugging Face API service for Qwen 2.5
- [ ] System prompt tuned for Buddhist philosophical interpretation
- [ ] AI interpretation generation on post creation
- [ ] Regenerate button for post author
- [ ] Fallback to Ollama local if HF API fails

### Phase 3 — Polish & Admin
- [ ] Admin authentication (simple session login)
- [ ] Admin panel: manage chapters, delete comments, view stats
- [ ] Honeypot + rate limiting spam protection
- [ ] Docs page (architecture + credits)
- [ ] Mobile responsive refinement
- [ ] Loading states and error handling

### Phase 4 — Deployment
- [ ] Dockerfile + docker-compose.yml
- [ ] Environment variable configuration
- [ ] Dokploy deployment on homelab
- [ ] PostgreSQL connection to shared instance
- [ ] Basic health check endpoint

---

## Environment Variables

```
DATABASE_URL=postgresql://awakening_user:password@postgres:5432/awakening_club
HF_API_TOKEN=hf_xxxxxxxxxxxxx
HF_MODEL=Qwen/Qwen2.5-72B-Instruct
OLLAMA_URL=http://ollama:11434
ADMIN_USERNAME=admin
ADMIN_PASSWORD_HASH=bcrypt_hash_here
SECRET_KEY=random_secret_for_sessions
```

---

## Key Decisions Log

| Decision | Rationale |
|----------|-----------|
| No user registration | Remove friction — existing book club, trust-based |
| Book text hidden from frontend | Copyright safe — text is LLM input only |
| Display name per action (no cookie) | Simplest approach, members type name each time |
| Qwen 2.5 over Llama | Better Chinese cultural/Buddhist context understanding |
| AI interprets independently from member thoughts | Creates genuine contrast for richer discussion |
| Side-by-side layout | Visual comparison invites readers to evaluate both perspectives |
| PostgreSQL shared instance | One DB server for all homelab apps, easier ops |
| No moderation queue | Trust-based community + admin delete as safety net |
| One book at a time | Focused discussion, can archive and start new book later |
