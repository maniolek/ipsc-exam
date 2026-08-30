# ipsc-exam

## What this project is

A single-page, static quiz app for studying IPSC shooting-sport rulebooks
(currently Polish IPSC Handgun rules). No build step, no backend — `index.html`
is the entire app; question banks live in sibling `*.json` files it fetches
at runtime.

- **Live app:** `index.html` (open via a local HTTP server, or GitHub Pages —
  it uses `fetch()`, so it will *not* work opened directly as a `file://` URL)
- **Question pools:** one JSON file per rulebook, e.g. `pistol-012026.json`
  (IPSC Handgun rules, January 2026 edition, 308 questions)

### Flow the app implements

1. **Configure screen** — pick a question pool (radio list), pick which
   chapters/annexes to include (checkboxes, all selected by default), pick
   how many questions (`Wszystkie` / 5 / 10 / 20).
2. **Quiz screen** — one question at a time, 4 answer options with their
   display order shuffled per question per session, immediate feedback
   showing the quoted rule text after answering.
3. **Results screen** — score, a per-question table, and a **"Powtórz błędne
   pytania"** button that restarts the quiz using only the missed questions
   (recursively — retrying a retry narrows further).

## Data file schema

Every pool file has this shape:

```json
{
  "meta": {
    "title": "Tor Przepisów IPSC",
    "rulesTitle": "Przepisy – Pistolet Dynamiczny IPSC",
    "rulesFile": "pistolet-ipsc-2026.pdf",
    "rulesPublisher": "PZSS / IPSC, tłum. Janusz Kuliś",
    "publicationDate": "2026-01",
    "publicationNote": "Wydanie: styczeń 2026",
    "language": "pl",
    "generated": "2026-08-30",
    "count": 308,
    "coverage": {
      "coveredProvisions": 302,
      "totalProvisions": 891,
      "percent": 34,
      "note": "explanation of the methodology, see below"
    },
    "chapterNames": {
      "Rozdział 1": "Projektowanie torów",
      "Załącznik D1": "Klasa Open"
    }
  },
  "questions": [
    {
      "id": 1,
      "chapter": "Rozdział 1",
      "q": "Question text ending in ?",
      "o": ["option A", "option B", "option C", "option D"],
      "c": 0,
      "ref": "przepis 1.1.3",
      "refText": "Faithful quote or tight paraphrase of the actual rule text."
    }
  ]
}
```

Field notes:
- `id` — unique within the file; sequential, no gaps needed elsewhere.
- `chapter` — **the source-filter grouping key**. The UI builds its
  chapter/annex checkbox list directly from the distinct values of this
  field, sorted naturally (see "Chapter naming" below). Get this wrong and
  the same section splits into two checkboxes.
- `o` — exactly 4 options, in any order (the UI shuffles display order
  itself; storage order doesn't matter and isn't shown to users).
- `c` — index (0-3) into `o` of the correct answer.
- `ref` — short citation shown as a badge before the question and in the
  results table: `"przepis 5.6.3.5"`, `"Załącznik D1, pkt 15"`,
  `"Glosariusz, przepis 12.5"`. Keep the format consistent with existing
  entries — the coverage script (below) parses this field.
- `refText` — shown to the user after they answer, quoted with „ … ”. Should
  be a faithful quote (or a tight, non-misleading paraphrase) of the actual
  rule paragraph, not a restatement of the question.
- `meta.chapterNames` — maps every `chapter` code used in the file to a
  short human-readable title (e.g. `"Rozdział 1"` → `"Projektowanie torów"`,
  `"Załącznik D1"` → `"Klasa Open"`). The chapter-selection checklist shows
  this title as the primary label with the bare code underneath, so a new
  pool needs an entry here for **every** distinct `chapter` value it uses —
  a missing entry just falls back to showing the bare code, it won't break
  anything, but it's worse for the user.

### Chapter naming (avoid this bug)

Use **one canonical label per section**, never a label with a parenthetical
suffix for only *some* of that section's questions:

- Chapters: `"Rozdział 1"` … `"Rozdział 12"`.
- The glossary (przepis 12.5) is its own bucket: `"Rozdział 12 (Glosariusz)"`.
- Annexes: `"Załącznik A1"`, `"Załącznik D4"`, `"Załącznik D4a"`, etc. —
  bare annex code only. Do **not** write `"Załącznik D1 (Open)"` for some
  D1 questions and `"Załącznik D1"` for others; pick one and use it for
  every question from that annex. (This exact mistake happened once during
  the first big expansion and had to be cleaned up — see git history around
  "Normalized inconsistent annex chapter labels".)

Before shipping a new/updated pool file, sanity-check this with:

```bash
python3 -c "
import json
from collections import Counter
d = json.load(open('pistol-012026.json', encoding='utf-8'))
for k, v in sorted(Counter(q['chapter'] for q in d['questions']).items()):
    print(v, k)
"
```
Eyeball the list for near-duplicate labels (same annex/chapter, different string).

## Registering a pool in the UI

`index.html` has a `POOLS` array (search for `const POOLS = [`). Each entry
is just `{ id, name, file }` — everything else (title, coverage, chapter
list, per-chapter question counts) is read from the pool's own JSON at
runtime, so adding a second pool is a one-line addition to that array plus
dropping the new `*.json` file next to `index.html`. Nothing else in
`index.html` needs to change to support a new pool.

