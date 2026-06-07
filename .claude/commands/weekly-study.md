# /weekly-study

Generate a deep-dive study document for a week in the 2026 learning plan, add it to the MkDocs site, update the site navigation, and deploy to GitHub Pages.

## Usage

- `/weekly-study` — auto-detects current week from today's date
- `/weekly-study 3` — generate for a specific week number
- `/weekly-study --no-deploy` — generate locally, skip the GitHub Pages deploy

---

## Instructions

You are generating a structured 2-hour study guide as a Markdown file for MkDocs Material. Follow every step below in order.

### Step 1 — Determine the week

1. Read the file `plan.json` in the project root.
2. If the user passed a week number argument (e.g., `3`), use that week.
3. Otherwise, compute the current week: find the entry in `plan.json` whose `date` is the most recent Monday on or before today's date. If today is past the last week's date, use the last week.
4. Extract: `week`, `date`, `topic`, `slug`, `details`, `tags`.

### Step 2 — Check if it already exists

Check whether `docs/weeks/{slug}.md` already exists.
- If it does, ask the user: "Week {N} ({topic}) already exists at `docs/weeks/{slug}.md`. Regenerate it? (yes/no)"
- If the user says no, stop here.

### Step 3 — Generate the study document

Write a comprehensive Markdown study document to `docs/weeks/{slug}.md`.

The document must follow this exact structure:

```markdown
# Week {N} — {topic}

**Week of:** {date formatted as "Month D, YYYY"}
**Estimated study time:** ~2 hours
**Tags:** `tag1` `tag2` ...

---

## Overview
[3-4 paragraphs. Why this topic matters for a senior/staff engineer.
How it connects to the user's AESF stack (middleware, ETL, Salesforce, GCP).
What the reader will be able to do after studying this.]

---

## 1. [First major concept area]
[Deep explanation with prose, tables, code examples]

## 2. [Second major concept area]
...

## 3. [Third major concept area]
...

[Continue through 6-10 sections as needed to cover the topic thoroughly.
Each section should include:
- Clear prose explanation of WHY, not just WHAT
- Code examples in Python where applicable (use the user's stack: FastAPI, SQLAlchemy, etc.)
- Comparison tables where there are trade-offs
- Diagrams using ASCII art or Mermaid code blocks where helpful
- Real-world callouts connecting the concept to AESF/middleware/ETL/Salesforce/GCP]

---

## [N]. Key Concepts Summary
[A structured ASCII tree or table summarizing the mental model for the whole topic]

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** [Question]
...
**20.** [Question]

---

### Answers

??? note "Reveal Answers"

    **1.** [Answer]
    ...
    **20.** [Answer]
```

**Content quality requirements:**
- Target 3,500–5,000 words of body text (excluding quiz answers) — enough for ~2 hours of focused reading
- Every code example must be runnable and use the user's actual stack (Python 3.13, FastAPI, SQLAlchemy, Alembic, httpx, GCP, Kubernetes)
- Explain the WHY and the internals, not just the API surface
- Include at least one "common mistake" or anti-pattern per major section
- Quiz questions must cover a range: conceptual understanding, specific details, applied scenarios, trade-offs, and "what would you do if..." situations
- Answers must be detailed — 2-5 sentences each, not one-liners

### Step 4 — Update mkdocs.yml navigation

Read `mkdocs.yml`. Find the `nav:` section, specifically the `Weeks:` list. Add the new entry in week-number order:

```yaml
    - "Week {N} — {topic}": weeks/{slug}.md
```

If the `Weeks:` key doesn't exist yet, create it under `nav:`.

### Step 5 — Update docs/index.md progress tracker

Read `docs/index.md`. In the Progress Tracker table, find the row for this week and change `⏳ Upcoming` to `✅ Available`, and wrap the topic in a link:

```markdown
| [Week N](weeks/{slug}.md) | {topic} | ✅ Available |
```

### Step 6 — Deploy (unless --no-deploy was passed)

Run the following commands in order from the project root:

```bash
git add docs/weeks/{slug}.md mkdocs.yml docs/index.md
git commit -m "Add Week {N}: {topic}"
git push origin main
.venv/Scripts/mkdocs gh-deploy --force
```

If `gh-deploy` fails because the `gh-pages` branch doesn't exist yet, run it once with `--force` which creates it.

After deploy, tell the user:
- The file was written to `docs/weeks/{slug}.md`
- The site is live at: `https://hussainsajib.github.io/weekly-learning/weeks/{slug}/`
- They can print any page with Ctrl+P for a clean printout

### Step 7 — Summary

Output a brief summary:
- Week number and topic
- Word count of the generated document
- File path
- Live URL
- Reminder that answers are hidden behind the "Reveal Answers" toggle on the site
