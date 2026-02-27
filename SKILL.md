---
name: knowledge
version: 1.0.0
description: |
  Personal knowledge base for capturing, summarizing, organizing, and
  recalling knowledge from URLs, YouTube videos, documents, and files.

  Activate this skill in two scenarios:

  1. Explicit knowledge operations: the user says "save this", "add to
     knowledge", "capture this article", "summarize this video", "sort
     knowledge", "organize categories", "import knowledge", etc.

  2. Proactive recall: the user asks about ANY topic where the knowledge
     base might have relevant entries. This includes general questions
     like "what's a good workout routine", "how should I approach this
     design problem", "what do I know about sleep training", etc. When
     in doubt, check the knowledge base — it is an extension of memory.
     Search it whenever the user's question touches a topic that could
     plausibly have been captured previously.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - WebFetch
  - AskUserQuestion
  - TodoWrite
---

# Knowledge: Capture, Summarize, and Organize

Manage a personal knowledge base that extracts content from artifacts, generates summaries, and organizes entries into fluid categories.

## Configuration

### Knowledge Base Path

The knowledge base path is not hardcoded. On first invocation:

1. **Check agent memory.** Look for a previously stored knowledge base path in the agent's persistent memory or configuration (e.g., MEMORY.md, project config, environment variable `KNOWLEDGE_BASE_PATH`).

2. **If no path is stored**, ask the user where to store the knowledge base. Suggest a sensible default based on the platform:
   - macOS/Linux: `~/Documents/knowledge/` or `~/knowledge/`
   - If the current working directory seems intentional, offer that as an option
   - If the agent has a home or workspace directory, suggest a `knowledge/` subdirectory within it

3. **Save the chosen path** to the agent's persistent memory so it is available across sessions. The mechanism depends on the agent:
   - Claude Code: write to the auto memory directory (e.g., `MEMORY.md`)
   - Other agents: use whatever persistent key-value store or config file is available

4. **On subsequent invocations**, read the stored path from memory and use it. Do not ask again unless the path no longer exists or the user requests a change.

The knowledge base directory is separate from the skill definition. The skill can live anywhere (a skills directory, a plugin repo); the data lives at the configured path.

### Knowledge Index in Agent Memory

The agent's persistent memory should contain a lightweight index of the knowledge base so that category awareness is always in context — even before this skill is activated. This is what makes proactive recall possible.

After any operation that changes the knowledge base structure (adding entries, sorting, creating/splitting/merging/renaming categories), update the index in agent memory. The index should look approximately like this:

```
## Knowledge Base
Path: ~/Documents/knowledge/
Categories:
- fitness: exercise routines, nutrition, recovery
- parenting: child development, sleep training, education approaches
- software-design: architecture patterns, API design, system modeling
- industrial-design: product design, materials, manufacturing
Unsorted: 3 entries
```

The index contains:
- The knowledge base path
- Each category name with a brief description (pulled from `_category.md`)
- The count of unsorted entries

Keep the index concise — it lives in the agent's system prompt and should not exceed ~20 lines. The purpose is awareness, not exhaustive detail. The agent can always read the actual files for specifics.

When categories change (split, merge, rename, create, delete), update the index immediately. This ensures the agent's ambient awareness of what knowledge exists stays current.

### Conventions

- **Date format:** YYYY-MM-DD (ISO 8601 date only)
- **Filename format:** `{descriptive-slug}--{YYYY-MM-DD}.md`
  - Slug: lowercase, hyphenated, 3-6 words describing the content
  - Raw and summary files for the same artifact share the same filename
- **Summary length:** 100-400 words

## Directory Structure

```
{knowledge-base-path}/
  README.md
  CHANGELOG.md
  unsorted/
    raw/
    summary/
  {category}/
    _category.md
    raw/
    summary/
```

Category directories are created on demand when knowledge is sorted into them.

### First-Run Initialization

