---
name: cheatsheet-creator
description: Create a study cheatsheet by extracting and synthesizing key concepts from lecture materials, homework, and past exams. Use this skill whenever a user asks for help making a cheatsheet, study sheet, formula sheet, exam reference sheet, or "one-pager" — especially when they upload lecture slides (PDF/PPTX/DOCX), problem sets, past exams, or scanned/handwritten notes. Trigger this even for casual phrasings like "help me prep for my final," "summarize my course materials into something I can bring to the exam," or "make me a study reference from these slides and homeworks." Tuned for math/science (formulas, proofs, theorems) and CS/engineering (code, algorithms). Processes one file at a time and persists intermediate summaries to disk to avoid blowing past the context window on large courses.
---

# Cheatsheet Creator

Build a focused, exam-ready markdown cheatsheet from a student's course materials.

## Why this skill exists

Students often arrive with a pile of lecture slides, problem sets, and past exams and want them distilled into something they can actually study from or bring into an exam. Two things make this hard:

1. **Memory.** Course materials are big. Twelve lectures × 60 slides + eight homeworks + three past exams will blow past the working context if loaded all at once.
2. **Signal vs. noise.** Lectures cover everything; *exams* reveal what actually matters. The sharpest cheatsheets are anchored in what's been tested.

This skill addresses both: parse one file at a time, persist intermediate summaries to disk, then use past exams and homework as a "what to emphasize" filter applied at the end. The order matters — do not try to do this all at once.

## Workflow

### Step 0: Inventory, categorize, and confirm scope

Before parsing anything, list every input file and confirm with the user which are:

- **Lectures** — slides, notes, scanned handwritten notes, transcripts
- **Homework** — problem sets, ideally with solutions
- **Past exams** — midterms, finals, practice exams, ideally with solutions

**Suggest one of these two organizing conventions** so categorization is unambiguous (and so this skill can be re-run cleanly later):

- **Folder structure**: separate `lectures/`, `homework/`, `exams/` directories.
- **Filename prefixes**: `lecture_03_continuity.pdf`, `hw_05.pdf`, `exam_midterm_2023.pdf` — any consistent prefix works.

If the user's files already follow one of these, proceed silently. If they don't, ask the user to confirm classifications and gently suggest organizing this way next time. Don't reshuffle their files for them.

Misclassifying matters: a homework problem summarized as "lecture content" pollutes the master concept list with one student's wrong-turn intermediate work.

Also confirm with the user:

- **Target size.** Always ask — the user told us they want this configurable each time. "One page front-and-back," "three pages," "exhaustive study guide," etc. This drives how aggressively to compress in the final synthesis step. Note that worked examples (Step 4) count toward this size budget.
- **Cheatsheet rules**, if any. Some exams allow only handwritten cheatsheets, some allow typed. The user might also have their own constraints like "formulas only, no worked examples."
- **Subject/course name**, so the final file can be named sensibly and the writing tuned (heavy LaTeX for a real analysis class vs. heavy code blocks for a data structures class).
- **Save intermediates?** Whether to include the per-lecture summaries, master concept list, exam frequency tally, and worked-examples scratch in the final output folder (under `cheatsheet/intermediate/`). Default to **yes** — they're useful while studying (you can see *why* a concept made it onto the sheet, audit the source attributions, and re-run the synthesis at a different size budget without reparsing). Skip them only if the user explicitly says they don't want them.

Sketch a quick plan ("I'll read 12 lectures, 8 homeworks, 3 past exams, target 2 pages, include worked examples, save intermediates") and confirm before grinding through everything.

### Step 1: Read and summarize each lecture, one at a time

This is the memory-critical step. The pattern is: open one lecture → write a structured summary to disk → drop the raw lecture from working memory → next lecture.

For each lecture file:

1. **Parse it with the right tool.** If a more specific skill is available, use it; otherwise fall back to `file-reading` as the router.
   - Text-based PDF: use the `pdf-reading` skill.
   - Scanned or handwritten PDF: rasterize pages and read them visually (still via `pdf-reading`). Handwritten math is the most error-prone part of the whole pipeline — go slowly and double-check transcribed formulas.
   - PPTX: use the `pptx` skill.
   - DOCX: use the `docx` skill.
   - Standalone images of handwritten notes: read them directly as images.

