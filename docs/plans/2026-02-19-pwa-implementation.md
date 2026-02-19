# Lojban PWA Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a mobile-first PWA that delivers 210 daily Lojban lessons with interactive exercises, progress tracking, and offline support.

**Architecture:** Single-page app with hash-based routing. Markdown lessons converted to JSON at build time. Vanilla JS renders lessons and handles exercise validation. Service worker caches everything for offline use.

**Tech Stack:** Vanilla JS, esbuild, Node.js build scripts, GitHub Pages

**Design Doc:** `docs/plans/2026-02-19-pwa-design.md`

---

## Task 1: Project Setup

**Files:**
- Create: `app/package.json`
- Create: `app/.gitignore`

**Step 1: Create app directory structure**

```bash
mkdir -p app/src/js app/src/css app/build app/dist
```

**Step 2: Create package.json**

```json
{
  "name": "lojban-pwa",
  "version": "1.0.0",
  "description": "Lojban 30-week course PWA",
  "scripts": {
    "convert": "node build/convert.js",
    "bundle": "node build/bundle.js",
    "build": "npm run convert && npm run bundle",
    "dev": "npx serve dist -l 3000",
    "deploy": "npx gh-pages -d dist"
  },
  "devDependencies": {
    "esbuild": "^0.20.0",
    "marked": "^12.0.0"
  }
}
```

**Step 3: Create .gitignore**

```
node_modules/
dist/
.DS_Store
```

**Step 4: Install dependencies**

Run: `cd app && npm install`
Expected: node_modules created, package-lock.json created

**Step 5: Commit**

```bash
git add app/package.json app/.gitignore app/package-lock.json
git commit -m "chore: initialize app project with dependencies"
```

---

## Task 2: Markdown to JSON Converter

**Files:**
- Create: `app/build/convert.js`

**Step 1: Create the converter script**

```javascript
const fs = require('fs');
const path = require('path');
const { marked } = require('marked');

const LESSONS_SRC = path.join(__dirname, '../../lessons');
const LESSONS_DIST = path.join(__dirname, '../dist/lessons');

// Ensure output directory exists
fs.mkdirSync(LESSONS_DIST, { recursive: true });

// Get all lesson files
const files = fs.readdirSync(LESSONS_SRC)
  .filter(f => f.match(/^w\d{2}d\d\.md$/))
  .sort();

console.log(`Converting ${files.length} lessons...`);

files.forEach(file => {
  const md = fs.readFileSync(path.join(LESSONS_SRC, file), 'utf-8');
  const json = parseLesson(md, file);
  const outFile = file.replace('.md', '.json');
  fs.writeFileSync(
    path.join(LESSONS_DIST, outFile),
    JSON.stringify(json, null, 2)
  );
});

console.log('Done!');

function parseLesson(md, filename) {
  // Extract week and day from filename (w01d1.md)
  const match = filename.match(/w(\d{2})d(\d)/);
  const week = parseInt(match[1], 10);
  const day = parseInt(match[2], 10);
  const id = `w${match[1]}d${match[2]}`;

  // Extract title from first H1
  const titleMatch = md.match(/^# Week \d+ · Day \d+ — (.+)$/m);
  const title = titleMatch ? titleMatch[1] : 'Untitled';

  // Extract goal from blockquote
  const goalMatch = md.match(/\*\*Today's goal:\*\* (.+)/);
  const goal = goalMatch ? goalMatch[1] : '';

  // Extract time
  const timeMatch = md.match(/\*\*Time:\*\* (.+)/);
  const time = timeMatch ? timeMatch[1] : '15 minutes';

  // Split into sections by H2
  const sections = [];
  const sectionRegex = /^## (.+)$/gm;
  const sectionStarts = [];
  let match2;

  while ((match2 = sectionRegex.exec(md)) !== null) {
    sectionStarts.push({ title: match2[1], index: match2.index });
  }

  for (let i = 0; i < sectionStarts.length; i++) {
    const start = sectionStarts[i].index;
    const end = sectionStarts[i + 1]?.index || md.length;
    const sectionMd = md.slice(start, end);
    const sectionTitle = sectionStarts[i].title;

    const section = parseSection(sectionTitle, sectionMd);
    if (section) sections.push(section);
  }

  return { id, week, day, title, goal, time, sections };
}

function parseSection(title, md) {
  const titleLower = title.toLowerCase();

  if (titleLower.includes('warm-up') || titleLower.includes('warm up')) {
    return {
      type: 'warmup',
      content: renderContent(md)
    };
  }

  if (titleLower.includes('new material')) {
    const keyPointMatch = md.match(/#### Key Point\s+> (.+)/);
    return {
      type: 'material',
      title: title,
      content: renderContent(md),
      keyPoint: keyPointMatch ? keyPointMatch[1] : null
    };
  }

  if (titleLower.includes('practice')) {
    return {
      type: 'practice',
      content: renderContent(md),
      exercises: parseExercises(md)
    };
  }

  if (titleLower.includes('conversation')) {
    return {
      type: 'conversation',
      content: renderContent(md),
      prompts: parseConversationPrompts(md)
    };
  }

  if (titleLower.includes('vocab')) {
    return {
      type: 'vocab',
      items: parseVocabTable(md)
    };
  }

  if (titleLower.includes('summary')) {
    const tomorrowMatch = md.match(/\*\*Tomorrow preview:\*\* (.+)/);
    return {
      type: 'summary',
      content: renderContent(md),
      tomorrow: tomorrowMatch ? tomorrowMatch[1] : null
    };
  }

  // Generic section
  return {
    type: 'content',
    title: title,
    content: renderContent(md)
  };
}

function renderContent(md) {
  // Remove the H2 header line
  const content = md.replace(/^## .+$/m, '').trim();
  return marked.parse(content);
}

function parseExercises(md) {
  const exercises = [];
  const exerciseRegex = /### Exercise \d+: (.+)\n([\s\S]+?)(?=### Exercise|\n---|\n## |$)/g;
  let match;

  while ((match = exerciseRegex.exec(md)) !== null) {
    const type = match[1].toLowerCase();
    const content = match[2];

    exercises.push({
      type: detectExerciseType(type),
      instructions: extractInstructions(content),
      items: extractExerciseItems(content, type)
    });
  }

  return exercises;
}

function detectExerciseType(typeStr) {
  if (typeStr.includes('translation') && typeStr.includes('english')) return 'translation-to-english';
  if (typeStr.includes('translation') && typeStr.includes('lojban')) return 'translation-to-lojban';
  if (typeStr.includes('fill')) return 'fill-blank';
  if (typeStr.includes('match')) return 'matching';
  if (typeStr.includes('error') || typeStr.includes('correction')) return 'error-correction';
  if (typeStr.includes('pronunciation')) return 'pronunciation';
  return 'free-production';
}

function extractInstructions(content) {
  const lines = content.trim().split('\n');
  // First non-empty line after exercise header is usually instructions
  for (const line of lines) {
    const trimmed = line.trim();
    if (trimmed && !trimmed.startsWith('>') && !trimmed.match(/^\d+\./)) {
      return trimmed;
    }
  }
  return '';
}

function extractExerciseItems(content, type) {
  const items = [];
  // Match numbered items: 1. `text` or 1. text → answer
  const itemRegex = /^\d+\.\s+(.+?)(?:\s*→\s*(.+))?$/gm;
  let match;

  while ((match = itemRegex.exec(content)) !== null) {
    const prompt = match[1].replace(/`/g, '').trim();
    const answer = match[2] ? match[2].replace(/`/g, '').trim() : null;

    if (answer) {
      // Check for parenthetical answer like (answer: a, i)
      const parenMatch = answer.match(/\(answer:\s*(.+)\)/);
      items.push({
        prompt,
        answer: parenMatch ? parenMatch[1] : answer
      });
    } else {
      // Check if answer is in parentheses within prompt
      const inlineAnswer = prompt.match(/\(answer:\s*(.+)\)/);
      if (inlineAnswer) {
        items.push({
          prompt: prompt.replace(/\s*\(answer:.+\)/, ''),
          answer: inlineAnswer[1]
        });
      } else {
        items.push({ prompt, answer: null });
      }
    }
  }

  return items;
}

function parseConversationPrompts(md) {
  const prompts = [];
  // Match numbered items under "Say aloud:"
  const saySection = md.match(/\*\*Say aloud:\*\*\s*([\s\S]+?)(?=\*\*Answer key|\n---|\n## |$)/);
  if (!saySection) return prompts;

  const itemRegex = /^\d+\.\s+(.+)$/gm;
  let match;

  while ((match = itemRegex.exec(saySection[1])) !== null) {
    prompts.push({
      prompt: match[1].trim(),
      answer: null // Filled from answer key if present
    });
  }

  // Try to match answer key
  const answerSection = md.match(/\*\*Answer key:\*\*\s*([\s\S]+?)(?=\n---|\n## |$)/);
  if (answerSection) {
    const answerRegex = /^\d+\.\s+(.+)$/gm;
    let i = 0;
    while ((match = answerRegex.exec(answerSection[1])) !== null) {
      if (prompts[i]) {
        prompts[i].answer = match[1].replace(/`/g, '').trim();
      }
      i++;
    }
  }

  return prompts;
}