On first invocation:
1. Resolve the knowledge base path (see Configuration above).
2. If `README.md` does not exist at the knowledge base path, create:
   - `README.md` from the README template at the end of this skill
   - `CHANGELOG.md` with an initial entry
   - `unsorted/raw/` and `unsorted/summary/` directories (use `mkdir -p` via Bash)
3. Save the resolved path to agent memory if not already stored.

### Version Control

If git is available and the knowledge base directory is not already a git repository, initialize one (`git init`). The knowledge base benefits from version control — it provides history for category reorganizations, a safety net for bulk imports, and makes it easy to sync across machines.

After meaningful operations (adding knowledge, sorting, reorganizing), commit the changes with a short descriptive message. Keep commits atomic: one commit per logical operation rather than batching unrelated changes. Do not push unless the user has configured a remote and asks to push.

---

## File Formats

### Raw File

```markdown
---
id: {descriptive-slug}--{YYYY-MM-DD}
title: "{Human-readable title}"
source_type: url | youtube | document | audio | file | text
source: "{URL, file path, or description of origin}"
captured: YYYY-MM-DD
---

{Full extracted content. Preserve original structure where possible.}
```

Rules:
- Preserve content faithfully. Do not editorialize.
- For YouTube transcripts, clean auto-sub artifacts but keep `[HH:MM:SS]` timestamps at paragraph boundaries.
- For URLs, strip navigation, ads, and boilerplate. Keep the article body.

### Summary File

```markdown
---
id: {descriptive-slug}--{YYYY-MM-DD}
title: "{Human-readable title}"
source_type: url | youtube | document | audio | file | text
source: "{URL, file path, or description of origin}"
captured: YYYY-MM-DD
raw: ../raw/{descriptive-slug}--{YYYY-MM-DD}.md
related:
  - {id-of-related-entry}
tags:
  - {tag}
---

## Summary

{2-4 paragraphs capturing the main ideas, arguments, or content.
Write in plain, direct prose.}

## Key Points

- {Bullet point}

## Takeaways

{1-2 sentences on why this matters or how it connects to broader themes.}
```

Rules:
- The `id` matches the raw file's `id` exactly.
- The `raw` field uses a relative path from summary to raw: always `../raw/{filename}`.
- The `related` field lists `id` values (not file paths). Resolve at read time via `Glob("**/summary/{id}.md")`.
- The `tags` field uses lowercase, hyphenated keywords. Check existing tags across the knowledge base (Grep for `^  - ` under `tags:` in summary files) and reuse established tags.
- When generating, read the full raw content first. Capture the core argument, not surface details.

### _category.md

