# Lojban PWA Design Document

**Date:** 2026-02-19
**Status:** Approved
**Purpose:** Design a Progressive Web App for the Lojban 30-week curriculum

---

## Overview

A mobile-first PWA that delivers 210 daily lessons (30 weeks × 7 days) with interactive exercises, progress tracking, and full offline support. Users can install it to their home screen and learn Lojban in 15 minutes per day.

---

## Requirements

| Requirement | Decision |
|-------------|----------|
| Exercise interactivity | Full validation (input + check + feedback) |
| Vocabulary review | Lesson-only (no separate SRS) |
| Hosting | GitHub Pages |
| Visual style | Minimal, clean, mobile-first |
| Content format | Markdown → JSON at build time |
| Tech stack | Vanilla JS + esbuild |

---

## Architecture

**Approach:** Single-page app with hash-based routing

```
index.html ← single shell, never reloads
app.js     ← handles routing (#w01d1, #w02d3, etc.)
lessons/   ← JSON files fetched on demand
sw.js      ← caches everything for offline
```

- One HTML file, JS swaps content based on URL hash
- Lessons are JSON, fetched and rendered client-side
- Progress in localStorage
- Service worker pre-caches all 210 lessons on first visit

---

## File Structure

```
lojban-app/
├── src/
│   ├── index.html          # App shell
│   ├── css/
│   │   └── style.css       # All styles
│   ├── js/
│   │   ├── app.js          # Main entry: router, state, init
│   │   ├── router.js       # Hash-based navigation
│   │   ├── renderer.js     # Renders lesson JSON to HTML
│   │   ├── exercises.js    # Exercise validation logic
│   │   └── progress.js     # localStorage wrapper
│   ├── sw.js               # Service worker
│   └── manifest.json       # PWA manifest
│
├── lessons/                 # Source markdown (existing files)
│   ├── w01d1.md
│   └── ...
│
├── build/
│   ├── convert.js          # Node script: md → JSON
│   └── bundle.js           # esbuild script
│
├── dist/                    # Build output (deploy this)
│   ├── index.html
│   ├── app.js              # Bundled JS
│   ├── style.css
│   ├── sw.js
│   ├── manifest.json
│   └── lessons/
│       ├── w01d1.json
│       └── ...
│
└── package.json            # npm scripts: build, dev, deploy
```

---

## Lesson JSON Format

```json
{
  "id": "w01d1",
  "week": 1,
  "day": 1,
  "title": "Your Mouth Learns Lojban",
  "goal": "Pronounce all Lojban vowels...",
  "time": "15 minutes",

  "sections": [
    {
      "type": "warmup",
      "content": "<p>HTML content...</p>"
    },
    {
      "type": "material",
      "title": "Lojban Sounds",
      "content": "<p>HTML content...</p>",
      "keyPoint": "Every Lojban letter has exactly one sound."
    },
    {
      "type": "exercise",
      "exerciseType": "fill-blank",
      "instructions": "Which vowel sounds do you hear?",
      "items": [
        { "prompt": "prami → ___, ___", "answer": "a, i" }
      ]
    },
    {
      "type": "conversation",
      "prompts": [
        { "english": "Say all 12 gismu in order", "answer": "prami, klama..." }
      ]
    },
    {
      "type": "vocab",
      "items": [
        { "lojban": "prami", "place": "x1 loves x2", "english": "love" }
      ]
    },
    {
      "type": "summary",
      "content": "You learned Lojban's 6 vowels...",
      "tomorrow": "You'll learn what mi, do, and cu mean..."
    }
  ]
}
```

### Exercise Types

| Type | Validation |
|------|------------|
| `pronunciation` | None (say aloud) |
| `fill-blank` | Exact match, normalized |
| `translation-to-english` | Exact match |
| `translation-to-lojban` | Exact match, accept alternatives |
| `matching` | Tap pairs to connect |
| `error-correction` | Compare to correct answer |
| `free-production` | Show sample answer (no strict check) |

---

## UI Layout