2. **Write the summary** to `cheatsheet-work/lecture-summaries/<lecture-name>.md`. Use this template:

   ```markdown
   # Lecture: <name or topic>     <!-- source: <filename> -->

   ## Core concepts
   - <concept>: <one-sentence definition>     <!-- src: slide 12 -->

   ## Formulas / theorems / algorithms
   - <name>: <statement, in LaTeX or pseudocode>     <!-- src: slide 17 -->
     - When it applies: <conditions>
     - Common pitfalls: <if mentioned in lecture>

   ## Standard problem-solving structures
   - <problem type>: <bare-bones approach the lecturer taught>     <!-- src: slide 22 -->
     - Example: "To prove a sequence converges: (1) propose a candidate limit L, (2) for any ε > 0, show ∃ N s.t. n > N implies |a_n − L| < ε."

   ## Worked-example patterns
   - <pattern name>: <terse description of approach>     <!-- src: slide 31 -->

   ## Notation and conventions
   - <symbol> = <meaning>
   ```

   The HTML-comment `<!-- src: ... -->` markers exist so Step 4 can produce an expanded version with back-references to source documents. Keep them on every substantive item — slide number, page number, or problem number, depending on the format.

   The "Standard problem-solving structures" section is important and easy to overlook. Lecturers often teach a templated approach to whole categories of problems (the standard ε–δ continuity proof, the standard induction setup, the standard divide-and-conquer recurrence solution). Capture these *as templates*, not just as examples — they go into the final cheatsheet because students reach for them under exam pressure.

3. **Preserve fidelity on the things students actually need.** For math/science: keep theorem statements and formulas verbatim in LaTeX (`$...$` or `$$...$$`). Don't paraphrase a theorem. For CS/engineering: keep algorithms in pseudocode or fenced code blocks — don't reduce a sorting algorithm to prose like "it sorts by partitioning." That loses the thing the student needs to see.

4. **Then move on.** Don't carry the raw lecture into the next iteration. The summary on disk is now the canonical short version of that lecture.

### Step 2: Build a unified concept list

Read just the (small) summary files and assemble one master list of all concepts, formulas, definitions, and algorithms across the course. Save to `cheatsheet-work/master-concepts.md`.

Group related items — all "limit theorems" together, all "graph traversal algorithms" together. This grouping is what gives the final cheatsheet its structure later.

Do **not** start cutting yet, even if the target size is small. Cuts happen after Step 3, because the exams will tell you what to keep.

### Step 3: Mine homework and past exams for caveats and emphasis

This is where the cheatsheet gets sharp. Lectures are democratic — they cover everything. Exams are not. Walk through each homework and exam problem and:

1. Identify which concept(s) from the master list it tests. Record the source location alongside (e.g., `HW3 P5`, `Final 2023 P2`).
2. Note anything a student would only learn by *doing* the problem:
   - Edge cases the lecture glossed over
   - Specific tricks or shortcuts
   - Common wrong approaches (especially when a solution explains why a wrong answer was wrong)
   - Recurring problem patterns ("whenever you see X, try Y first")
   - Formulas that get used over and over vs. ones that appeared once and never returned

3. Annotate the master list with these as `> Caveat:` or `> Tested:` callouts under the relevant concept, with source references. Save the annotated version to `cheatsheet-work/concepts-with-caveats.md`.

4. Build a **frequency tally**: which concepts show up on multiple exams? Save to `cheatsheet-work/exam-frequency.md`. This drives both what to keep (Step 4) and which homework problems become worked examples.

5. **Identify heavily-tested homework problems.** Walk the homework set and flag any HW problem whose underlying concept(s) appear on past exams (especially on multiple exams). These are the strongest candidates for the worked-examples section of the final cheatsheet — they're a representative problem of a class of question the student is likely to see again.

   For each flagged HW problem, capture:
   - The problem statement (verbatim or close)
   - The step-by-step solution, with these rules:
     - **If a solution is provided** in the source materials: extract its *key steps* — not every algebraic manipulation, but the load-bearing moves a student needs to reproduce. Aim for 3–8 steps for a typical problem.
     - **If no solution is provided**: propose a solution and **explicitly mark it `[PROPOSED — verify]`** at the top of the solution. Do not silently invent a solution and present it as authoritative. The student must know which solutions came from their course and which came from this skill.
   - The concept(s) it demonstrates, and a note on which exam concept this maps to.

   Save these to `cheatsheet-work/worked-examples.md`.

