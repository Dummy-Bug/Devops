# Devops Folder — Context for Claude Sessions

This folder is the notes vault for a **live DevOps course** the user is enrolled in. Classes are recorded with OBS, transcribed locally, and turned into permanent notes here.

> [!info] **The rig that produces these notes is documented elsewhere — read it before transcribing anything.**
> `~/Desktop/Transcribe/PROMPT-FOR-CLAUDE.md` is how the transcription pipeline works. `~/Desktop/Transcribe/tracks/devops.md` carries this track's running status, the per-part transcript table, the course-repo caveats and the **playback-speed rules** (recording at 2× destroys a transcript, and the usual density check misses it).
> **This file is about the notes**: what they are for, how they must read, how the folder is laid out.

**Scope of a DevOps session:** this folder and the DevOps track only. RAG, AlgoCamp, Python-Utils, LLD, HLD and the interview-prep tracks are other sessions' work — do not read their files, do not cross-reference them, do not let their status distract this one.

## The course

| | |
|---|---|
| **Format** | Live cohort class with student Q&A. **Wednesday and Saturday, 9:00 pm, 2–2½ hours.** |
| **Language** | ~95% Hindi, ~5% English |
| **Started** | 2026-08-05 |
| **Modules** | The folder list under **Status** is the module list — there is no separate syllabus file |
| **Course repo / lab files** | `github.com/Rohitnegi9/Thunder` → `04 Devops/Lecture <nn>/`. Per-lecture `Notes.pdf` and an Excalidraw export. **Read the caveats in `~/Desktop/Transcribe/tracks/devops.md` before trusting it** — it is a lesson plan rather than a record, and its PDF text layer eats the spaces in commands. |

**The instructor is deliberately not named in this file.** Every note is provenance-free by rule, and his personal details are the single recurring hygiene problem in this track — name, email, GitHub account, employment history and a live access token all appear on screen across the recordings. Keeping him out of here as well means there is nowhere for it to leak from. The per-class warnings under **Status** say which recording carries what.

## Who the notes are for

**The notes are the product.** They get published and read by strangers with zero context — not by someone who attended the class. That single fact drives every rule below.

The user is a backend engineer (Java/Spring, Python/FastAPI) with ~2.6 years' experience, preparing for interviews at a higher band. So the notes have two jobs at once: **teach the thing properly**, and **be defensible in an interview**. A note that lets him say "I set up X" but not "here's why X and not Y, and here's what it cost" has failed at the second job.

---

## Folder layout

- **Folders are numbered in READING order — the order he should study them in.** He renumbered them himself on 2026-08-20 for exactly that reason: the number tells him what to read first. So far reading order and course order agree; **if the course ever teaches a subject out of the order it should be learned, the folder number follows the reading order and the class mapping lives in this file rather than in the note.**
- **Notes INSIDE a folder stay in class order.** Folder number = what to read first; note number = which class it came from.
- A folder is created when its module starts. Gaps in numbering mean a module not yet reached.
- **Images are embedded by absolute vault path** — `![[Devops/01-Module/Images/file.png]]`. Any folder rename must rewrite every embed in the same operation.
> [!important] **One flat folder per subject — these are standing preferences, learned the hard way.**
> - **One folder per subject, not per class.** Three classes of Linux is one subject.
> - **No `00-Syllabus.md` files.** Put currency checks and scope caveats inside the notes they belong to.
> - **Merge rather than split.** A 500–700 line note is fine; a 60-line one is not.
> - **Notes are numbered in CLASS ORDER, not learning order — until he says otherwise.** Reordering for pedagogy is a separate, later decision, and it is his to make.
> - **No source footer, and no exception to the no-provenance rule.** A note ends on its last real idea and mentions no course, class, date, recording or lecture anywhere in it. Which class fed which note is recorded in `TRACK-HISTORY.md` and the track file.
> - **When merging, remove duplication only.** Never compress an explanation to save space — breadth and depth must survive the merge intact.

---

## House style

- **No H1 headings.** Open directly with prose; Obsidian shows the filename as the title.
- **No "Next:" trailer lines.** Each note ends on its own content.
- **Never use italics.** Not asterisks, not underscores, and not for emphasis, a product name, a quoted phrase or an aside. Bold when something must stand out, plain text otherwise — in tables, callouts, blockquotes and headings alike.
- **Problem before solution.** Never open with "X is a tool that…". Open with the situation that makes X necessary, show the naive approach, then break it. Motivate a tool by first showing a simpler tool suffices on the easy case, then breaking it on a harder case.
- **Justify with scale numbers, not adjectives.** Not "this is slow" — "this took 40 minutes per deploy across 12 services". If the lecture gave a number, the note keeps it.
- **Plain English. No jargon before it's introduced.** Every term gets explained at first use.
- **One concept per note.** Split when the content justifies it, not before — folder economy matters.
- **"Guarantees / doesn't guarantee"** framing wherever a tool makes a promise with an edge to it.
- **Full capture** — every example, number, distinction, analogy and warning from the class. But **lecture depth only**: don't bolt on material the class didn't cover. If you add something beyond it, mark it clearly and flag it to the user.