function parseVocabTable(md) {
  const items = [];
  // Match table rows: | `word` | place structure | english |
  const rowRegex = /\|\s*`(\w+)`\s*\|\s*(.+?)\s*\|\s*(.+?)\s*\|/g;
  let match;

  while ((match = rowRegex.exec(md)) !== null) {
    // Skip header row
    if (match[1] === 'Lojban' || match[2].includes('---')) continue;
    items.push({
      lojban: match[1],
      place: match[2].trim(),
      english: match[3].trim()
    });
  }

  return items;
}
```

**Step 2: Run converter**

Run: `cd app && npm run convert`
Expected: "Converting 210 lessons... Done!" and `dist/lessons/` populated with JSON files

**Step 3: Verify a sample JSON**

Run: `cat app/dist/lessons/w01d1.json | head -30`
Expected: Valid JSON with id, week, day, title, sections array

**Step 4: Commit**

```bash
git add app/build/convert.js
git commit -m "feat: add markdown to JSON converter"
```

---

## Task 3: Build Script (esbuild)

**Files:**
- Create: `app/build/bundle.js`

**Step 1: Create the bundle script**

```javascript
const esbuild = require('esbuild');
const fs = require('fs');
const path = require('path');

const SRC = path.join(__dirname, '../src');
const DIST = path.join(__dirname, '../dist');

// Ensure dist exists
fs.mkdirSync(DIST, { recursive: true });

// Bundle JS
esbuild.buildSync({
  entryPoints: [path.join(SRC, 'js/app.js')],
  bundle: true,
  minify: process.env.NODE_ENV === 'production',
  outfile: path.join(DIST, 'app.js'),
  format: 'iife',
});

// Copy static files
const staticFiles = [
  'index.html',
  'css/style.css',
  'manifest.json',
  'sw.js',
];

staticFiles.forEach(file => {
  const src = path.join(SRC, file);
  const dest = path.join(DIST, file);
  if (fs.existsSync(src)) {
    fs.mkdirSync(path.dirname(dest), { recursive: true });
    fs.copyFileSync(src, dest);
    console.log(`Copied: ${file}`);
  }
});

// Copy icons if they exist
const iconsDir = path.join(SRC, 'icons');
if (fs.existsSync(iconsDir)) {
  const destIcons = path.join(DIST, 'icons');
  fs.mkdirSync(destIcons, { recursive: true });
  fs.readdirSync(iconsDir).forEach(file => {
    fs.copyFileSync(path.join(iconsDir, file), path.join(destIcons, file));
  });
  console.log('Copied: icons/');
}

