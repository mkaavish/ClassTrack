# ClassTrack — AI Syllabus Deadline Tracker

> iOS app that turns any course syllabus into a smart deadline calendar using AI.

---

## What It Does

Students upload a syllabus — as a PDF, photo, or text file. ClassTrack uses AI to instantly extract every exam, quiz, and assignment, then organizes them into a personal deadline dashboard with Apple Calendar sync.

No manual entry. No missed deadlines.

---

## Features

- **AI Parsing** — Upload a PDF or photo; Claude extracts all graded events with titles, types, and ISO due dates
- **Urgency Dashboard** — Color-coded cards (red ≤ 2 days, gold ≤ 7 days) with stats for exams, quizzes, and assignments
- **Monthly Calendar** — Dot indicators on event days with a tappable day view
- **Apple Calendar Sync** — One-tap export or auto-sync on upload via EventKit
- **OCR Support** — Photos of physical syllabi work via Apple Vision framework
- **Professor Info** — Contact details extracted and stored per course
- **Free / Pro Tiers** — Free accounts support 2 courses; Pro removes the limit
- **Theming** — Multiple accent colors with a persistent dark UI

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              iOS App (SwiftUI)               │
│                                             │
│  UploadView → Vision OCR / PDFKit           │
│       │                                     │
│       ▼                                     │
│  APIClient ──────────────────────────────┐  │
│                                          │  │
└──────────────────────────────────────────┼──┘
                                           │
                                     HTTPS + JWT
                                           │
                          ┌────────────────▼────────────────┐
                          │     Vercel Serverless Function   │
                          │        POST /api/parse           │
                          │                                  │
                          │  1. Verify Supabase JWT          │
                          │  2. Check free-tier limit        │
                          │  3. Call Claude Haiku            │
                          │  4. Parse structured JSON        │
                          │  5. Write to Supabase            │
                          └──────────┬──────────────────────┘
                                     │
                     ┌───────────────┴──────────────┐
                     │         Supabase              │
                     │                              │
                     │  auth.users                  │
                     │  profiles (free/pro tier)    │
                     │  courses                     │
                     │  professors                  │
                     │  course_events               │
                     └──────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| iOS UI | Swift / SwiftUI |
| OCR | Apple Vision framework |
| PDF parsing | PDFKit |
| Calendar sync | EventKit |
| Auth & Database | Supabase (PostgreSQL) |
| Secure storage | Keychain |
| Backend API | TypeScript / Node.js |
| Deployment | Vercel Serverless Functions |
| AI | Anthropic Claude Haiku |

---

## How the AI Parsing Works

The backend sends raw syllabus text to Claude with a structured system prompt that enforces a strict JSON schema:

```typescript
// Simplified system prompt
`Extract all graded events from this syllabus.
Return ONLY a JSON object:
{
  "course_name": string | null,
  "course_code": string | null,
  "semester": string | null,
  "professor": { "name", "email", "office_location" } | null,
  "events": [
    {
      "title": string,
      "event_type": "exam" | "quiz" | "assignment" | "other",
      "due_date": "YYYY-MM-DD" | null,
      "uncertain": boolean
    }
  ]
}`
```

Claude returns structured data in one shot — no post-processing regex or date parsing needed. Uncertain dates (e.g. "Week 8") are flagged with `uncertain: true` and shown with a gold warning in the UI.

---

## iOS Urgency Logic

Event cards are color-coded client-side with no extra API calls:

```swift
private var cardColor: Color {
    guard let days = event.daysUntilDue else { return .white.opacity(0.07) }
    if days <= 2 { return .red.opacity(0.15) }
    if days <= 7 { return .scholarGold.opacity(0.1) }
    return .white.opacity(0.07)
}
```

---

## Database Schema (simplified)

```sql
-- Courses created from each syllabus upload
create table courses (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid references auth.users,
  course_name text,
  course_code text,
  semester    text,
  raw_text    text,
  created_at  timestamptz default now()
);

-- Individual deadlines extracted by AI
create table course_events (
  id            uuid primary key default gen_random_uuid(),
  course_id     uuid references courses,
  user_id       uuid references auth.users,
  title         text not null,
  event_type    text,  -- exam | quiz | assignment | other
  due_date      date,
  apple_event_id text,  -- set after Apple Calendar sync
  created_at    timestamptz default now()
);
```

---

## Key Design Decisions

**Why Claude Haiku?** Fast, cheap, and accurate enough for structured extraction. The system prompt enforces a strict JSON schema — no output parsing needed beyond `JSON.parse()`.

**Why Vercel Functions?** The parse endpoint is the only server-side logic. A full server would be overkill; a single function keeps cold starts low and cost near zero.

**Why Supabase?** Auth + Postgres + Row Level Security in one service. RLS policies ensure users can only read their own courses and events — no authorization code needed in the API.

**Free tier limit enforced server-side** — the iOS app checks count before calling Claude, but the Vercel function is the authoritative gate to prevent client-side bypasses.

---

## Status

Actively in development. Core upload → parse → display flow is working. Apple Calendar sync, push notifications, and Pro upgrade flow are implemented.

---

*The full source code is private. This repo is a portfolio showcase.*
