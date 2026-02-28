# Polyglotte — Suggestions & Improvement Roadmap

> Generated: 2026-02-28
> Fork: EmersonBraun/polyglotte | Upstream: lucasalbuquerque/polyglotte

---

## Notes on Upstream Issue Analysis

The GitHub CLI (`gh`) and all other network-dependent tools were unavailable during
this analysis due to a pending Xcode license agreement on this machine (`sudo xcodebuild -license`).
As a result, the upstream issue list from `lucasalbuquerque/polyglotte` could not be fetched
automatically.

**Action required:** Run `sudo xcodebuild -license` in a terminal, agree to the Xcode/Apple SDK
license, and then re-run `gh issue list --repo lucasalbuquerque/polyglotte --limit 50 --state open`
to retrieve the live issue list and reconcile it with the plans below.

The rest of this document provides comprehensive, actionable improvement plans for the most
commonly requested features in language-learning applications of this type.

---

## Section 1 — Common Upstream Issues (Typical for This Class of App)

Based on the nature of the polyglotte project (a language-learning application), the following
categories of issues are typical in upstream trackers and are addressed with implementation plans.

---

### Issue Category 1 — Missing Spaced Repetition System

**Problem:** Vocabulary reviews are not scheduled intelligently; users review words at fixed
intervals or randomly, leading to poor retention.

**Implementation Plan:**

1. Implement the SM-2 algorithm (SuperMemo 2) as the default scheduler.
   - Each vocabulary card stores: `easiness_factor` (default 2.5), `interval` (days),
     `repetition_count`, and `next_review_date`.
   - After each review the user rates recall quality from 0–5.
   - Update formula:
     ```
     if quality >= 3:
       if repetition == 0: interval = 1
       elif repetition == 1: interval = 6
       else: interval = round(interval * easiness_factor)
       repetition += 1
     else:
       repetition = 0
       interval = 1

     easiness_factor = max(1.3, easiness_factor + 0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02))
     next_review_date = today + interval
     ```
2. Create a `reviews` table/collection in the database with columns:
   `id`, `user_id`, `word_id`, `ease_factor`, `interval_days`, `repetition`,
   `last_reviewed_at`, `next_review_at`.
3. Build a `/review` route that surfaces only cards due today or overdue.
4. Add a dashboard widget showing "X cards due today" and a burndown chart.
5. Allow users to switch between SM-2 and a simpler Leitner box system via settings.

---

### Issue Category 2 — No Gamification (Streaks, XP, Achievements)

**Problem:** Users lack motivation signals beyond passive progress bars.

**Implementation Plan — Streaks:**

1. Track `last_activity_date` per user.
2. On each session completion, compare with today's date:
   - Same day: no change.
   - Consecutive day: `streak_count += 1`.
   - Gap > 1 day: `streak_count = 1` (reset).
3. Show streak flame icon with count in the nav bar.
4. Send a push/email reminder at 20:00 local time when a streak is at risk.

**Implementation Plan — XP System:**

| Action                        | XP Reward |
|-------------------------------|-----------|
| Complete a lesson              | +10 XP    |
| Correct answer (first try)     | +5 XP     |
| Correct answer (after hint)    | +2 XP     |
| Finish a review session        | +8 XP     |
| 7-day streak milestone         | +50 XP    |
| Complete a language level      | +100 XP   |

5. Store `xp_total` and `xp_this_week` in the user profile.
6. Add a weekly leaderboard (opt-in) showing top learners by XP.

**Implementation Plan — Achievements:**

7. Define an `achievements` table: `id`, `slug`, `title`, `description`, `icon`, `xp_bonus`.
8. Define an `user_achievements` table: `user_id`, `achievement_id`, `unlocked_at`.
9. Create an achievement engine that runs after every session and checks unlock conditions.
10. Example achievements:
    - "First Word" — learn your first vocabulary item.
    - "Week Warrior" — maintain a 7-day streak.
    - "Century" — answer 100 questions correctly.
    - "Polyglot" — start studying a second language.
    - "Night Owl" — complete a lesson after midnight.
    - "Speed Learner" — complete a lesson in under 2 minutes.
11. Show an animated modal when an achievement is unlocked.

---

### Issue Category 3 — No Audio Pronunciation

**Problem:** Users cannot hear how words are pronounced, which is critical for language acquisition.

**Implementation Plan:**

1. Integrate a Text-to-Speech (TTS) provider. Recommended options (in order of preference):
   - **Web Speech API** — free, browser-native, no backend needed, covers ~40 languages.
   - **Google Cloud TTS** — high quality Neural2/Studio voices, pay-per-character.
   - **ElevenLabs** — best naturalness, higher cost.
2. For Web Speech API (zero-cost starting point):
   ```js
   function speak(text, langCode = 'en-US') {
     const utterance = new SpeechSynthesisUtterance(text);
     utterance.lang = langCode;
     utterance.rate = 0.85; // slightly slower for learners
     window.speechSynthesis.speak(utterance);
   }
   ```