console.log('Build complete!');
```

**Step 2: Commit**

```bash
git add app/build/bundle.js
git commit -m "feat: add esbuild bundle script"
```

---

## Task 4: HTML Shell

**Files:**
- Create: `app/src/index.html`

**Step 1: Create the app shell**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="theme-color" content="#2d3748">
  <title>Lojban Course</title>
  <link rel="stylesheet" href="css/style.css">
  <link rel="manifest" href="manifest.json">
  <link rel="icon" href="icons/icon-192.png">
  <link rel="apple-touch-icon" href="icons/icon-192.png">
</head>
<body>
  <div id="app">
    <header id="header">
      <button id="back-btn" class="icon-btn" aria-label="Back">&#8592;</button>
      <h1 id="header-title">Lojban</h1>
      <button id="settings-btn" class="icon-btn" aria-label="Settings">&#9881;</button>
    </header>

    <div id="progress-bar">
      <div id="progress-fill"></div>
      <span id="progress-text"></span>
    </div>

    <nav id="section-tabs"></nav>

    <main id="content">
      <!-- Dynamic content rendered here -->
    </main>

    <footer id="footer">
      <button id="prev-btn" class="nav-btn">&#8592; Previous</button>
      <button id="next-btn" class="nav-btn primary">Next Day &#8594;</button>
    </footer>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

**Step 2: Commit**

```bash
git add app/src/index.html
git commit -m "feat: add HTML app shell"
```

---

## Task 5: PWA Manifest

**Files:**
- Create: `app/src/manifest.json`
- Create: `app/src/icons/` (placeholder)

**Step 1: Create manifest**

```json
{
  "name": "Lojban 30-Week Course",
  "short_name": "Lojban",
  "description": "Learn Lojban in 15 minutes a day",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2d3748",
  "icons": [
    {
      "src": "icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

**Step 2: Create placeholder icons**

```bash
mkdir -p app/src/icons
# Create simple placeholder SVG converted to PNG (or use any 192x192 and 512x512 PNGs)
```

For now, we'll skip actual icon files and add them later. The app works without them.

**Step 3: Commit**

```bash
git add app/src/manifest.json
git commit -m "feat: add PWA manifest"
```

---

## Task 6: CSS Styles

**Files:**
- Create: `app/src/css/style.css`

**Step 1: Create styles**

```css
/* === Reset & Base === */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:root {
  --bg: #ffffff;
  --bg-alt: #f7f7f8;
  --text: #1a1a1a;
  --text-muted: #6b7280;
  --primary: #2563eb;
  --primary-hover: #1d4ed8;
  --success: #16a34a;
  --error: #dc2626;
  --border: #e5e7eb;
  --shadow: 0 1px 3px rgba(0,0,0,0.1);
  --font-sans: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-mono: 'SF Mono', SFMono-Regular, Consolas, 'Liberation Mono', Menlo, monospace;
  --max-width: 40rem;
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 1rem;
  --space-4: 1.5rem;
  --radius: 6px;
}

html {
  font-size: 16px;
}

body {
  font-family: var(--font-sans);
  color: var(--text);
  background: var(--bg);
  line-height: 1.6;
  min-height: 100vh;
}

/* === Layout === */
#app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-3);
  border-bottom: 1px solid var(--border);
  position: sticky;
  top: 0;
  background: var(--bg);
  z-index: 100;
}

header h1 {
  font-size: 1.1rem;
  font-weight: 600;
}

.icon-btn {
  background: none;
  border: none;
  font-size: 1.25rem;
  cursor: pointer;
  padding: var(--space-2);
  color: var(--text-muted);
  border-radius: var(--radius);
  min-width: 44px;
  min-height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-btn:hover {
  background: var(--bg-alt);
}

.icon-btn.hidden {
  visibility: hidden;
}

/* === Progress Bar === */
#progress-bar {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  padding: var(--space-2) var(--space-3);
  background: var(--bg-alt);
}

#progress-fill {
  flex: 1;
  height: 6px;
  background: var(--border);
  border-radius: 3px;
  overflow: hidden;
}

#progress-fill::after {
  content: '';
  display: block;
  height: 100%;
  width: var(--progress, 0%);
  background: var(--primary);
  border-radius: 3px;
  transition: width 0.3s ease;
}

#progress-text {
  font-size: 0.875rem;
  color: var(--text-muted);
  min-width: 4rem;
  text-align: right;
}

/* === Section Tabs === */
#section-tabs {
  display: flex;
  gap: var(--space-2);
  padding: var(--space-2) var(--space-3);
  overflow-x: auto;
  border-bottom: 1px solid var(--border);
}

#section-tabs:empty {
  display: none;
}

.section-tab {
  padding: var(--space-2) var(--space-3);
  border: none;
  background: var(--bg-alt);
  border-radius: var(--radius);
  font-size: 0.875rem;
  cursor: pointer;
  white-space: nowrap;
  color: var(--text);
}

.section-tab:hover {
  background: var(--border);
}

.section-tab.active {
  background: var(--primary);
  color: white;
}

/* === Main Content === */
main {
  flex: 1;
  padding: var(--space-3);
  max-width: var(--max-width);
  margin: 0 auto;
  width: 100%;
}

/* === Typography === */
h2 {
  font-size: 1.25rem;
  margin: var(--space-4) 0 var(--space-3);
  padding-top: var(--space-3);
  border-top: 1px solid var(--border);
}

h2:first-child {
  margin-top: 0;
  padding-top: 0;
  border-top: none;
}

h3 {
  font-size: 1.1rem;
  margin: var(--space-3) 0 var(--space-2);
}

h4 {
  font-size: 1rem;
  margin: var(--space-2) 0;
}

p {
  margin: var(--space-2) 0;
}

/* === Lojban Text === */
code, .lojban {
  font-family: var(--font-mono);
  background: var(--bg-alt);
  padding: 0.15em 0.4em;
  border-radius: 3px;
  font-size: 0.95em;
}

pre {
  background: var(--bg-alt);
  padding: var(--space-3);
  border-radius: var(--radius);
  overflow-x: auto;
  font-family: var(--font-mono);
  font-size: 0.9rem;
  line-height: 1.5;
}

/* === Tables === */
.table-wrapper {
  overflow-x: auto;
  margin: var(--space-3) 0;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9rem;
}

th, td {
  padding: var(--space-2) var(--space-3);
  text-align: left;
  border-bottom: 1px solid var(--border);
}

th {
  background: var(--bg-alt);
  font-weight: 600;
}

tr:nth-child(even) {
  background: var(--bg-alt);
}

/* === Blockquotes (Key Points) === */
blockquote {
  border-left: 3px solid var(--primary);
  padding: var(--space-3);
  margin: var(--space-3) 0;
  background: var(--bg-alt);
  border-radius: 0 var(--radius) var(--radius) 0;
}

blockquote p {
  margin: 0;
}

/* === Lists === */
ul, ol {
  padding-left: 1.5rem;
  margin: var(--space-2) 0;
}

li {
  margin: var(--space-1) 0;
}

/* === Exercises === */
.exercise {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: var(--space-3);
  margin: var(--space-3) 0;
}

.exercise-instructions {
  font-weight: 500;
  margin-bottom: var(--space-3);
}

.exercise-item {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding: var(--space-2) 0;
  border-bottom: 1px solid var(--border);
}

.exercise-item:last-child {
  border-bottom: none;
}

.exercise-prompt {
  font-family: var(--font-mono);
}

.exercise-input-row {
  display: flex;
  gap: var(--space-2);
}

.exercise-input {
  flex: 1;
  padding: var(--space-2) var(--space-3);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  font-family: var(--font-mono);
  font-size: 1rem;
}

.exercise-input:focus {
  outline: none;
  border-color: var(--primary);
}

.exercise-input.correct {
  border-color: var(--success);
  background: #f0fdf4;
}

.exercise-input.incorrect {
  border-color: var(--error);
  background: #fef2f2;
}

.check-btn {
  padding: var(--space-2) var(--space-3);
  background: var(--primary);
  color: white;
  border: none;
  border-radius: var(--radius);
  cursor: pointer;
  font-weight: 500;
}

.check-btn:hover {
  background: var(--primary-hover);
}

.check-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.exercise-feedback {
  font-size: 0.9rem;
  padding: var(--space-2);
  border-radius: var(--radius);
}

.exercise-feedback.correct {
  color: var(--success);
  background: #f0fdf4;
}

.exercise-feedback.incorrect {
  color: var(--error);
  background: #fef2f2;
}

/* === Reveal Button === */
.reveal-btn {
  background: var(--bg-alt);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: var(--space-2) var(--space-3);
  cursor: pointer;
  font-size: 0.9rem;
}

.reveal-btn:hover {
  background: var(--border);
}

.revealed-answer {
  font-family: var(--font-mono);
  background: var(--bg-alt);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius);
  margin-top: var(--space-2);
}

/* === Matching Exercise === */
.matching-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: var(--space-3);
}

.matching-column {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.matching-item {
  padding: var(--space-2) var(--space-3);
  background: var(--bg-alt);
  border: 2px solid var(--border);
  border-radius: var(--radius);
  cursor: pointer;
  text-align: center;
  transition: all 0.15s ease;
}

.matching-item:hover {
  border-color: var(--primary);
}

.matching-item.selected {
  border-color: var(--primary);
  background: #eff6ff;
}

.matching-item.matched {
  border-color: var(--success);
  background: #f0fdf4;
  cursor: default;
}

.matching-item.wrong {
  animation: shake 0.3s ease;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

/* === Footer Navigation === */
footer {
  display: flex;
  gap: var(--space-3);
  padding: var(--space-3);
  border-top: 1px solid var(--border);
  background: var(--bg);
  position: sticky;
  bottom: 0;
}

.nav-btn {
  flex: 1;
  padding: var(--space-3);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  background: var(--bg);
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  min-height: 44px;
}

.nav-btn:hover {
  background: var(--bg-alt);
}

.nav-btn.primary {
  background: var(--primary);
  color: white;
  border-color: var(--primary);
}

.nav-btn.primary:hover {
  background: var(--primary-hover);
}

.nav-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* === Home Screen === */
.week-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
  gap: var(--space-3);
  padding: var(--space-3) 0;
}

.week-card {
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: var(--space-3);
  cursor: pointer;
  transition: all 0.15s ease;
}

.week-card:hover {
  border-color: var(--primary);
  box-shadow: var(--shadow);
}

.week-card-title {
  font-weight: 600;
  margin-bottom: var(--space-2);
}

.week-card-progress {
  font-size: 0.875rem;
  color: var(--text-muted);
}

.week-card-bar {
  height: 4px;
  background: var(--border);
  border-radius: 2px;
  margin-top: var(--space-2);
  overflow: hidden;
}

.week-card-bar-fill {
  height: 100%;
  background: var(--success);
  border-radius: 2px;
}

/* === Settings Modal === */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
}

.modal {
  background: var(--bg);
  border-radius: var(--radius);
  padding: var(--space-4);
  max-width: 90%;
  width: 20rem;
}

.modal h2 {
  margin: 0 0 var(--space-3);
  padding: 0;
  border: none;
}

.modal-close {
  float: right;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: var(--text-muted);
}

/* === Utilities === */
.hidden {
  display: none !important;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  border: 0;
}
```

**Step 2: Commit**

```bash
git add app/src/css/style.css
git commit -m "feat: add CSS styles"
```

---

## Task 7: Progress Module

**Files:**
- Create: `app/src/js/progress.js`

**Step 1: Create progress module**

```javascript
// progress.js - localStorage wrapper for user progress

const STORAGE_KEY = 'lojban-progress';
const VERSION = 1;

function getDefaultState() {
  return {
    version: VERSION,
    currentLesson: null,
    completed: {},
    exerciseState: {},
    settings: {
      fontSize: 'medium'
    }
  };
}

function load() {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (!stored) return getDefaultState();

    const data = JSON.parse(stored);
    if (data.version !== VERSION) {
      // Handle migrations if needed
      return { ...getDefaultState(), ...data, version: VERSION };
    }
    return data;
  } catch (e) {
    console.error('Failed to load progress:', e);
    return getDefaultState();
  }
}

function save(state) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  } catch (e) {
    console.error('Failed to save progress:', e);
  }
}

let state = load();

export function getCurrentLesson() {
  return state.currentLesson;
}

export function setCurrentLesson(lessonId) {
  state.currentLesson = lessonId;
  save(state);
}

export function isCompleted(lessonId) {
  return !!state.completed[lessonId];
}

export function markCompleted(lessonId) {
  state.completed[lessonId] = true;
  save(state);
}

export function getWeekProgress(weekNum) {
  let completed = 0;
  for (let day = 1; day <= 7; day++) {
    const id = `w${String(weekNum).padStart(2, '0')}d${day}`;
    if (state.completed[id]) completed++;
  }
  return { completed, total: 7 };
}

export function getNextIncompleteDay(weekNum) {
  for (let day = 1; day <= 7; day++) {
    const id = `w${String(weekNum).padStart(2, '0')}d${day}`;
    if (!state.completed[id]) return day;
  }
  return 7; // All done
}

export function getExerciseState(lessonId) {
  return state.exerciseState[lessonId] || {};
}

export function setExerciseState(lessonId, exerciseId, answered, correct) {
  if (!state.exerciseState[lessonId]) {
    state.exerciseState[lessonId] = {};
  }
  state.exerciseState[lessonId][exerciseId] = { answered, correct };
  save(state);
}

export function getSetting(key) {
  return state.settings[key];
}

export function setSetting(key, value) {
  state.settings[key] = value;
  save(state);
}

export function resetProgress() {
  state = getDefaultState();
  save(state);
}

export function getCompletedCount() {
  return Object.keys(state.completed).length;
}
```

**Step 2: Commit**

```bash
git add app/src/js/progress.js
git commit -m "feat: add progress tracking module"
```

---

## Task 8: Router Module

**Files:**
- Create: `app/src/js/router.js`

**Step 1: Create router module**

```javascript
// router.js - Hash-based routing

let currentRoute = null;
let routeHandler = null;

export function init(handler) {
  routeHandler = handler;

  // Listen for hash changes
  window.addEventListener('hashchange', handleHashChange);

  // Handle initial route
  handleHashChange();
}

function handleHashChange() {
  const hash = window.location.hash.slice(1) || ''; // Remove #

  if (hash === currentRoute) return;
  currentRoute = hash;

  const route = parseRoute(hash);
  if (routeHandler) {
    routeHandler(route);
  }
}

function parseRoute(hash) {
  if (!hash || hash === '/') {
    return { type: 'home' };
  }

  // Match lesson: w01d1, w02d3, etc.
  const lessonMatch = hash.match(/^(w\d{2}d\d)$/);
  if (lessonMatch) {
    return { type: 'lesson', id: lessonMatch[1] };
  }

  // Match week: week/1, week/15, etc.
  const weekMatch = hash.match(/^week\/(\d+)$/);
  if (weekMatch) {
    return { type: 'week', weekNum: parseInt(weekMatch[1], 10) };
  }

  // Settings
  if (hash === 'settings') {
    return { type: 'settings' };
  }

  // Unknown route - go home
  return { type: 'home' };
}

export function navigate(path) {
  window.location.hash = path;
}

export function goBack() {
  window.history.back();
}

export function getCurrentRoute() {
  return currentRoute;
}

// Helper to build lesson IDs
export function lessonId(week, day) {
  return `w${String(week).padStart(2, '0')}d${day}`;
}

export function parseLesson(id) {
  const match = id.match(/^w(\d{2})d(\d)$/);
  if (!match) return null;
  return {
    week: parseInt(match[1], 10),
    day: parseInt(match[2], 10)
  };
}

export function nextLesson(id) {
  const parsed = parseLesson(id);
  if (!parsed) return null;

  if (parsed.day < 7) {
    return lessonId(parsed.week, parsed.day + 1);
  } else if (parsed.week < 30) {
    return lessonId(parsed.week + 1, 1);
  }
  return null; // End of course
}

export function prevLesson(id) {
  const parsed = parseLesson(id);
  if (!parsed) return null;

  if (parsed.day > 1) {
    return lessonId(parsed.week, parsed.day - 1);
  } else if (parsed.week > 1) {
    return lessonId(parsed.week - 1, 7);
  }
  return null; // Start of course
}
```

**Step 2: Commit**

```bash
git add app/src/js/router.js
git commit -m "feat: add hash-based router module"
```

---

## Task 9: Exercise Validation Module

**Files:**
- Create: `app/src/js/exercises.js`

**Step 1: Create exercises module**

```javascript
// exercises.js - Exercise validation and interaction

import * as progress from './progress.js';

// Normalize answer for comparison
function normalize(str) {
  return str
    .toLowerCase()
    .trim()
    .replace(/\s+/g, ' ')       // Collapse whitespace
    .replace(/\s*,\s*/g, ', ')  // Normalize comma spacing
    .replace(/^\.+|\.+$/g, '')  // Remove leading/trailing dots
    .replace(/\./g, '');        // Remove dots from names
}

// Check if answer matches (supports array of valid answers)
function checkAnswer(userAnswer, correctAnswer) {
  const normalizedUser = normalize(userAnswer);

  if (Array.isArray(correctAnswer)) {
    return correctAnswer.some(ans => normalize(ans) === normalizedUser);
  }

  return normalize(correctAnswer) === normalizedUser;
}

// Create fill-in-blank exercise UI
export function createFillBlank(item, index, lessonId) {
  const div = document.createElement('div');
  div.className = 'exercise-item';
  div.dataset.index = index;

  const prompt = document.createElement('div');
  prompt.className = 'exercise-prompt';
  prompt.textContent = item.prompt;
  div.appendChild(prompt);

  const inputRow = document.createElement('div');
  inputRow.className = 'exercise-input-row';

  const input = document.createElement('input');
  input.type = 'text';
  input.className = 'exercise-input';
  input.placeholder = 'Your answer...';
  input.autocomplete = 'off';
  input.autocapitalize = 'none';

  const checkBtn = document.createElement('button');
  checkBtn.className = 'check-btn';
  checkBtn.textContent = 'Check';

  inputRow.appendChild(input);
  inputRow.appendChild(checkBtn);
  div.appendChild(inputRow);

  const feedback = document.createElement('div');
  feedback.className = 'exercise-feedback hidden';
  div.appendChild(feedback);

  // Load previous state
  const savedState = progress.getExerciseState(lessonId)[`ex${index}`];
  if (savedState?.answered) {
    input.value = savedState.correct ? item.answer : '';
    input.disabled = true;
    checkBtn.disabled = true;
    input.classList.add(savedState.correct ? 'correct' : 'incorrect');
    if (!savedState.correct) {
      feedback.textContent = `Correct answer: ${item.answer}`;
      feedback.classList.remove('hidden');
      feedback.classList.add('incorrect');
    }
  }

  checkBtn.addEventListener('click', () => {
    const userAnswer = input.value;
    const isCorrect = checkAnswer(userAnswer, item.answer);

    input.disabled = true;
    checkBtn.disabled = true;

    if (isCorrect) {
      input.classList.add('correct');
      feedback.textContent = '✓ Correct!';
      feedback.classList.add('correct');
    } else {
      input.classList.add('incorrect');
      feedback.textContent = `✗ Correct answer: ${Array.isArray(item.answer) ? item.answer[0] : item.answer}`;
      feedback.classList.add('incorrect');
    }
    feedback.classList.remove('hidden');

    progress.setExerciseState(lessonId, `ex${index}`, true, isCorrect);
  });

  // Allow Enter key to check
  input.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && !checkBtn.disabled) {
      checkBtn.click();
    }
  });

  return div;
}

// Create reveal-only exercise (pronunciation, free production)
export function createReveal(item, index) {
  const div = document.createElement('div');
  div.className = 'exercise-item';

  const prompt = document.createElement('div');
  prompt.className = 'exercise-prompt';
  prompt.textContent = item.prompt;
  div.appendChild(prompt);

  if (item.answer) {
    const revealBtn = document.createElement('button');
    revealBtn.className = 'reveal-btn';
    revealBtn.textContent = 'Show Answer';
    div.appendChild(revealBtn);

    const answer = document.createElement('div');
    answer.className = 'revealed-answer hidden';
    answer.textContent = item.answer;
    div.appendChild(answer);

    revealBtn.addEventListener('click', () => {
      answer.classList.remove('hidden');
      revealBtn.classList.add('hidden');
    });
  }

  return div;
}

// Create matching exercise
export function createMatching(items, lessonId) {
  const container = document.createElement('div');
  container.className = 'matching-container';

  // Separate left and right items
  const leftItems = items.map((item, i) => ({ text: item.prompt, index: i }));
  const rightItems = items.map((item, i) => ({ text: item.answer, index: i }));

  // Shuffle right side
  shuffleArray(rightItems);

  const leftCol = document.createElement('div');
  leftCol.className = 'matching-column';

  const rightCol = document.createElement('div');
  rightCol.className = 'matching-column';

  let selectedLeft = null;
  let matchedCount = 0;

  leftItems.forEach(item => {
    const el = document.createElement('div');
    el.className = 'matching-item';
    el.textContent = item.text;
    el.dataset.index = item.index;
    el.dataset.side = 'left';

    el.addEventListener('click', () => handleMatchClick(el, 'left'));
    leftCol.appendChild(el);
  });

  rightItems.forEach(item => {
    const el = document.createElement('div');
    el.className = 'matching-item';
    el.textContent = item.text;
    el.dataset.index = item.index;
    el.dataset.side = 'right';

    el.addEventListener('click', () => handleMatchClick(el, 'right'));
    rightCol.appendChild(el);
  });

  container.appendChild(leftCol);
  container.appendChild(rightCol);

  function handleMatchClick(el, side) {
    if (el.classList.contains('matched')) return;

    if (side === 'left') {
      // Select left item
      leftCol.querySelectorAll('.matching-item').forEach(item => {
        item.classList.remove('selected');
      });
      el.classList.add('selected');
      selectedLeft = el;
    } else if (selectedLeft) {
      // Check match
      const leftIndex = selectedLeft.dataset.index;
      const rightIndex = el.dataset.index;

      if (leftIndex === rightIndex) {
        // Correct match
        selectedLeft.classList.remove('selected');
        selectedLeft.classList.add('matched');
        el.classList.add('matched');
        matchedCount++;
        selectedLeft = null;

        if (matchedCount === items.length) {
          progress.setExerciseState(lessonId, 'matching', true, true);
        }
      } else {
        // Wrong match
        el.classList.add('wrong');
        selectedLeft.classList.add('wrong');
        setTimeout(() => {
          el.classList.remove('wrong');
          selectedLeft.classList.remove('wrong', 'selected');
          selectedLeft = null;
        }, 300);
      }
    }
  }

  return container;
}

function shuffleArray(array) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
}
```

**Step 2: Commit**

```bash
git add app/src/js/exercises.js
git commit -m "feat: add exercise validation module"
```

---

## Task 10: Renderer Module

**Files:**
- Create: `app/src/js/renderer.js`

**Step 1: Create renderer module**

```javascript
// renderer.js - Renders lesson JSON to HTML

import * as exercises from './exercises.js';

export function renderHome(weekProgress) {
  const container = document.createElement('div');

  const title = document.createElement('h2');
  title.textContent = 'Lojban 30-Week Course';
  container.appendChild(title);

  const grid = document.createElement('div');
  grid.className = 'week-grid';

  for (let week = 1; week <= 30; week++) {
    const progress = weekProgress(week);
    const card = document.createElement('div');
    card.className = 'week-card';
    card.dataset.week = week;

    const cardTitle = document.createElement('div');
    cardTitle.className = 'week-card-title';
    cardTitle.textContent = `Week ${week}`;
    card.appendChild(cardTitle);

    const progressText = document.createElement('div');
    progressText.className = 'week-card-progress';
    progressText.textContent = `${progress.completed}/${progress.total} days`;
    card.appendChild(progressText);

    const bar = document.createElement('div');
    bar.className = 'week-card-bar';
    const fill = document.createElement('div');
    fill.className = 'week-card-bar-fill';
    fill.style.width = `${(progress.completed / progress.total) * 100}%`;
    bar.appendChild(fill);
    card.appendChild(bar);

    grid.appendChild(card);
  }

  container.appendChild(grid);
  return container;
}

export function renderLesson(lesson) {
  const container = document.createElement('div');
  container.className = 'lesson';

  // Goal
  if (lesson.goal) {
    const goalBlock = document.createElement('blockquote');
    goalBlock.innerHTML = `<strong>Today's goal:</strong> ${lesson.goal}`;
    container.appendChild(goalBlock);
  }

  // Sections
  lesson.sections.forEach((section, sectionIndex) => {
    const sectionEl = renderSection(section, sectionIndex, lesson.id);
    container.appendChild(sectionEl);
  });

  return container;
}

function renderSection(section, sectionIndex, lessonId) {
  const div = document.createElement('div');
  div.className = 'lesson-section';
  div.id = `section-${sectionIndex}`;

  switch (section.type) {
    case 'warmup':
      div.innerHTML = `<h2>Warm-up</h2>${section.content}`;
      break;

    case 'material':
      div.innerHTML = `<h2>New Material</h2>${section.content}`;
      if (section.keyPoint) {
        const kp = document.createElement('blockquote');
        kp.innerHTML = `<strong>Key Point:</strong> ${section.keyPoint}`;
        div.appendChild(kp);
      }
      break;

    case 'practice':
      div.innerHTML = `<h2>Practice</h2>`;
      if (section.exercises) {
        section.exercises.forEach((exercise, exIndex) => {
          const exEl = renderExercise(exercise, exIndex, lessonId);
          div.appendChild(exEl);
        });
      } else if (section.content) {
        div.innerHTML += section.content;
      }
      break;

    case 'conversation':
      div.innerHTML = `<h2>Conversation Challenge</h2>`;
      if (section.prompts && section.prompts.length > 0) {
        section.prompts.forEach((prompt, i) => {
          const item = exercises.createReveal(
            { prompt: prompt.prompt, answer: prompt.answer },
            `conv-${i}`
          );
          div.appendChild(item);
        });
      } else if (section.content) {
        div.innerHTML += section.content;
      }
      break;

    case 'vocab':
      div.innerHTML = `<h2>Today's Vocab</h2>`;
      const table = renderVocabTable(section.items);
      div.appendChild(table);
      break;

    case 'summary':
      div.innerHTML = `<h2>Summary</h2>${section.content}`;
      if (section.tomorrow) {
        const preview = document.createElement('p');
        preview.innerHTML = `<strong>Tomorrow:</strong> ${section.tomorrow}`;
        div.appendChild(preview);
      }
      break;

    default:
      if (section.title) {
        div.innerHTML = `<h2>${section.title}</h2>`;
      }
      if (section.content) {
        div.innerHTML += section.content;
      }
  }

  return div;
}

function renderExercise(exercise, exIndex, lessonId) {
  const div = document.createElement('div');
  div.className = 'exercise';

  const instructions = document.createElement('div');
  instructions.className = 'exercise-instructions';
  instructions.textContent = exercise.instructions || '';
  div.appendChild(instructions);

  if (!exercise.items || exercise.items.length === 0) {
    return div;
  }

  switch (exercise.type) {
    case 'fill-blank':
    case 'translation-to-english':
    case 'translation-to-lojban':
    case 'error-correction':
      exercise.items.forEach((item, i) => {
        const itemEl = exercises.createFillBlank(
          item,
          `${exIndex}-${i}`,
          lessonId
        );
        div.appendChild(itemEl);
      });
      break;

    case 'matching':
      const matching = exercises.createMatching(exercise.items, lessonId);
      div.appendChild(matching);
      break;

    case 'pronunciation':
    case 'free-production':
    default:
      exercise.items.forEach((item, i) => {
        const itemEl = exercises.createReveal(item, `${exIndex}-${i}`);
        div.appendChild(itemEl);
      });
      break;
  }

  return div;
}

function renderVocabTable(items) {
  const wrapper = document.createElement('div');
  wrapper.className = 'table-wrapper';

  const table = document.createElement('table');
  table.innerHTML = `
    <thead>
      <tr>
        <th>Lojban</th>
        <th>Place Structure</th>
        <th>English</th>
      </tr>
    </thead>
    <tbody></tbody>
  `;

  const tbody = table.querySelector('tbody');
  items.forEach(item => {
    const row = document.createElement('tr');
    row.innerHTML = `
      <td><code>${item.lojban}</code></td>
      <td>${item.place}</td>
      <td>${item.english}</td>
    `;
    tbody.appendChild(row);
  });

  wrapper.appendChild(table);
  return wrapper;
}

export function renderSectionTabs(sections) {
  const tabs = [];

  sections.forEach((section, index) => {
    let label = '';
    switch (section.type) {
      case 'warmup': label = 'Warm-up'; break;
      case 'material': label = 'Learn'; break;
      case 'practice': label = 'Practice'; break;
      case 'conversation': label = 'Speak'; break;
      case 'vocab': label = 'Vocab'; break;
      case 'summary': label = 'Summary'; break;
      default: label = section.title || 'Section';
    }
    tabs.push({ label, index });
  });

  return tabs;
}
```

**Step 2: Commit**

```bash
git add app/src/js/renderer.js
git commit -m "feat: add lesson renderer module"
```

---

## Task 11: Main App Entry Point

**Files:**
- Create: `app/src/js/app.js`

**Step 1: Create main app module**

```javascript
// app.js - Main entry point

import * as router from './router.js';
import * as progress from './progress.js';
import * as renderer from './renderer.js';

// DOM elements
const headerTitle = document.getElementById('header-title');
const backBtn = document.getElementById('back-btn');
const settingsBtn = document.getElementById('settings-btn');
const progressBar = document.getElementById('progress-bar');
const progressFill = document.getElementById('progress-fill');
const progressText = document.getElementById('progress-text');
const sectionTabs = document.getElementById('section-tabs');
const content = document.getElementById('content');
const footer = document.getElementById('footer');
const prevBtn = document.getElementById('prev-btn');
const nextBtn = document.getElementById('next-btn');

// Current state
let currentLesson = null;

// Initialize
function init() {
  router.init(handleRoute);

  // Event listeners
  backBtn.addEventListener('click', handleBack);
  settingsBtn.addEventListener('click', showSettings);
  prevBtn.addEventListener('click', handlePrev);
  nextBtn.addEventListener('click', handleNext);

  // Register service worker
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registered'))
      .catch(err => console.log('SW registration failed:', err));
  }
}