### Callouts are varied rather than uniform

Nine types are in use across this track: `[!important]`, `[!info]`, `[!tip]`, `[!note]`, `[!question]`, `[!warning]`, `[!danger]`, `[!bug]`, `[!failure]`. **A folder should spread across five or more** — a folder that uses two reads flat, which is what `04-Networking/` did across 49 callouts before it was fixed.

**`[!warning]` and `[!danger]` are reserved for genuine danger or a real trap**, never for ordinary emphasis.

### Vary the punctuation

**No single mark does every job.** Set 2026-11-17, after he found the em dash standing in for every kind of join across the vault, about one per hundred words, which reads as monotonous and machine-made. Asides go in commas or parentheses, an explanation or a list goes after a colon, and a second complete thought gets its own sentence. **The em dash stays, for a sharp break or an aside that already contains commas, but a paragraph carries at most one and roughly one join in four is an em dash, not every one.** **A definition hung off a word with a comma is only safe when the sentence has no other commas.** After a list (the order service, MySQL, and a helper, a small companion program…) the comma reads as one more list item, so the definition goes in an em dash or parentheses instead. Removing them without replacing them is the other failure: the sentence still needs its join, so pick the mark that fits.

### Never hard-wrap a paragraph

**This vault renders a single newline as a line break.** Obsidian's **Strict line breaks** setting is off, so a paragraph hard-wrapped at ~100 columns in the source shows up in reading view with ragged breaks mid-sentence — a break after "with", a break after "any of this".

**Write one paragraph as one source line, however long it runs.** The same goes for list items, callout lines and blockquote lines — a `>` line is wrapped by the reader's window, never by you. Code fences, tables and mermaid blocks keep their own line structure and are never touched.

**When reflowing a file that is already wrapped, watch the seam.** A join that loses its space produces `gets packaged.The`, `verifying,installing`, `not inyour code`. Grep `[a-z][.,;][A-Za-z]` outside code blocks afterwards, and prove the reflow changed nothing by comparing token streams with whitespace and `>` markers stripped before writing anything back.

---

## Visuals

**Mermaid over screenshots** for pipelines, architectures, request flows and state machines. Screenshots are for **dashboards, architecture drawings and UI**, where the picture is the point.

**Every conceptual section gets a diagram** unless the concept is genuinely one sentence long.

### Colour carries the meaning — an unstyled diagram is a missed one

**The reference implementation is `07-Docker/`.** Every flowchart there declares the palette as `classDef` and assigns nodes with `class`, rather than repeating a `style` line per node:

```mermaid
classDef mech   fill:#1f4f7a,stroke:#2f6ea6,stroke-width:1.5px,color:#fff
classDef good   fill:#1f6f3f,stroke:#2e9c5a,stroke-width:1.5px,color:#fff
classDef broken fill:#7a1f1f,stroke:#b03030,stroke-width:1.5px,color:#fff
classDef step   fill:#7a5a1f,stroke:#b08a2e,stroke-width:1.5px,color:#fff
classDef actor  fill:#2d333b,stroke:#4b5563,stroke-width:1.5px,color:#fff
classDef inert  fill:#3a3a3a,stroke:#5a5a5a,stroke-width:1.5px,color:#fff
class J,J1 good
```

**One colour means the same thing in every diagram in the vault.** The palette is the one set on 2026-09-12 and it did not change in the conversion — only the mechanism did.

| Class | Fill | What it marks |
|---|---|---|
| `mech` | `#1f4f7a` blue | the mechanism being explained — the thing the diagram is about |
| `good` | `#1f6f3f` green | the working path, the correct outcome |
| `broken` | `#7a1f1f` red | the failure, the attacker, the thing being argued against |
| `step` | `#7a5a1f` amber | an intermediate stage, or a caution |
| `actor` | `#2d333b` dark grey | a neutral participant — a person, a client, an external system |
| `inert` | `#3a3a3a` grey | present but not participating |

**If every node in a diagram ends up in one class, the colour is saying nothing.** That is a signal the diagram has no argument in it — either find the contrast it is meant to show, or leave it unstyled rather than painting it uniformly.

Sequence diagrams take no node styling and are left alone.

### Images

**Prefer an image where it beats a diagram** — a widely reproduced standard illustration such as the OSI stack, or a real timeline with measurements on it such as the TLS 1.3 handshake — and keep mermaid for anything specific to a folder's own running example. Download locally, never hot-link, open and read the file before embedding, and record source, author, licence and any modification in that folder's `Images/CREDITS.md`.

---