```markdown
---
category: {category-slug}
created: YYYY-MM-DD
description: "{One-sentence scope definition}"
---

## {Category Name}

{1-3 sentences defining what belongs in this category and what does not.
Be specific enough to resolve ambiguity when sorting.}

## Entries

Summaries: `summary/*.md`
Raw content: `raw/*.md`

## Access Notes

{Optional guidance on navigating this category.}
```

### CHANGELOG.md Format

```markdown
# Knowledge Base Changelog

## YYYY-MM-DD

### Added
- [{id}] "{title}" from {source_type} -- placed in `{category}/`

### Moved
- [{id}] moved from `{old-category}/` to `{new-category}/`

### Categories
- Created `{category-name}`: {description}
- Renamed `{old-name}` to `{new-name}`
- Split `{old-name}` into `{new-name-1}` and `{new-name-2}`
- Merged `{name-1}` and `{name-2}` into `{new-name}`
```

Group entries under the current date heading. If today's heading exists, append to it.

---

## Core Abilities

### Ability 1: Create Knowledge from an Artifact

When the user provides a URL, YouTube link, file path, or content to capture:

**Step 1: Detect source type.**
- YouTube URL: contains `youtube.com/watch`, `youtu.be/`, or `youtube.com/shorts/`
- General URL: starts with `http://` or `https://`
- File path: exists on local filesystem
- Pasted text: inline content provided directly

**Step 2: Extract raw content.**

| Source | Method |
|--------|--------|
| URL | `WebFetch(url, "Extract the main article content of this page as plain text")` |
| YouTube | Bash: `yt-dlp --write-auto-sub --sub-lang "en" --skip-download --print title --print description -o "/tmp/yt-%(id)s" "{url}"` then Read the `.vtt` file and clean it (see transcript cleaning below) |
| Local file | `Read(filepath)` |
| Audio | Bash: `whisper "{path}" --output_format txt` if available; otherwise note as placeholder |
| Pasted text | Use directly |

**YouTube transcript cleaning:**
1. Remove VTT headers (`WEBVTT`, `Kind:`, `Language:`)
2. Remove timestamp lines (lines matching `XX:XX:XX.XXX --> XX:XX:XX.XXX`)
3. Remove duplicate consecutive lines
4. Collapse blank lines
5. Insert `[HH:MM:SS]` markers approximately every 60 seconds as paragraph breaks
6. Clean up via Bash pipeline, then read and format the result
7. Clean up temporary files in `/tmp/`

**Step 3: Generate descriptive slug.** Read the extracted content and produce a 3-6 word lowercase hyphenated slug. Examples:
- `deep-work-productivity-strategies--2026-02-27`
- `react-server-components-explained--2026-02-27`
- `infant-sleep-training-methods--2026-02-27`

**Step 4: Check for duplicates.** Grep across `**/raw/*.md` for the source URL in frontmatter. If found, warn the user and ask whether to overwrite or create a suffixed version.

**Step 5: Write the raw file** to `unsorted/raw/{slug}--{date}.md`. Create the directory with `mkdir -p` if needed.

**Step 6: Generate the summary.** Read the full raw content. Produce a summary following the template above. Scan existing summaries (Glob `**/summary/*.md`, spot-read a few) to populate the `related` field and reuse existing tags.

**Step 7: Write the summary file** to `unsorted/summary/{slug}--{date}.md`.

**Step 8: Update CHANGELOG.md.** Append an "Added" entry under today's date.

**Step 9: Report to the user.** Display the title, source, summary key points, file locations, and suggest sorting if appropriate categories exist.

### Ability 2: Sort Unsorted Knowledge

When the user asks to sort knowledge or after adding new knowledge:

1. **Enumerate unsorted entries.** Glob `unsorted/summary/*.md`. Read frontmatter for IDs, titles, tags.

2. **Enumerate existing categories.** Glob `*/_category.md` (excluding `unsorted/`). Read each for scope.

3. **Propose categorization.** For each unsorted entry:
   - If an existing category fits, propose it.
   - If no category fits, propose creating a new one with a slug and description.
   - If the user specified a target, use that.

4. **Confirm with the user** via AskUserQuestion. Present proposed moves as a list. Wait for confirmation.

5. **Execute moves.** For each entry:
   a. Create target category directory structure if needed (`mkdir -p {category}/raw {category}/summary`).
   b. Write `_category.md` if this is a new category.
   c. Move raw file: Bash `mv unsorted/raw/{filename} {category}/raw/{filename}`.
   d. Move summary file: Bash `mv unsorted/summary/{filename} {category}/summary/{filename}`.
   e. The `raw` relative path in the summary (`../raw/{filename}`) remains correct since the sibling structure is preserved.

6. **Update CHANGELOG.md.** Record moves and new categories.

7. **Report to the user.**

### Ability 3: Reorganize Categories

Always confirm with the user before executing any reorganization.

#### Split a Category

1. Read all summary files in the source category.
2. Propose which entries go to which new category. Confirm with user.
3. Create new category directories and `_category.md` files.
4. Move raw and summary files via Bash `mv`.
5. Relative `raw` paths remain correct (sibling structure preserved).
6. `related` fields use IDs (not paths) so no updates needed.
7. Delete the old empty category directory via Bash `rm -r` after confirming it is empty.
8. Update CHANGELOG.md.

#### Merge Categories

1. Create the target category if it does not exist.
2. Move all raw and summary files from source categories to target.
3. Handle filename collisions: append `-2` suffix, update `id` and `raw` path in the affected summary.
4. Write or update target `_category.md`.
5. Remove empty source directories.
6. If any IDs changed, scan all summaries for `related` references to old IDs and update.
7. Update CHANGELOG.md.

#### Rename a Category

1. Rename directory via Bash `mv`.
2. Update `_category.md` frontmatter.
3. No changes needed in files: `raw` paths are relative (still valid), `related` uses IDs (not paths).
4. Update CHANGELOG.md.

### Ability 4: Recall Knowledge

When the user asks a question, references a topic, or requests context that the knowledge base may contain:

1. **Search the knowledge base.** Use Grep across `**/summary/*.md` for keywords related to the query. Also check `**/raw/*.md` if summaries don't surface enough.

2. **Read matching summaries.** For each hit, read the full summary file to assess relevance.

3. **Load into working context.** Read the relevant summary files fully. If deeper detail is needed, read the corresponding raw files. The goal is to internalize the knowledge — absorb it into the current conversation context, embedding, or working memory so it informs subsequent responses.

4. **Synthesize across entries.** When multiple entries are relevant, connect them. Surface patterns, contradictions, or complementary perspectives across the loaded knowledge.

5. **Cite sources.** When drawing on loaded knowledge, reference the entry by title and ID so the user can trace back to the original artifact.

6. **Proactive recall.** When the user is discussing a topic and relevant knowledge exists in the base, proactively load and reference it without being asked. Treat the knowledge base as an extension of memory — if relevant knowledge is there, use it.

The key principle: the knowledge base is not just storage. It is an active resource. When knowledge exists that is relevant to the current conversation, load it and let it inform reasoning. The mechanism for loading (reading files into context, embedding lookup, RAG retrieval) depends on the agent's capabilities — the instruction is to make the knowledge available by whatever means the agent supports.

### Ability 5: Import Existing Knowledge

When the user points to an existing directory, collection of files, or structured knowledge source to onboard into the knowledge base:

1. **Scan the source.** Use Glob and Read to inventory the source directory or file list. Identify:
   - File types present (markdown, text, PDF, HTML, etc.)
   - Any existing organizational structure (subdirectories, naming conventions, tags, metadata)
   - Total number of files and approximate scope

2. **Present a migration plan.** Before touching anything, report:
   - How many files were found and their types
   - Proposed category mapping (if the source has directories/folders, suggest mapping them to knowledge base categories)
   - Any files that can't be processed (binary formats, unsupported types)
   - Ask the user to confirm or adjust the plan

3. **Process each file.** For each importable file:
   a. Read the content.
   b. If the file already has structured metadata (YAML frontmatter, title, date), preserve and adapt it to the knowledge base format.
   c. If the file is unstructured, infer a title from the filename or first heading, and use the file's modification date as the captured date.
   d. Write the raw file using the standard template and naming convention.
   e. Generate a summary following the standard template.
   f. Place files according to the migration plan — either into a mapped category or into `unsorted/`.

4. **Preserve source structure as categories.** If the source has meaningful subdirectories:
   - Map each subdirectory to a knowledge base category.
   - Create `_category.md` files with descriptions inferred from the directory name and contents.
   - If the source structure is flat, place everything in `unsorted/` for manual sorting later.

5. **Build relational links.** After all files are imported:
   - Scan all new summaries for thematic overlap.
   - Populate `related` fields where connections are apparent.
   - Reuse existing tags from the knowledge base; introduce new tags only when necessary.

6. **Update CHANGELOG.md.** Record the import as a batch operation:
   ```
   ### Imported
   - Imported {count} entries from `{source_path}`
   - Categories created: {list}
   - Entries placed in unsorted: {count}
   ```

7. **Report to the user.** Summarize what was imported, where it landed, and flag anything that needs manual attention (duplicates, unprocessable files, ambiguous categorization).

Rules:
- Never modify or delete the source files. The import is a copy operation.
- If an entry with the same content or source already exists in the knowledge base, flag it as a duplicate and skip unless the user says otherwise.
- For large imports (50+ files), process in batches and report progress.
- If the source contains its own README, index, or table of contents, read it first to inform category mapping.

---

## Relational Links

The knowledge base uses a two-layer linking system designed to survive category reorganization:

**Layer 1 -- Raw-to-Summary correlation:**
Raw and summary files share the same filename in sibling `raw/` and `summary/` directories. The summary's `raw` field is always `../raw/{filename}`, which is structurally identical in every category.

**Layer 2 -- Cross-entry references:**
The `related` field in summaries contains `id` values (the filename without `.md`). IDs do not encode category. To resolve an ID to a file, use `Glob("**/summary/{id}.md")`. This makes references stable across category moves.