async function handleRoute(route) {
  switch (route.type) {
    case 'home':
      showHome();
      break;
    case 'lesson':
      await showLesson(route.id);
      break;
    case 'week':
      showWeek(route.weekNum);
      break;
    default:
      showHome();
  }
}

function showHome() {
  headerTitle.textContent = 'Lojban Course';
  backBtn.classList.add('hidden');
  progressBar.classList.add('hidden');
  sectionTabs.innerHTML = '';
  footer.classList.add('hidden');

  const homeContent = renderer.renderHome(progress.getWeekProgress);
  content.innerHTML = '';
  content.appendChild(homeContent);

  // Add click handlers to week cards
  content.querySelectorAll('.week-card').forEach(card => {
    card.addEventListener('click', () => {
      const week = parseInt(card.dataset.week, 10);
      const nextDay = progress.getNextIncompleteDay(week);
      router.navigate(router.lessonId(week, nextDay));
    });
  });
}

async function showLesson(lessonId) {
  try {
    const response = await fetch(`/lessons/${lessonId}.json`);
    if (!response.ok) throw new Error('Lesson not found');
    const lesson = await response.json();
    currentLesson = lesson;

    // Update header
    headerTitle.textContent = `Week ${lesson.week} · Day ${lesson.day}`;
    backBtn.classList.remove('hidden');

    // Update progress bar
    progressBar.classList.remove('hidden');
    const dayProgress = (lesson.day / 7) * 100;
    progressFill.style.setProperty('--progress', `${dayProgress}%`);
    progressText.textContent = `Day ${lesson.day}/7`;

    // Render section tabs
    const tabs = renderer.renderSectionTabs(lesson.sections);
    sectionTabs.innerHTML = '';
    tabs.forEach((tab, i) => {
      const btn = document.createElement('button');
      btn.className = 'section-tab';
      btn.textContent = tab.label;
      btn.addEventListener('click', () => {
        document.getElementById(`section-${tab.index}`)?.scrollIntoView({
          behavior: 'smooth'
        });
        sectionTabs.querySelectorAll('.section-tab').forEach(t => t.classList.remove('active'));
        btn.classList.add('active');
      });
      sectionTabs.appendChild(btn);
    });

    // Render lesson content
    const lessonContent = renderer.renderLesson(lesson);
    content.innerHTML = '';
    content.appendChild(lessonContent);

    // Update footer nav
    footer.classList.remove('hidden');
    const prev = router.prevLesson(lessonId);
    const next = router.nextLesson(lessonId);
    prevBtn.disabled = !prev;
    nextBtn.disabled = !next;
    nextBtn.textContent = next ? 'Next Day →' : 'Course Complete!';

    // Save current lesson
    progress.setCurrentLesson(lessonId);

    // Scroll to top
    window.scrollTo(0, 0);

  } catch (err) {
    console.error('Failed to load lesson:', err);
    content.innerHTML = `<p>Failed to load lesson. <a href="#">Go home</a></p>`;
  }
}