3. Add a speaker icon button next to every vocabulary word and example sentence.
4. Add a "slow pronunciation" mode (rate = 0.5) toggled by a double-click on the speaker icon.
5. For premium/server-side TTS: cache generated audio files in object storage (S3/R2)
   using `{lang}_{word_slug}.mp3` naming to avoid duplicate API calls.
6. Add a "pronunciation quiz" exercise type: play audio only, user types what they hear.

---

### Issue Category 4 — Insufficient Progress Tracking

**Problem:** Users cannot see their learning history, weak areas, or time invested.

**Implementation Plan:**

1. Create a `learning_sessions` table: `id`, `user_id`, `language`, `started_at`,
   `ended_at`, `words_reviewed`, `correct_count`, `xp_earned`.
2. Build a `/progress` dashboard page with:
   - Total words learned (by language).
   - Accuracy rate over time (line chart, 30-day window).
   - Minutes studied per day (bar chart).
   - Vocabulary heatmap (GitHub-style, one square per day).
   - Skill breakdown radar chart by category (verbs, nouns, phrases, grammar).
3. Add a "weak words" list: words answered incorrectly more than twice in the last 7 days,
   surfaced automatically at the start of each session.
4. Export progress as CSV or PDF for users who want offline records.

---

### Issue Category 5 — No Offline Mode

**Problem:** The app requires an internet connection; users cannot study on planes, subways, etc.

**Implementation Plan:**

1. Convert the app to a Progressive Web App (PWA):
   - Add `manifest.json` with `name`, `short_name`, `icons`, `start_url`, `display: standalone`.
   - Register a Service Worker (`sw.js`) using the Workbox library.
2. Cache strategy:
   - **App shell** (HTML/CSS/JS): Cache-first.
   - **Lesson content** (JSON word lists): Stale-while-revalidate; pre-cache on first load.
   - **Audio files**: Cache on first play; evict LRU when cache exceeds 50 MB.
3. Sync strategy:
   - Queue review results in IndexedDB while offline using a `pending_syncs` store.
   - On reconnection, flush the queue to the server in a background sync event.
4. Add an offline indicator banner when `navigator.onLine === false`.
5. Let users manually select language packs to download for offline use from the settings page.

---

### Issue Category 6 — Mobile / PWA Support

**Problem:** The app is not installable on mobile devices and may not be responsive on small screens.

**Implementation Plan:**

1. Complete the PWA setup described in Issue Category 5 above.
2. Ensure all tap targets are at least 44×44 px (WCAG 2.1 minimum).
3. Use CSS `clamp()` for fluid typography so text scales naturally between 320 px and 1440 px.
4. Implement swipe gestures on flashcard components:
   - Swipe right → "I knew this" (correct).
   - Swipe left → "I didn't know" (incorrect).
   - Use the `PointerEvents` API or a lightweight library like `hammer.js`.
5. Add a bottom navigation bar on mobile (Home, Learn, Review, Progress, Profile).
6. Test on iOS Safari and Android Chrome; handle `safe-area-inset` for notched phones.
7. Add a "Install App" prompt (BeforeInstallPrompt event) after the user's second session.

---

### Issue Category 7 — Limited Language Pairs

**Problem:** The app supports only a small number of language combinations.

**Implementation Plan:**

1. Audit existing data model to ensure `source_language` and `target_language` are first-class
   fields on all word/lesson records (not hardcoded).
2. Create a `languages` table: `code` (BCP-47), `name_en`, `name_native`, `flag_emoji`,
   `rtl` (boolean), `tts_locale`.
3. Prioritize adding languages by global learner demand:
   - Tier 1 (high demand): Spanish, French, German, Japanese, Korean, Mandarin, Italian,
     Portuguese, Arabic.
   - Tier 2 (medium demand): Russian, Dutch, Swedish, Polish, Turkish, Hindi, Vietnamese.
   - Tier 3 (community-contributed): Any language with a willing contributor.
4. Build a community contribution workflow:
   - A `word_contributions` table for user-submitted words pending moderation.
   - A simple admin review UI to approve/reject submissions.
5. Support right-to-left (RTL) rendering for Arabic and Hebrew using `dir="rtl"` on the
   relevant containers and `unicode-bidi: isolate` in CSS.

---

### Issue Category 8 — No Vocabulary Flashcards

**Problem:** Users have no dedicated flashcard review mode; all learning is exercise-only.

**Implementation Plan:**

1. Create a `Flashcard` component:
   - Front: target-language word + optional image.
   - Back: source-language translation + example sentence + pronunciation button.
   - 3D CSS flip animation on click/tap.
2. Flashcard deck modes:
   - **Auto-play** — cards advance every N seconds (configurable 3–10 s).
   - **Self-paced** — user taps to flip, then marks Known / Unknown.
   - **Cram mode** — cycles through a fixed set until all are marked Known.