```
┌─────────────────────────────────┐
│  ◀ Week 1 · Day 1    ⚙️        │  Header
├─────────────────────────────────┤
│  ████████░░░░░░░░░░  Day 1/7   │  Progress bar
├─────────────────────────────────┤
│  [Warm-up] [Learn] [Practice]   │  Section tabs
├─────────────────────────────────┤
│                                 │
│  Lesson content...              │  Scrollable content
│  Exercises...                   │
│  Vocab table...                 │
│  Summary...                     │
│                                 │
├─────────────────────────────────┤
│  [◀ Previous]    [Next Day ▶]  │  Footer nav
└─────────────────────────────────┘
```

### Navigation Flow

```
Home (#)
  └── Week selector grid (Weeks 1-30)
        └── Day view (#w01d1)
              ├── Section tabs (scroll within page)
              ├── Previous/Next day buttons
              └── Back to week selector
```

---

## Exercise Validation

### Fill-in-the-blank

1. Normalize: lowercase, trim, collapse spaces
2. Compare exact match OR fuzzy (ignore comma spacing)
3. Lojban-specific: case-insensitive, `.` optional before names

### Multiple valid answers

```json
{ "prompt": "I love you", "answer": ["mi prami do", "mi do prami"] }
```

### Matching

1. Tap left item (highlights)
2. Tap right item
3. Correct: both turn green, lock
4. Wrong: shake, reset

### Feedback States

| State | Visual |
|-------|--------|
| Unanswered | Neutral border |
| Correct | Green border + ✓ |
| Incorrect | Red border + ✗ + correct answer |

---

## Progress Tracking

### localStorage Schema

```javascript
{
  "version": 1,
  "currentLesson": "w03d2",
  "completed": {
    "w01d1": true,
    "w01d2": true
  },
  "exerciseState": {
    "w03d2": {
      "ex1": { "answered": true, "correct": true }
    }
  },
  "settings": {
    "fontSize": "medium"
  }
}
```

### Completion Logic

- Lesson marked complete when user clicks "Next Day" or "Mark Complete"
- Explicit action required (not auto-complete)
- Revisiting completed lessons allowed

---

## PWA & Offline

### Service Worker Strategy

- **Install:** Pre-cache all assets (shell + 210 lessons)
- **Fetch:** Cache-first for everything
- **Update:** New SW installs, activates on next visit

### Cache Size

| Content | Size |
|---------|------|
| HTML shell | ~5 KB |
| CSS | ~15 KB |
| JS (bundled) | ~30 KB |
| 210 lesson JSONs | ~1.5 MB |
| Icons | ~50 KB |
| **Total** | **~1.6 MB** |

### manifest.json

```json
{
  "name": "Lojban 30-Week Course",
  "short_name": "Lojban",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2d3748"
}
```

---

## Visual Design

### Design Tokens

```css
:root {
  --bg: #ffffff;
  --bg-alt: #f7f7f8;
  --text: #1a1a1a;
  --text-muted: #6b7280;
  --primary: #2563eb;
  --success: #16a34a;
  --error: #dc2626;
  --border: #e5e7eb;
  --font-sans: system-ui, -apple-system, sans-serif;
  --font-mono: 'SF Mono', Consolas, monospace;
  --max-width: 40rem;
}
```

### Key Patterns

| Element | Style |
|---------|-------|
| Lojban text | Monospace, light gray background |
| Tables | Subtle borders, alternating rows |
| Buttons | Rounded, solid primary color |
| Cards | White bg, subtle shadow |
| Tap targets | Min 44px height/width |

### Animations

- Button press: subtle scale
- Incorrect answer: quick shake
- Section scroll: smooth
- Everything else static

---

## Build Pipeline

```
lessons/*.md  →  convert.js  →  dist/lessons/*.json
src/*.js      →  esbuild     →  dist/app.js
src/*.css     →  copy        →  dist/style.css
src/sw.js     →  copy        →  dist/sw.js
```

### npm scripts

```json
{
  "scripts": {
    "build": "node build/bundle.js",
    "convert": "node build/convert.js",
    "dev": "npx serve dist",
    "deploy": "gh-pages -d dist"
  }
}
```

---

## Dev Dependencies

- `esbuild` — JS bundler
- `marked` or regex — markdown parsing in convert.js
- `gh-pages` — deploy to GitHub Pages
- `serve` — local dev server

No runtime dependencies. Pure vanilla JS.