function handleBack() {
  if (currentLesson) {
    router.navigate('');
  }
}

function handlePrev() {
  if (!currentLesson) return;
  const prev = router.prevLesson(currentLesson.id);
  if (prev) router.navigate(prev);
}

function handleNext() {
  if (!currentLesson) return;

  // Mark current as complete
  progress.markCompleted(currentLesson.id);

  const next = router.nextLesson(currentLesson.id);
  if (next) {
    router.navigate(next);
  } else {
    // Course complete!
    router.navigate('');
  }
}

function showSettings() {
  const backdrop = document.createElement('div');
  backdrop.className = 'modal-backdrop';

  const modal = document.createElement('div');
  modal.className = 'modal';
  modal.innerHTML = `
    <button class="modal-close">&times;</button>
    <h2>Settings</h2>
    <p>Progress: ${progress.getCompletedCount()}/210 lessons completed</p>
    <button id="reset-btn" style="margin-top: 1rem; color: var(--error);">Reset Progress</button>
  `;

  backdrop.appendChild(modal);
  document.body.appendChild(backdrop);

  modal.querySelector('.modal-close').addEventListener('click', () => {
    backdrop.remove();
  });

  backdrop.addEventListener('click', (e) => {
    if (e.target === backdrop) backdrop.remove();
  });

  modal.querySelector('#reset-btn').addEventListener('click', () => {
    if (confirm('Reset all progress? This cannot be undone.')) {
      progress.resetProgress();
      backdrop.remove();
      showHome();
    }
  });
}

