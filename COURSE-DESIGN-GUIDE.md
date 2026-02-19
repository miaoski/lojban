# Lojban Course — Daily Lesson Design Guide

This document defines how to produce a detailed daily lesson from a weekly outline.
Use `PROJECT.md` (this file) + `week-NN.md` to compose a full Day N lesson.

---

## How to Use This File

Given a request like "Write the lesson for Week 5, Day 3":

1. Read `PROJECT.md` (this file) for structure, pedagogy, and format rules.
2. Read `week-05.md` for that week's vocab, grammar, refresh lists, and daily outline.
3. Read `PLAN.md` for overall curriculum context (what phase, what came before, what comes next).
4. Compose the daily lesson following the template and rules below.

---

## Lesson Format Template

Every daily lesson follows this structure. Time budget: **15 minutes total**.

```markdown
# Week NN · Day D — [Title]

> **Today's goal:** [One sentence: what the learner can do after this session.]
> **Time:** 15 minutes

---

## Warm-up (2-3 min)

[Flashcard drill or quick recall exercise on prior material.]

---

## New Material (5-7 min)

### [Concept Name]

[Explanation — keep it short, concrete, example-first.]

#### Examples

| Lojban | Gloss | English |
|--------|-------|---------|
| `...`  | ...   | ...     |

#### Key Point

> [One-sentence takeaway in a blockquote.]

---

## Practice (4-5 min)

### Exercise 1: [Type]
[Exercise with blanks, translation, or matching.]

### Exercise 2: [Type]
[Second exercise, different format.]

---

## Conversation Challenge (1-2 min)

**Say aloud:**
- [Prompt 1]
- [Prompt 2]
- [Prompt 3]

---

## Today's Vocab

| Lojban | Place Structure | English |
|--------|----------------|---------|
| `...`  | x1 ... x2 ...  | ...     |

---

## Summary

> [One sentence: what you learned today.]

**Tomorrow preview:** [One sentence: what's coming next.]
```

---

## Pedagogy Rules

### 1. Example First, Rule Second

Never start with an abstract rule. Always show a concrete Lojban sentence first, then explain what's happening. The learner should see the pattern before hearing the name.

**Bad:**
> "The `se` operator swaps x1 and x2 of the selbri."
> Example: `do se tavla mi`

**Good:**
> `mi tavla do` — I talk to you.
> `do se tavla mi` — You are talked-to by me. (Same event, different focus.)
> What happened? `se` swapped who's in x1. Now the listener is the subject.

### 2. Place Structure Always Shown

Every new gismu must be introduced with its full place structure, not just a one-word English gloss.

**Bad:**
> `dunda` — give

**Good:**
> `dunda` — x1 gives x2 (gift) to x3 (recipient)

### 3. Gloss Column in Examples

Every Lojban example sentence gets a word-by-word gloss AND a natural English translation.

| Lojban | Gloss | English |
|--------|-------|---------|
| `mi dunda la .cukta. la .alis.` | I give [name book] [name Alice] | I give the book to Alice. |

### 4. Contrast Pairs for Grammar

When teaching a grammar point, show a minimal pair — two sentences that differ by exactly one thing.

| Lojban | English | What changed |
|--------|---------|-------------|
| `mi tavla do` | I talk to you | normal |
| `do se tavla mi` | You are talked-to by me | `se` swapped x1↔x2 |

### 5. Error Examples

In drill/practice weeks and for tricky grammar, include a deliberate wrong example marked with ✗, and the corrected version marked with ✓.

- ✗ `mi dunda la .alis. la .cukta.` — (wrong: Alice in x2 = gift?!)
- ✓ `mi dunda la .cukta. la .alis.` — I give the book to Alice.

### 6. Spaced Repetition

Every lesson starts with a warm-up that reviews prior material. Rules for what to refresh:

| Lesson | What to refresh |
|--------|----------------|
| Day 1-2 of any week | Last 2 weeks' vocab + grammar |
| Day 3-5 of any week | This week's earlier days + a random older week |
| Day 6-7 of any week | This week's full content + random sample from all prior |

### 7. Conversation Challenge Format

The conversation challenge must be things the learner says aloud (not reads silently). Give prompts in English; the learner produces Lojban.

**Format:**
- English prompt → (learner says Lojban) → answer key below

Example:
> **Say:** "I give the book to Alice."
> **Answer:** `mi dunda la .cukta. la .alis.`

### 8. Progressive Difficulty Within Each Day

Each lesson moves from recognition → guided production → free production:

1. **Warm-up:** Recognition (flashcards, match meaning)
2. **New material:** Comprehension (read examples, understand rules)
3. **Practice:** Guided production (fill blanks, translate with hints)
4. **Conversation:** Free production (create sentences from English prompts)

---

## Exercise Types

Use these exercise formats. Vary them across days to prevent monotony.

### Type A: Translation (Lojban → English)

> Translate to English:
> 1. `mi klama la .paris.`
> 2. `do citka la .plise.`

### Type B: Translation (English → Lojban)

> Translate to Lojban:
> 1. I see the dog.
> 2. Alice gives the book to Tom.

### Type C: Fill the Blank

> Fill in the missing word:
> 1. `mi ___ do` (I love you) → `prami`
> 2. `do ___ la .paris.` (You go to Paris) → `klama`

### Type D: Place Structure Drill

> Fill all places of `klama`:
> - x1 (goer): ___
> - x2 (destination): ___
> - x3 (origin): ___
> - x4 (route): ___
> - x5 (vehicle): ___
> Full sentence: ___

### Type E: Conversion Drill

> Rewrite using `se`:
> - `mi tavla do` → ___
>
> Rewrite using `fa/fe`:
> - `do se tavla mi` → ___

### Type F: Tanru Building

