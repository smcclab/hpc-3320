# COMP3320 High Performance Computing — Course Snapshot

A static snapshot of the **Semester 2, 2024** offering of COMP3320 / 6464 at ANU,
extracted from a Moodle backup for sharing with someone outside ANU. Open
`index.html` in any browser, or serve the repo with any static webserver, e.g.:

```bash
python -m http.server 8000
```

then visit <http://localhost:8000/>.

## Layout

| Folder            | What's in it                                                        |
|-------------------|---------------------------------------------------------------------|
| `index.html`      | Week-by-week schedule, plus categorized listings at the bottom.     |
| `lectures/`       | Lecture slide decks (PDFs). Video recordings are intentionally not included — see *Caveats*. |
| `labs/`           | Empty in this snapshot — see *Labs* below.                          |
| `assessments/`    | Past final exams (2012–2022) and past mid-semester exams (2018–2022). |
| `resources/`      | Course readings (OpenMP, cache coherence, floating point, etc.).    |
| `pages/`          | Standalone HTML for longer Moodle pages — lab walkthroughs, the original course schedule, NCI access guide, mid-semester exam instructions, etc. |
| `assets/`         | Images and supporting files (lab tarballs, sample code) embedded in pages, plus `style.css`. |

Everything is plain files. There are no servers, databases, or build steps required.

## Where to start

1. Open `index.html` for the **week-by-week schedule** (Week 1 → Week 12, with
   Mid-Semester Test wedged between Weeks 6 and 7, then Final Exam and Course
   Review Material at the end).
2. Scroll past the schedule for the **categorized lists** (Lectures, Labs,
   Assessments, Resources) if you want to scan a category in bulk.
3. The lab walkthrough pages (Lab 5–8) are linked from the relevant weeks
   *and* from the Labs categorized list — but read *Labs* below before
   relying on them.

## Caveats — read this before you take anything at face value

### Labs

The Semester 2, 2024 offering **delivered labs via GitLab, not Moodle**. The
*Accessing Labs* page (`pages/accessing-labs.html`) is the authoritative source:

- **Pre-Lab Introduction** + **Lab 1** had public GitLab repos linked at the
  time the backup was taken.
- **Labs 2–6** were marked **TBA** — the repos were intended to be published
  week by week during the live course. Whether those repos still exist or are
  still public is something I haven't verified.
- ANU's GitLab moved from `gitlab.cecs.anu.edu.au` to `gitlab.comp.anu.edu.au`
  between when the course ran and when this snapshot was made. The URLs in
  this repo point at the new `comp` subdomain.

The lab walkthrough pages included in this snapshot (titled `Lab 5: Vectorisation
using SSE`, `Lab 6 - OpenMP`, `Lab 7: Advanced OpenMP`, `Lab 8: Catch-up and
assignment lab`) are **carry-overs from an earlier iteration of the course**
(roughly 2018-era), kept hidden by the lecturer in the live 2024 Moodle but
preserved in the backup. Treat them as **reference material for what a previous
Moodle-native version of the labs looked like**, not as the labs students did
in 2024.

The expected 2024 labs (per the course schedule) are L1–L6:

| Week | Lab                                            |
|------|------------------------------------------------|
| 2    | L1: Getting Started and Performance Benchmarking |
| 3    | L2: Loops and Code Performance                 |
| 5    | L3: SIMD                                       |
| 9    | L4: OpenMP                                     |
| 10   | L5: Advanced OpenMP                            |
| 11   | L6: OpenMP SIMD                                |

### Video recordings

Lecture video recordings were excluded from this repo to keep the bundle a
manageable size. Each affected schedule entry shows the note
*"video recording — not included in this shared bundle"* so you can see what
existed; the slides for those same lectures are in `lectures/` as PDFs.

### Hidden Moodle content

Moodle marks content visible/hidden per student access. The filtering policy
for this snapshot is:

- **Live (visible) sections** — included: Week 1–12, Mid-Semester Test, Past
  Exam Papers.
- **Hidden sections kept anyway** — included: Final Exam, Course Review
  Material (these were hidden from students for access control, not because
  they were stale).
- **Hidden sections dropped** — Old Material - Week 1 through Old Material -
  Week 7 (prior-year archives) and other previous-semester sections.
- **Hidden modules inside visible sections** — Moodle *pages* (HTML content,
  e.g. lab walkthroughs) are kept; hidden *resources* (e.g. last-year sample
  solutions, old uploads) are dropped to avoid duplication.

### Interactive elements not preserved

The original Moodle course had forums, feedback surveys, and an LTI tool
(Echo360). None of these have meaningful offline equivalents, so they appear
in the schedule with a parenthetical note like *"(forum: discussion content
not preserved)"* — they list the activity title for completeness, but the
content lives only in the live Moodle.

## Origin

Generated from `backup-moodle2-course-68-comp3320_sem2_2024-20260511-1135-nu.mbz`.
The .mbz itself is not in this repo (5.5GB). The design notes are in
`docs/superpowers/specs/` and the implementation plan in `docs/superpowers/plans/`,
if you want to understand or reproduce the extraction.