// Start the app
init();
```

**Step 2: Commit**

```bash
git add app/src/js/app.js
git commit -m "feat: add main app entry point"
```

---

## Task 12: Service Worker

**Files:**
- Create: `app/src/sw.js`

**Step 1: Create service worker**

```javascript
// sw.js - Service Worker for offline support

const CACHE_NAME = 'lojban-v1';

// Assets to cache on install
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/app.js',
  '/css/style.css',
  '/manifest.json',
];

// Generate lesson URLs
const LESSON_URLS = [];
for (let week = 1; week <= 30; week++) {
  for (let day = 1; day <= 7; day++) {
    const id = `w${String(week).padStart(2, '0')}d${day}`;
    LESSON_URLS.push(`/lessons/${id}.json`);
  }
}

const ALL_ASSETS = [...STATIC_ASSETS, ...LESSON_URLS];

// Install - cache all assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => {
        console.log('Caching all assets...');
        return cache.addAll(ALL_ASSETS);
      })
      .then(() => self.skipWaiting())
  );
});

// Activate - clean up old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys()
      .then(keys => {
        return Promise.all(
          keys
            .filter(key => key !== CACHE_NAME)
            .map(key => caches.delete(key))
        );
      })
      .then(() => self.clients.claim())
  );
});