3. Allow users to create custom decks from any word list or their personal vocabulary list.
4. Enable deck sharing via a shareable URL token.
5. Integrate flashcard performance data into the spaced repetition scheduler (Section 1).

---

### Issue Category 9 — No Grammar Exercises

**Problem:** The app focuses on vocabulary but lacks structured grammar practice.

**Implementation Plan:**

1. Define exercise types as a plugin/strategy pattern so new types are easy to add:
   - `fill-in-the-blank` — sentence with one word removed; user types or selects.
   - `word-order` — scrambled words that the user drags into correct order.
   - `conjugation-table` — given a verb infinitive, fill in conjugations for listed pronouns.
   - `multiple-choice-grammar` — choose the grammatically correct form from 4 options.
   - `error-correction` — identify and correct the mistake in a sentence.
2. Create a `grammar_rules` table: `id`, `language`, `category` (tense, gender, case, etc.),
   `title`, `explanation_html`, `examples_json`.
3. Link each exercise to one or more grammar rules so wrong answers surface a relevant
   grammar tip immediately.
4. Build a grammar "skills tree" UI similar to a course curriculum: complete earlier nodes
   to unlock advanced ones.
5. Track grammar accuracy by category and display in the Progress dashboard radar chart.

---

## Section 2 — General Architecture Suggestions

### 2.1 — Database Schema Additions Summary

```sql
-- Spaced repetition
ALTER TABLE user_words ADD COLUMN ease_factor   REAL    DEFAULT 2.5;
ALTER TABLE user_words ADD COLUMN interval_days INTEGER DEFAULT 1;
ALTER TABLE user_words ADD COLUMN repetition    INTEGER DEFAULT 0;
ALTER TABLE user_words ADD COLUMN next_review_at TIMESTAMP;

-- Gamification
ALTER TABLE users ADD COLUMN streak_count    INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN last_activity_date DATE;
ALTER TABLE users ADD COLUMN xp_total        INTEGER DEFAULT 0;

CREATE TABLE achievements (
  id          SERIAL PRIMARY KEY,
  slug        TEXT UNIQUE NOT NULL,
  title       TEXT NOT NULL,
  description TEXT,
  icon        TEXT,
  xp_bonus    INTEGER DEFAULT 0
);

CREATE TABLE user_achievements (
  user_id        INTEGER REFERENCES users(id),
  achievement_id INTEGER REFERENCES achievements(id),
  unlocked_at    TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, achievement_id)
);

-- Sessions
CREATE TABLE learning_sessions (
  id             SERIAL PRIMARY KEY,
  user_id        INTEGER REFERENCES users(id),
  language       TEXT NOT NULL,
  started_at     TIMESTAMP NOT NULL,
  ended_at       TIMESTAMP,
  words_reviewed INTEGER DEFAULT 0,
  correct_count  INTEGER DEFAULT 0,
  xp_earned      INTEGER DEFAULT 0
);
```

### 2.2 — Performance Recommendations

- Paginate all word list API responses (max 50 items per page).
- Add a database index on `next_review_at` filtered to `WHERE next_review_at <= NOW()`.
- Use a CDN (Cloudflare, Fastly) to serve audio and image assets.
- Implement HTTP/2 push or `<link rel="preload">` for lesson JSON on the lesson route.

### 2.3 — Accessibility

- Ensure all interactive elements have ARIA labels (`aria-label` or `aria-labelledby`).
- Support keyboard-only navigation through all exercise and flashcard flows.
- Provide a high-contrast theme toggle.
- Add `lang` attribute to all rendered foreign-language text fragments so screen readers
  use the correct TTS voice.

### 2.4 — Testing Strategy

- Unit tests for the SM-2 algorithm and XP calculation logic (Jest or Vitest).
- Component tests for Flashcard flip animation and swipe gesture handling.
- E2E tests (Playwright) for the full lesson → review → progress dashboard flow.
- Accessibility audit with `axe-core` in CI.

---

## Section 3 — Quick-Win Checklist (Effort vs. Impact)

| Priority | Feature                          | Effort | Impact |
|----------|----------------------------------|--------|--------|
| P0       | Spaced repetition (SM-2)         | Medium | High   |
| P0       | Audio pronunciation (Web Speech) | Low    | High   |
| P1       | Streak counter                   | Low    | High   |
| P1       | PWA manifest + service worker    | Low    | High   |
| P1       | Flashcard component              | Medium | High   |
| P2       | XP system                        | Medium | Medium |
| P2       | Progress dashboard               | Medium | High   |
| P2       | Grammar fill-in-the-blank        | Medium | High   |
| P3       | Achievements system              | High   | Medium |
| P3       | Additional language pairs        | High   | High   |
| P3       | Community word contributions     | High   | Medium |

---

*End of SUGGESTIONS.md*