Process homeworks and exams the same memory-conservative way as lectures — one file at a time, parse, extract caveats and (for HW) candidate worked examples, move on.

### Step 4: Synthesize the final cheatsheet (two outputs + optional intermediates)

Now produce the final markdown, respecting the size constraint the user gave in Step 0.

**Output folder structure**: everything goes inside a single `cheatsheet/` folder at `/mnt/user-data/outputs/cheatsheet/`.

```
/mnt/user-data/outputs/cheatsheet/
├── <course>-cheatsheet.md            (concise, primary deliverable)
├── <course>-cheatsheet-expanded.md   (with italic source refs)
└── intermediate/                     (only if user opted in — default yes)
    ├── lecture-summaries/
    │   ├── lecture-01.md
    │   └── ...
    ├── master-concepts.md
    ├── concepts-with-caveats.md
    ├── exam-frequency.md
    └── worked-examples.md
```

The two cheatsheet files:

1. `<course>-cheatsheet.md` — the user's primary deliverable. Concise, dense, no source citations cluttering the page.
2. `<course>-cheatsheet-expanded.md` — same content, plus inline source references rendered as **italic markdown**: `*(L3 slide 17)*`, `*(HW2 P5)*`, `*(Final 2023 P3)*`. Italic styling keeps references visible but visually subordinate, so the eye still lands on the actual content first.

The clean way to do this: write the *expanded* version first (carrying through the `<!-- src: ... -->` markers from earlier steps and turning them into italicized parenthetical references like `*(L3 slide 17)*`), then derive the concise version by stripping those italic references entirely. Single source of truth, no drift between the two files.

**Intermediate folder**: if the user opted in (default), copy the contents of `cheatsheet-work/` into `cheatsheet/intermediate/`. If they opted out, only the two cheatsheet files go in `cheatsheet/`. Either way, present *all* the resulting files via `present_files` so the user can see and download them.

**What goes in the cheatsheet** (in roughly this priority order, with the size budget governing how many you can fit):

1. **Concepts, formulas, definitions, theorems, algorithms** — anchored in lecture summaries, prioritized by exam frequency.
2. **Standard problem-solving structures** — the templated approaches captured in lecture summaries (ε–δ proof skeleton, induction template, standard recurrence-solving moves). These are high-value because they're what students reach for under pressure.
3. **Caveats** — pitfalls and tricks mined from homework and exams, attached as inline notes to the concepts they apply to.
4. **Worked examples** — heavily-tested homework problems with step-by-step solutions. **Place these in a dedicated `## Worked Examples` section at the end of the cheatsheet**, not interleaved with concepts. Each example carries concept tags (e.g., `Tags: continuity, ε–δ, intermediate value theorem`) so a student scanning the section can quickly find an example relevant to a concept they're stuck on. Flag clearly when a solution is `[PROPOSED — verify]` rather than student-provided. **These count toward the page count** — when budget is tight, include only the most representative example per concept cluster.

Priority order when cutting to fit:

1. **Always keep**: anything tested on 2+ exams, key formulas, definitions, algorithms, notation conventions, and the standard problem-solving structures for those concepts.
2. **Trim**: prose explanation around formulas; long worked examples can be compressed to their key 3–6 steps.
3. **Cut first**: concepts that appeared in lectures but never on exams or homework, redundant worked examples covering the same pattern, motivating stories.

Special note for courses with only one past exam: the "tested on 2+ exams" threshold is unreachable. Substitute "tested on the one exam plus appears in 2+ homework problems" as the high-priority threshold, and note this in the output so the student knows the prioritization is weaker.

A reasonable structure for the final file (shown here in expanded form, with italic refs):

```markdown
# <Course> Cheatsheet

## <Topic group 1>
**<Concept>**: <terse definition> *(L3 slide 12)*
$$<formula>$$
- *Caveat (HW3 P2):* <pitfall>

**Standard approach for <problem type>:** *(L4 slide 8)*
1. <step>
2. <step>
3. <step>

## <Topic group 2>
...

---

## Worked Examples

### Example 1: <descriptive name> *(HW5 P3, tested in Midterm 2023 P1)*
*Tags: <concept>, <concept>, <concept>*

**Problem:** <statement>
**Solution:**
1. <key step>
2. <key step>
...

### Example 2: <descriptive name> *(HW7 P2, tested in Final 2022 P4)*
*Tags: <concept>, <concept>*

**Problem:** <statement>
**Solution:** `[PROPOSED — verify]`
1. <key step>
2. <key step>
...
```