// Fetch - cache first, then network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(cached => {
        if (cached) {
          return cached;
        }
        return fetch(event.request)
          .then(response => {
            // Don't cache non-successful responses
            if (!response || response.status !== 200) {
              return response;
            }
            // Cache successful responses
            const responseClone = response.clone();
            caches.open(CACHE_NAME)
              .then(cache => cache.put(event.request, responseClone));
            return response;
          });
      })
  );
});
```

**Step 2: Commit**

```bash
git add app/src/sw.js
git commit -m "feat: add service worker for offline support"
```

---

## Task 13: Integration Test

**Step 1: Run full build**

```bash
cd app && npm run build
```

Expected: No errors, `dist/` populated with all files

**Step 2: Start dev server**

```bash
cd app && npm run dev
```

Expected: Server running at http://localhost:3000

**Step 3: Manual test checklist**

Open http://localhost:3000 in browser:

- [ ] Home screen shows 30 week cards
- [ ] Clicking a week navigates to lesson (e.g., #w01d1)
- [ ] Lesson renders with all sections
- [ ] Section tabs scroll to correct section
- [ ] Fill-blank exercises accept input and validate
- [ ] Reveal buttons show answers
- [ ] Next Day button navigates and marks complete
- [ ] Progress persists after refresh
- [ ] Back button returns to home
- [ ] App works offline (disable network in DevTools)

**Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: integration test fixes"
```