> Combine to make a tanru:
> - blue + dog → ___
> - fast + runner → ___
> What is the place structure of each?

### Type G: Error Correction

> Find and fix the error:
> - ✗ `mi dunda la .alis. la .cukta.`
> - ✓ ___

### Type H: Matching

> Match the Lojban to the English:
> 1. `prami` — a. go
> 2. `klama` — b. love
> 3. `tavla` — c. talk

### Type I: Free Production

> Describe your morning in 3 Lojban sentences. Use at least one tense marker.

### Type J: Dialogue Completion

> Complete the dialogue:
> A: `coi .i xu do klama la .paris.`
> B: `___` (answer: yes, I'm going)

---

## Vocabulary Presentation Rules

### New Gismu Format

Every new gismu appears with:

1. **The word** in bold
2. **Full place structure** with labeled slots
3. **One-word English gloss** in parentheses
4. **One example sentence** using the word
5. **Pronunciation note** only if non-obvious

```markdown
**dunda** — x1 gives x2 (gift) to x3 (recipient)
- `mi dunda la .cukta. la .alis.` — I give the book to Alice.
```

### New Cmavo Format

Cmavo appear with:

1. **The word** in bold
2. **Function description**
3. **Minimal pair** showing effect

```markdown
**se** — swaps x1 and x2 of the following selbri
- `mi tavla do` → `do se tavla mi` (You are talked-to by me)
```

### Vocab Review Format

In warm-ups, refreshed vocab appears as rapid-fire flashcards:

```
Lojban → English (cover right side, test yourself)
prami  → love (x1 loves x2)
klama  → go (x1 goes to x2 from x3...)
tavla  → talk (x1 talks to x2 about x3)
```

---

## Grammar Explanation Rules

### Length

- New concept explanations: **3-5 sentences maximum** for core idea
- One concept per day (never two unrelated grammar points)
- Additional detail only in later days when revisiting

### Structure

1. **Show** an example sentence
2. **Name** the concept (one sentence)
3. **Explain** the mechanism (1-2 sentences)
4. **Contrast** with what the learner already knows (minimal pair)
5. **Summarize** in a blockquote takeaway

### Lojban-Specific Notation

- Place structures: `x1 klama x2 x3 x4 x5` (always use x1/x2/x3 notation)
- Selbri in isolation: `klama` (backtick code format)
- Full bridi examples: `mi klama la .paris.` (backtick code format)
- Translations: always after an em-dash: `mi klama` — I go
- Cmavo glosses: always in parentheses: `se` (swap x1↔x2)

---

## Day Type Patterns

Different days within a week serve different functions:

| Day | Type | Purpose |
|-----|------|---------|
| 1 | Introduction | First exposure to the week's core concept + 2-3 new vocab |
| 2 | Deepening | Second angle on the concept + 2-3 new vocab |
| 3 | Extension | Add complexity or a related sub-concept + 2 new vocab |
| 4 | Drill | Intensive practice on Days 1-3 material + remaining vocab |
| 5 | Integration | Combine this week's concept with prior grammar |
| 6 | Conversation | Apply everything in realistic dialogue scenarios |
| 7 | Review | Full-week review: vocab quiz, grammar summary, self-test |

### Review/Practice Week Days (Weeks 9, 14, 30)

In review weeks, each day covers a different subtopic from the phase:

| Day | Type | Purpose |
|-----|------|---------|
| 1-5 | Targeted review | One major concept per day, with drills |
| 6 | Mixed challenge | Exercises combining all reviewed concepts |
| 7 | Self-assessment | Identify weak spots, plan extra practice |

### Fluency Week Days (Weeks 27-28)

In fluency weeks, each day is a scenario:

| Day | Type | Purpose |
|-----|------|---------|
| 1-5 | Scenario | One real-life situation per day (greetings, shopping, etc.) |
| 6 | Extended production | Write or speak for the full 15 minutes |
| 7 | Reflection | Review output, note errors, celebrate progress |

---

## Tone & Style

- **Direct and encouraging.** No filler. No "Great job!" fluff.
- **Honest about difficulty.** "This is hard. Here's why. Here's how to get through it."
- **Lojban-first.** Show Lojban before English whenever possible.
- **No jargon without definition.** First use of any term (bridi, selbri, sumti, etc.) must include a parenthetical definition.
- **Consistent formatting.** Use the template. Don't improvise structure.

---

## Content Boundaries

### What a daily lesson INCLUDES:
- One grammar concept (or drill/review)
- 1-4 new vocabulary items (spread across the week to total 8-12)
- Warm-up on prior material
- Practice exercises (2-3)
- One conversation challenge
- Preview of tomorrow

### What a daily lesson DOES NOT include:
- Cultural or historical commentary about Lojban
- Comparisons to other conlangs
- Meta-discussion about language learning theory
- Links to external resources (those are in PLAN.md)
- Audio/video references (text-only course)

---

## File Naming

Daily lesson files, when produced, go in a `lessons/` directory:

```
lessons/
  w01d1.md   ← Week 1, Day 1
  w01d2.md   ← Week 1, Day 2
  ...
  w30d7.md   ← Week 30, Day 7
```

Total: 210 lesson files (30 weeks × 7 days).

---

## Checklist: Before Submitting a Lesson

- [ ] Total time is ≤ 15 minutes (timed by reading + doing exercises aloud)
- [ ] Warm-up reviews material from a prior week
- [ ] New material shows example before rule
- [ ] Every new gismu has full place structure
- [ ] Every example has gloss + English translation
- [ ] Practice has at least 2 exercises of different types
- [ ] Conversation challenge requires speaking aloud
- [ ] Tomorrow preview is included
- [ ] No more than 4 new vocab items introduced
- [ ] Formatting matches the template exactly