In `<course>-cheatsheet.md` the italic source references are stripped entirely (so `**Concept**: definition *(L3 slide 12)*` becomes `**Concept**: definition`); in `<course>-cheatsheet-expanded.md` they remain as shown.

Density tips that matter:

- Bold the term names so the eye can scan down the page.
- Math goes in `$...$` or `$$...$$`. Don't render formulas as plain text.
- Code goes in fenced blocks with the language tagged.
- Tables are great for comparing related items — derivative tables, Big-O tables, complexity comparisons, common integrals, transform pairs.
- For tight space budgets, inline comma-separated lists are denser than bullet lists.
- Worked-example solutions: aim for 3–8 numbered steps. More than that and you're transcribing, not summarizing.

When done, present everything via `present_files` — list the concise cheatsheet first (primary deliverable), then the expanded version, then the intermediate files if they were saved.

## Working directory layout

```
cheatsheet-work/
├── lecture-summaries/
│   ├── lecture-01.md
│   ├── lecture-02.md
│   └── ...
├── master-concepts.md
├── concepts-with-caveats.md
├── exam-frequency.md       (concept → exams it appeared on)
└── worked-examples.md      (heavily-tested HW problems with solutions)
```

This layout exists for a reason: partial progress survives. If something goes sideways at lecture 8, lectures 1–7 don't need re-parsing. It also means that re-running the synthesis step with a different size budget (e.g., the user wants to try a one-pager *and* a three-pager) doesn't require redoing any of the parsing work.

If the user opted to save intermediates (default), this entire `cheatsheet-work/` directory gets copied into `cheatsheet/intermediate/` as part of the final deliverable in Step 4.

## Handling tricky inputs

**Handwritten or scanned lectures.** The `pdf-reading` skill can rasterize pages so they can be read visually. For standalone image uploads, read them as images directly. Treat handwritten formulas with extra care — these are the highest-error-rate inputs in the whole pipeline. When unsure about a transcribed symbol, flag it in the summary rather than guessing.

**Lectures without exams or homework.** Skip Step 3 and produce the cheatsheet from the master concept list alone. Tell the user the cheatsheet would be sharper with past exams in case they have any to share.

**Exams without solutions.** Still useful — they tell you which topics were tested even when you don't know the correct answers. Note this limitation in the output so the student knows to double-check those sections.

**Solutions in a separate file from the problems.** Pair them up before mining — the solution often contains the insight that should become a caveat ("note that you have to apply L'Hôpital's twice here, not once").

**Mixed-language materials** (e.g., slides in English, student's notes in another language). Write the cheatsheet in whichever language the user studies in. Ask if unclear.

**Materials from different courses mixed together.** Stop and flag it before continuing. A muddled cheatsheet across two courses is worse than no cheatsheet.

**A single lecture that's too big to summarize in one pass.** Recurse: split it into chunks (by section or by page range), summarize each chunk separately into the same lecture's summary file, then collapse those chunk-summaries into the lecture-level summary at the end.

## What good looks like

The student should be able to open `<course>-cheatsheet.md` and:

- Find any formula or definition in under five seconds, thanks to bolded headwords and topic grouping.
- See, at a glance, which concepts the exams have hit hardest (via caveats and "tested on 2+ exams" markers).
- Have a "standard approach" template for recurring problem types (proof setups, induction skeletons, divide-and-conquer recurrences) right next to the concepts they apply to, so they're not improvising under pressure.
- Flip to the **Worked Examples** section at the end, scan the concept tags, and find a step-by-step solution to a problem of the type they're stuck on — flagged clearly as `[PROPOSED — verify]` if the solution didn't come from their materials.
- Trust that nothing important from the course is missing.
- Edit it easily — it's plain markdown.

And `<course>-cheatsheet-expanded.md` lets them chase any item back to the slide, homework problem, or exam question it came from when something looks unfamiliar or wrong.
