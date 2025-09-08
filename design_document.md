Personal Weekly Journaling Tool - Design Document
=================================================

Author: Kamol
Date: 2025-09-08

-------------------------------------------------
1. Product Overview
-------------------------------------------------
A personal web app to track weekly progress as a micro-SaaS founder. 
It enforces a structured journaling format while integrating an AI assistant 
to provide real-time feedback and polishing. The goal is accountability, 
reflection, and easy retrieval of past insights.

-------------------------------------------------
2. User Experience & UI
-------------------------------------------------
Entry Flow:
- Typeform-style: one question at a time, full-screen focus.
- Prompts:
  1. What I did this week?
  2. What went well?
  3. What could be improved?
  4. What's left to do next week?
  5. Anything cool or interesting I found?
  6. What insight or finding did I learn this week?
- Smooth fade/slide transitions.
- Top progress bar (e.g., 2/6).

Writing Mode:
- Font: Geist Mono, larger size (16–18px, line-height 1.6) for deliberate pace.
- Side feedback panel (collapsible, docked right):
  * Tone feedback (concise, rambling, etc.)
  * Suggested tags/topics
  * Draft polishing suggestions
- Real-time updates triggered on pause.

Reading Mode:
- Font: Geist Sans for long-form readability.
- Highlights:
  * Insights in callout boxes
  * “Cool finds” rendered as card gallery with link previews
- Clean, journal-like feel with minimalist design.
- Option for compare view (two weeks side by side).

Library & Navigation:
- Vertical timeline view of weekly entries.
- Filters: tags, topics, date range.
- Dashboard: top tags, word count trends, insight bullets.

Accountability Features:
- Automatic weekly entry creation (Monday start).
- Roll-over of previous week’s “Next week” items.
- Weekly nudge notification/reminder.
- Streak indicator (subtle, habit reinforcing).
- Quarterly review mode (auto-summary of 12 weeks).

-------------------------------------------------
3. Architecture & Stack
-------------------------------------------------
Frontend:
- Vite + React SPA
- React Router for navigation
- Geist font family (Mono for writing, Sans for reading)

Backend:
- Firebase Auth (Google + Email/Password)
- Firestore for journal storage
- Firebase Functions for:
  * LLM proxy (polish, insights, tags)
  * Signed search key generation

Search:
- Firestore for structured filters (tags, date)
- Typesense (via Firebase extension) for full-text + faceted search

Hosting & CI/CD:
- Firebase Hosting
- GitHub Actions (PR previews + production deploys)

Testing:
- Playwright for E2E (entry creation, polish, search, permissions)

-------------------------------------------------
4. Data Model (Firestore)
-------------------------------------------------
Collection: journalEntries
{
  id: string (auto)
  userId: string (auth.uid)
  weekStart: string (ISO date of Monday)
  didThisWeek: string
  wentWell: string
  improvements: string
  nextWeek: string
  coolFinds: [{title: string, url: string}]
  insights: [string]
  tags: [string]
  topics: [string]
  wordCount: number
  createdAt: timestamp
  updatedAt: timestamp
}

Indexes:
- (userId, weekStart desc)
- (userId, tags array-contains)

-------------------------------------------------
5. AI Assistant Integration
-------------------------------------------------
Flow:
- User writes in prompt field.
- Content streamed to Firebase Function.
- Function calls LLM (structured JSON response).
- Feedback displayed in side panel in real-time.

LLM Returns:
- Polished section text
- Extracted insights (bullets)
- Suggested tags/topics
- One-line weekly summary

Modes:
- Inline feedback (non-blocking)
- Full “Review & Polish” screen after all prompts completed

-------------------------------------------------
6. Security & Privacy
-------------------------------------------------
- Firestore rules restrict read/write to own userId.
- Search engine access via signed per-user keys.
- No raw LLM API keys exposed to frontend.
- All entries private; not shareable by default.

-------------------------------------------------
7. Fonts & Design Language
-------------------------------------------------
- Writing mode: Geist Mono (raw, deliberate, reflective)
- Reading mode: Geist Sans (smooth, fast, reflective)
- Color palette: minimal white/gray with soft accent (blue/green)
- Typography hierarchy:
  * Prompts/questions: bold, 20px
  * Input text: 16–18px, line-height 1.6
  * Reading view body: 16px Geist Sans
  * Insights: italic or callout box

-------------------------------------------------
8. Rollout Checklist
-------------------------------------------------
1. Auth (Email/Password)
2. Firestore schema + security rules
3. Weekly entry editor (Typeform-style + autosave)
4. Side feedback panel with real-time AI integration
5. Search extension install + sync
6. Library timeline & dashboard
7. Playwright tests
8. GitHub Actions + Firebase Hosting deploy