---

## Task 14: Create Icons

**Files:**
- Create: `app/src/icons/icon-192.png`
- Create: `app/src/icons/icon-512.png`

**Step 1: Create simple SVG icon**

Create a simple "Lo" text icon or use any placeholder image. The app works without real icons, but PWA install requires them.

For a quick placeholder, create `app/src/icons/icon.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="512" height="512" viewBox="0 0 512 512">
  <rect width="512" height="512" fill="#2563eb"/>
  <text x="256" y="300" font-family="system-ui" font-size="200" font-weight="bold" fill="white" text-anchor="middle">Lo</text>
</svg>
```

Then convert to PNG (can be done manually or skip for now).

**Step 2: Commit**

```bash
git add app/src/icons/
git commit -m "feat: add PWA icons"
```

---

## Task 15: Deploy to GitHub Pages

**Step 1: Initialize git repo if needed**

```bash
cd /home/ubuntu/lojban
git init
git add -A
git commit -m "Initial commit"
```

**Step 2: Create GitHub repository and push**

```bash
gh repo create lojban-course --public --source=. --push
```

**Step 3: Deploy**

```bash
cd app && npm run deploy
```

Expected: Site live at https://[username].github.io/lojban-course/

**Step 4: Verify deployment**

- [ ] Site loads
- [ ] PWA installable (shows install prompt)
- [ ] Works offline after first load

---

## Summary

| Task | Description |
|------|-------------|
| 1 | Project setup (package.json, deps) |
| 2 | Markdown → JSON converter |
| 3 | esbuild bundle script |
| 4 | HTML app shell |
| 5 | PWA manifest |
| 6 | CSS styles |
| 7 | Progress module |
| 8 | Router module |
| 9 | Exercise validation |
| 10 | Lesson renderer |
| 11 | Main app entry |
| 12 | Service worker |
| 13 | Integration test |
| 14 | Icons |
| 15 | Deploy |

**Total estimated time:** 2-3 hours

**Dependencies between tasks:**
- Tasks 1-6 can run in parallel (setup)
- Tasks 7-11 must be sequential (modules depend on each other)
- Task 12 after Task 11
- Task 13 after all code tasks
- Tasks 14-15 after Task 13