File naming convention so far: `<discipline>-<MMYYYY>.json` matching the
rulebook edition, e.g. `pistol-012026.json` for the January 2026 pistol
rules. Follow this pattern for new disciplines/editions (e.g.
`rifle-MMYYYY.json`, `shotgun-MMYYYY.json`, or `pistol-<newdate>.json` for a
future revision).

## Processing a new rules/regulations file into a question pool

This is the repeatable playbook for "process the rules file and find all
the questions" — whether it's a revision of the existing IPSC pistol rules
or a new discipline (rifle, shotgun) or a different regulation entirely.

1. **Get the source text.** If given a PDF, extract it with
   `pdftotext -layout <file>.pdf <file>.txt` (install `poppler-utils` if
   missing). Work from the extracted text, not by skimming the PDF visually
   — page-by-page skimming misses sub-clauses.

2. **Read the entire document, chapter by chapter, in order.** Do not skip
   chapters that look thin or skip straight to annexes — every chapter
   fully covered means the chapter-filter feature stays meaningful (a
   chapter with zero questions can't be a useful filter). If a chapter
   truly can't be read (e.g. its text extraction was accidentally skipped),
   note it explicitly rather than silently omitting it — this happened once
   with "Rozdział 3" in the current pistol pool (it's genuinely absent) and
   should be revisited if that chapter turns out to have substantive rules.

3. **Author one question per distinct fact/rule**, not per chapter — aim
   for granular coverage of numbers, thresholds, definitions, procedures,
   and specific prohibitions/permissions, not vague summaries. Match the
   existing tone: direct factual questions, 4 plausible options (one
   clearly correct once you know the rule, but the distractors should be
   real values/concepts from the document, not throwaway jokes), and
   `refText` that quotes the actual rule closely enough that a reader can
   verify the answer against it.

4. **Compute the coverage metric** the same way the existing file's
   `meta.coverage` was computed, so the numbers stay comparable across
   pools:
   - **Denominator** (`totalProvisions`): count distinct numbered
     paragraphs in the source — regex for lines starting with a decimal
     rule number (`\d+\.\d+(\.\d+){0,3}`) in the main body, distinct
     `(annex_code, item_number)` pairs for numbered items inside each
     annex section, plus one count per distinct glossary term if the
     document has a glossary. Sum these three.
   - **Numerator** (`coveredProvisions`): parse the `ref` field of every
     question in the pool, normalize it back to the same identifiers
     (split compound ranges like `"5.6.1.1–5.6.1.2"` on the en-dash into
     two), and count the distinct set that actually matches something in
     the denominator's set. For glossary questions (all sharing one
     section-level `ref`), match by searching for each known glossary
     term's name inside the question text instead.
   - `percent = round(100 * coveredProvisions / totalProvisions)`.
   - This is deliberately an **approximate, transparently-computed**
     metric (label it as such in `coverage.note`) — it's meant to give the
     user a rough sense of how thorough a pool is, not a precise legal
     audit. Don't hand-wave a round number instead of running the count.

5. **Assemble `meta`** with all the fields in the schema above — in
   particular `rulesFile` (the source file's name, so the UI can show
   *which* document this pool is testing) and `publicationNote` (a short,
   human-readable edition/date string — this is what's shown on the
   pool-select screen next to the coverage badge).

6. **Validate before wiring it up:**
   ```bash
   python3 -c "
   import json
   d = json.load(open('<new-file>.json', encoding='utf-8'))
   qs = d['questions']
   assert len(qs) == d['meta']['count']
   assert len({q['id'] for q in qs}) == len(qs)
   for q in qs:
       assert len(q['o']) == 4 and 0 <= q['c'] <= 3
       assert q['ref'] and q['refText'] and q['chapter']
   print('OK', len(qs), 'questions')
   "
   ```

7. **Add the pool to `POOLS` in `index.html`**, then smoke-test the whole
   flow before pushing:
   ```bash
   python3 -m http.server 8793 &
   ```
   Then either open it in a browser, or drive it headlessly — Playwright
   with Chromium is preinstalled in this environment
   (`/opt/pw-browsers/chromium-*/chrome-linux/chrome`,
   `NODE_PATH=$(npm root -g) node your_test.js` since `playwright` is only
   installed globally, not as a project dependency). At minimum verify:
   - the new pool appears and its detail panel shows title/file/date/coverage
   - its chapter checkboxes render, all checked by default
   - unchecking everything disables Start; a partial selection filters
     `QUESTIONS` to only that chapter's questions
   - a full run reaches the results screen with a plausible score, and
     "Powtórz błędne pytania" narrows to just the missed ones
   - no console/page errors
   Kill the http.server afterward.

8. **Commit and push.** This repo's established practice for this project
   is to push straight to `master` when asked (fast-forward from the
   working branch — check with
   `git merge-base --is-ancestor origin/master <branch>` first; if it's not
   a fast-forward, stop and reconcile rather than force-pushing).

## Environment notes

- `pdftotext` (poppler-utils) may need installing: `apt-get install -y poppler-utils`.
- Playwright's Chromium is preinstalled; do **not** run `playwright install`.
  Launch with `executablePath: '/opt/pw-browsers/chromium-<ver>/chrome-linux/chrome'`
  (find the exact path with `find /opt/pw-browsers -iname chrome -type f`).
- The Node `playwright` package itself is global, not a project dependency —
  prefix ad-hoc test scripts with `NODE_PATH=$(npm root -g) node script.js`.