The only case requiring ID updates is filename collision during a merge.

---

## Category Management Heuristics

Suggest but never execute without user confirmation.

- **Create a new category:** 3+ entries in `unsorted/` share a common theme not covered by existing categories.
- **Split a category:** 15-20+ entries with entries clearly clustering into 2+ distinct subtopics, or the `_category.md` description has become overly broad.
- **Merge categories:** Two categories with fewer than 3 entries each and significantly overlapping scopes.
- **Rename a category:** Name no longer reflects actual contents, or name is ambiguous.

After adding knowledge, briefly note if reorganization might be warranted. Example: "Note: `unsorted/` now has 5 entries about cooking. Consider creating a `cooking` category."

---

## Process Conventions

### Idempotency
Before creating a new raw file, check if a file with the same `id` already exists anywhere (Glob `**/raw/{slug}--{date}.md`). If so, warn the user and ask whether to overwrite or suffix.

### Date Collisions
If two artifacts about similar topics are captured on the same day and produce the same slug, append a numeric suffix: `{slug}-2--{date}.md`.

### Empty Categories
When the last entry leaves a category, ask the user whether to delete the empty category or keep it for future use.

### Tag Consistency
When generating tags, check existing tags across the knowledge base and reuse established tags rather than creating near-duplicates.

### Error Handling
- If WebFetch fails, report the error and ask the user to provide content another way.
- If yt-dlp fails (no subtitles available), note this in the raw file and ask if the user can provide a transcript.
- If a move operation fails, do not update CHANGELOG until the move succeeds.

### Writing Style in Summaries
- Plain, direct prose. No promotional language.
- Active voice. Specific facts over vague characterizations.

---

## README Template

Use this when initializing the knowledge base:

```markdown
# Knowledge Base

Personal knowledge base managed by the `knowledge` skill.

## Structure

- `unsorted/` -- Newly captured knowledge awaiting categorization
- `{category}/` -- Categorized knowledge (e.g., `fitness/`, `parenting/`, `design/`)

Each directory contains:
- `raw/` -- Full extracted content from source artifacts
- `summary/` -- Concise summaries with metadata and relational links

Category directories also contain `_category.md` defining their scope.

## File Naming

`{descriptive-slug}--{YYYY-MM-DD}.md`

Raw and summary files for the same artifact share the same filename.

## Usage

- **Add knowledge:** Provide a URL, YouTube link, file path, or text
- **Sort knowledge:** Sort entries from `unsorted/` into categories
- **Reorganize:** Split, merge, or rename categories
- **Search:** Grep across summaries or raw content
- **Browse:** Read `_category.md` files for scope, then individual summaries

## Changelog

See `CHANGELOG.md` for all additions and structural changes.
```
