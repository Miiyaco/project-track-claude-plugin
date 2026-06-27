---
name: project-brief
description: Load project context and give a session brief — checks for index.md, creates it if missing, then reports status
user-invocable: true
argument-hint: <code_folder> <remote_repo> [data_location]
allowed-tools: Read, Write, Bash, Glob, Grep
---

The user wants a status brief before starting a work session. Parse $ARGUMENTS as space-separated values:
- Arg 1: `code_folder` — path to the code / project directory
- Arg 2: `remote_repo` — remote repository URL, or "N/A"
- Arg 3: `data_location` — (optional) path to the data directory; skip if not provided

The index file is always at `<code_folder>/index.md`.

Work through each step silently. Only output at Step 5.

---

## Step 0 — Check for index.md

Check if `<code_folder>/index.md` exists:
```bash
test -f <code_folder>/index.md && echo "exists" || echo "missing"
```

### If it EXISTS:
Continue to Step 1.

### If it is MISSING:
Infer the project title and description from whatever is available (try in this order):
1. `README.md` or `README.rst` in `code_folder` — extract the title and first paragraph
2. `git remote get-url origin` in `code_folder` — use the repo name
3. The folder name itself — convert to a readable title (e.g. `my-project` → "My Project")

Then create `<code_folder>/index.md` using this template:

```markdown
# <Inferred Project Title>

<One-line description of the project inferred from README or folder name.>

---

## Sessions

<!-- Each session is logged below, newest first. -->
<!-- Format: date · goal · what was done · what's next -->

```

After writing the file, tell the user:
> I created `index.md` in your project folder. I inferred the title as "<title>" — let me know if you'd like to change it.

Then continue to Step 1.

---

## Step 1 — Read index.md

Read `<code_folder>/index.md`. Extract:
- Project title and description (from the header)
- The **most recent session entry** (the first `###` block under `## Sessions`) — this is the last session's goal, progress, and next steps
- Any open TODOs or blockers mentioned

If there are no session entries yet, note that this is the first session.

---

## Step 2 — Read memory

Scan for Claude memory files related to this project:
```bash
find ~/.claude/projects/ -path "*/memory/*.md" 2>/dev/null | head -20
```
Read any that match the project name or `code_folder` path. Extract accumulated knowledge, past decisions, and working patterns.

---

## Step 3 — Read git log (if applicable)

```bash
git -C <code_folder> rev-parse --is-inside-work-tree 2>/dev/null && echo "yes" || echo "no"
```

If yes:
```bash
git -C <code_folder> log --oneline -10
git -C <code_folder> status --short
git -C <code_folder> branch --show-current
```

Extract: current branch, last commit (hash + message + date), and a summary of uncommitted changes.

---

## Step 4 — Read pipeline (results/ and scripts/)

```bash
ls -lt <code_folder>/results/ 2>/dev/null | head -10
ls <code_folder>/scripts/ 2>/dev/null
```

Read the most recently modified file in `results/`. Read all files in `scripts/`. Summarise: what scripts exist, what the latest output is, and what stage the pipeline appears to be at.

If `data_location` was provided:
```bash
ls <data_location> 2>/dev/null | head -20
```

---

## Step 5 — Output the status report

Print this report — no waffle, just facts:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PROJECT BRIEF
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Topic          <project title from index.md>
               <one-line description>
Remote         <remote_repo>

── Last Session ─────────────────────────
<If first session: "No previous sessions recorded.">
<Otherwise: paste the most recent session block from index.md verbatim>

── Git ──────────────────────────────────
Branch         <current branch, or "N/A">
Last commit    <hash> — <message> (<date>), or "N/A"
Uncommitted    <"clean" or list of modified/untracked files>

── Pipeline ─────────────────────────────
Scripts        <list of scripts/ files, or "N/A">
Latest output  <most recent file in results/ + one-line description, or "N/A">
Data           <data_location summary, or "N/A">

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Step 6 — Ask what to do, then log the session goal

After the report, ask:

> What do you want to work on this session?

Once the user answers, append a new session entry to the TOP of the `## Sessions` section in `<code_folder>/index.md` (insert after the `## Sessions` heading, before any existing entries):

```markdown
### <YYYY-MM-DD>
**Goal:** <what the user said they want to do>
**Done:** *(to be filled at end of session)*
**Next:** *(to be filled at end of session)*

---
```

Tell the user:
> Session logged in `index.md`. Let's get started.
