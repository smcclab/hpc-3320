# COMP3320 .mbz → Static HTML Site — Design

## Goal

Convert the Moodle backup `backup-moodle2-course-68-comp3320_sem2_2024-20260511-1135-nu.mbz` into a self-contained static HTML site at the repo root, so a prospective lecturer (no ANU access) can browse the course by opening `index.html` locally or behind any plain webserver.

Source of truth: the .mbz file. The .mbz itself is gitignored (5.5GB).

## Source layout (inside the .mbz)

The .mbz is a gzipped tar. Relevant entries:

- `moodle_backup.xml` — master manifest.
- `sections/section_<id>/section.xml` — one per Moodle section; provides section number, title, visibility, and ordered list of activity ids.
- `activities/<type>_<id>/module.xml` — per-activity metadata: `<modulename>`, `<sectionid>`, `<visible>`, plus type-specific siblings (`resource.xml`, `page.xml`, `url.xml`, `assign.xml`, etc.).
- `files.xml` — global file table; maps every uploaded file's content hash to its original filename, mimetype, and parent component/contextid.
- `files/<XX>/<sha1>` — file content, stored by SHA1 hash, no extensions on disk.

To resolve "activity 6534" to "an actual PDF":
1. Read `activities/resource_6534/resource.xml` and `inforef.xml` to get the file references.
2. Look those references up in `files.xml` to get `(sha1, original_filename, mimetype)`.
3. Read content from `files/<sha1[:2]>/<sha1>`.

## Output layout (at repo root)

```
index.html              # schedule (top) + categorized lists (bottom)
lectures/               # files from activities matched as lectures
labs/
assessments/
resources/              # everything content-bearing that didn't match the above
pages/                  # standalone HTML for long Moodle pages
assets/
  style.css             # small custom CSS layered on Bootstrap
  <images>              # images referenced from inlined or standalone pages
```

No per-folder index pages — `index.html` is the single entry point.

## Filtering: what makes it into the site

The .mbz contains old-version content the lecturer hid before backup. We respect those hide signals.

**Section inclusion:**
- Include the section if `section.visible == 1`, OR
- Include if the section title is in the allowlist: `{"Final Exam", "Course Review Material"}`.

Allowlist rationale: these two sections are hidden because the lecturer hides them from students, not because they're stale. They're real teaching artefacts a new lecturer would want.

**Module inclusion (within an included section):**
- If the section is visible: include modules with `module.visible == 1` only. This catches per-week cleanup where the lecturer hid an old version of a single resource.
- If the section is in the hidden-but-allowlisted set: include all modules regardless of `module.visible` (the whole section was hidden as a unit).

**Sections explicitly excluded** (hidden and not allowlisted): `Final Exam, 02 March 2023`-style supplementary sections, `Top 10 Performance Ranking for Cloth Code`, `Old Material - Week 1` through `Old Material - Week 7`.

**Content de-duplication:** as a safety net, if two included activities point to files with the same SHA1, the file is copied once and both activity entries in the schedule link to that single copy.

**Audit output:** every excluded section, module, and dedup hit is logged to `filtered.log` (gitignored). I review this log before final commit to catch over-zealous filtering.

## Categorization (which folder a file goes into)

Per activity title, case-insensitive, regex applied in order, first match wins:

| Folder | Regex (any match) |
|---|---|
| `lectures/` | `^lecture\b`, `^lec\s*\d+`, `\bweek\s*\d+\s*(lecture\|slides)\b` |
| `labs/` | `^lab\b`, `^lab\s*\d+`, `^practical\b`, `^tutorial\b` |
| `assessments/` | `^assignment\b`, `^assessment\b`, `^a\d+\b`, `^quiz\b`, `^exam\b`, `^project\b`, `^mid[- ]?sem` |
| `resources/` | (default) |

A `categorization.txt` log lists every activity and the folder it landed in, so misclassifications can be spotted and the regex tweaked before the final commit.

## Filenames

Output filename is `<slugified-activity-title>.<original-extension>`. Slugification: lowercase, ASCII-fold, non-alphanumerics → hyphens, collapse repeat hyphens, trim leading/trailing hyphens. On collision within a folder, append `-2`, `-3`, etc.

The original Moodle filename is preserved in `categorization.txt` for traceability but not used as the on-disk name.

## Moodle page activities (216 of them)

Each Moodle page is HTML content authored in the Moodle editor. For each included page:

1. Strip Moodle-specific markup (e.g. `@@PLUGINFILE@@` placeholders) and rewrite embedded `pluginfile.php` references to relative paths under `assets/`.
2. Measure inner text length (HTML stripped).
3. **If text length ≤ 500 chars:** inline the cleaned HTML into the schedule under the relevant section as a description. No standalone file.
4. **Else:** write `pages/<slug>.html` wrapped in a Bootstrap shell. Link from the schedule with the activity title.

Images referenced inside any page (inlined or standalone) get copied to `assets/` with deduped hash-based names, and the references rewritten.

## Schedule (index.html, top half)

Walk included Moodle sections in their original `<number>` order. For each section, render a Bootstrap card:

- **Section title** as the card header.
- **Inlined page intros** (short pages from that section) as descriptions inside the card.
- **Ordered list of the section's activities**, one line each:
  - `resource` → title linked to the file in its categorized folder.
  - `page` (long) → title linked to `pages/<slug>.html`.
  - `page` (short) → already inlined above; not relisted.
  - `url` → title linked to the external URL (target `_blank`).
  - `assign` → title with a short description (from the assignment XML); link to the spec file if attached.
  - `forum` → `(forum: <name> — discussion content not preserved)`.
  - `feedback` → `(feedback survey: <name> — not preserved)`.
  - `lti` → `(external tool: <name>) — launch URL: <url>` for reference only.

## Categorized lists (index.html, bottom half)

Below the schedule, four anchored sub-sections with simple `<ul>` listings:

- `<h2 id="lectures">Lectures</h2>` — every file in `lectures/`, alphabetical by activity title.
- Same for `labs`, `assessments`, `resources`.

A small in-page nav at the top of `index.html` links to `#schedule`, `#lectures`, `#labs`, `#assessments`, `#resources`.

## Styling

- Bootstrap 5 via CDN (`<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5...">`).
- One small custom CSS file at `assets/style.css` for typography tweaks and section-card spacing.
- No JS dependencies beyond Bootstrap's bundle (used only if any collapsible elements are wanted; otherwise omitted).
- Mobile-friendly via Bootstrap's default container grid.

## Implementation

A single Python 3 script `build.py`, kept local (gitignored along with `_work/` and the .mbz). Not part of the deliverable per "just commit the output, no script".

**Dependencies:** stdlib (`xml.etree.ElementTree`, `shutil`, `re`, `pathlib`, `hashlib`) plus `jinja2` (one pip install).

**Passes:**
1. **Extract:** `tar -xzf <mbz> -C _work/` once.
2. **Parse:** walk `_work/sections/` and `_work/activities/`, build an in-memory `Course` object: `Course(sections=[Section(activities=[Activity(...)])])`. Apply filter rules during parse — excluded items go to a separate list for the audit log.
3. **Copy files:** for each included activity with file content, slugify the title, copy from `_work/files/<XX>/<sha1>` to `<category>/<slug>.<ext>`. Track sha1→path mapping for dedup.
4. **Render:** Jinja2 templates produce `index.html`, `pages/*.html`. Inline-short-page logic and category-list generation happen here.
5. **Audit:** write `filtered.log` and `categorization.txt`.

**Idempotency:** re-running blows away `lectures/`, `labs/`, `assessments/`, `resources/`, `pages/`, `assets/`, and `index.html` and regenerates from `_work/`.

**Repo hygiene.** A `.gitignore` is committed listing the local-only files: `*.mbz`, `_work/`, `_templates/`, `build.py`, `filtered.log`, `categorization.txt`. Only the generated site (index.html, page folders, asset folders) is checked in.

## Verification

After generation, the script self-checks:
1. Every `<a href="...">` in `index.html` and `pages/*.html` resolves to an existing local file (or is an external URL).
2. The count of activity entries rendered in the schedule equals the count of included content-bearing activities in the parse step.
3. Total file count in `lectures/ labs/ assessments/ resources/` equals the count of included file-bearing activities (after dedup).

Mismatches fail the script with a clear diff.

After the script runs, I spot-check by reading `index.html` and a couple of the standalone page files, and by listing each output folder to confirm sensible filenames. Then I serve the site briefly with `python -m http.server` and confirm the home page loads and a sample link resolves. Only then do I ask you to review before committing.

## What is NOT done (per README and user)

- PDFs are not OCR'd or converted to editable text — linked as-is.
- Quizzes, feedback surveys, forums, LTI tools, and gradebook/grading config get placeholder lines, not reconstruction.
- No interactivity beyond static links.
- No backwards-compat for older Moodle backup formats — this script is for this one .mbz.