## DevOps-specific rules

These are the ones that don't come from the general house style, and they exist because DevOps notes fail differently from concept notes.

### 1. Command provenance — the load-bearing rule

> [!danger] **Never write a command, flag, path or config key that came out of the transcript.**
>
> The instructor types commands rather than dictating them. That typing is silence, and silence is exactly what makes whisper hallucinate. Anything command-shaped in the transcript is either garbled or invented.
>
> Every command in a note comes from **the course repo, the official docs, or a legible frame grab of the terminal** — and then gets sanity-checked. A wrong flag in a published note is worse than a missing one, because the reader runs it.

If a command can't be recovered from any of those sources, **say so in the note** rather than guessing — a `> [!info] The class ran a command here that the recording didn't capture cleanly` callout is honest and costs the reader nothing.

### 2. Version and currency

DevOps tooling moves faster than anything else in this vault, and the class will teach whatever version it teaches. So:

- **State the version** for anything version-sensitive — Kubernetes API versions, Docker Compose v1 vs v2 syntax, GitHub Actions runner images, Terraform providers.
- Each folder carries its **currency check dated at time of writing**, inside the note it belongs to, noting what is likely to drift.
- If the class teaches something already deprecated, note the current form **and** keep the class's version — the reader may be following along with the same course.

### 3. Show the failure, not just the happy path

A DevOps note that only shows the working command teaches nothing durable. Where the class shows an error, a broken build, a pod that won't schedule, a permission denied — **that is the most valuable content in the lecture**. Keep the error text, keep the diagnosis, keep the fix.

### 4. Config blocks are code blocks

YAML, Dockerfiles, HCL, shell — always fenced code blocks with the language tag, never screenshots. Copyable, greppable, and they render properly on the published site. Terminal **output** is also a code block.

**DevOps notes carry no code blocks unless a command is genuinely being taught.** This course has few so far, so config and shell fences stay rare and every one must still come from a verified source.

### 5. Live-class hygiene

- **Student names are stripped; questions and answers are kept.** The Q&A is genuinely good content — keep all of it, attribute none of it. **Grep before finishing.**
- **Neutralise named examples** — if the instructor demos with a personal account, real company name, real repo URL, real IP or real domain, rewrite it to a placeholder that behaves the same way for the point being made.
- **Never publish a credential, token, key, or account ID that appears on screen**, even a demo one.
> [!tip] **The placeholder conventions settled for `04-Networking/` — reuse them, do not reinvent them.**
> Every domain in these notes is one invented brand with its subdomains (`api.`, `admin.`, `manager.`, `blog.`), chosen so the notes read as one continuous example rather than a different placeholder per note. Example IP addresses keep the shape used on the board but with **legal octets** — the ones taught had values above 255, which is a real error to publish. Ports came from `/etc/services` rather than the recording, because the spoken numbers included at least one slip. **A later networking class continues this subject, so the same brand and the same addresses must carry over.**

### 6. Every example is Java
> [!important] **Every example is Java, never Node.js — set 2026-09-19, at his instruction.**
> He knows Java, and a Node.js example is one more thing to decode before the concept lands. The running application is a Spring Boot bookshop at `bookcart.in`, built with Maven: sources under `src/main/java`, tests under `src/test/java`, `pom.xml` at the root, test classes named `*Test.java` so Surefire picks them up. **When the class demonstrates in Node.js, translate the example to Java and Maven rather than carrying the Node version**, and verify every Maven command against the lifecycle docs before writing it. Node.js was removed from every folder on that date. JavaScript mentioned as the language of the browser front end is a different thing and stays.

---

## How to work here

**Derive before writing.** Never create or edit a file until it's explicitly asked for. Propose the note structure in chat, wait for approval, then write. Auto-writing and auto-advancing are the failure mode.

**Read the transcript in full before proposing anything.** Under-reading has been called out before.

**Socratic by default** outside of note-writing — the user answers first, then gets pushed on the assumption that's actually load-bearing.

**Correctness checks are one word** — "correct" or "incorrect", no explanation unless asked.

**Never run git.** The user owns version control, including read-only commands.

**Never launch a background job without explicit consent** — including `transcribe`. Default to telling him the command.

**Any claim about how a language, library or tool behaves gets executed before it is written down**, definitions included — a definition is a claim about behaviour. When something genuinely cannot be run, say so in the note and in chat, and label what is measured against what is reconstructed. The worked example behind this rule is in `~/Desktop/AI/CLAUDE.md` and `~/.claude/CLAUDE.md`; rule 1 above is the DevOps-specific sharpening of it.

---

## Status

Eight folders of notes. **The per-class record, the folder-by-folder history and every dated measurement now live in `TRACK-HISTORY.md`** — read it when you need that detail, and re-measure rather than trusting a number. This track's running status is in `~/Desktop/Transcribe/tracks/devops.md`.
