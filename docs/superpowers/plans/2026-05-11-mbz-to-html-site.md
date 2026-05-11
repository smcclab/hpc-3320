# COMP3320 .mbz → Static HTML Site — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate a static HTML site at the repo root that lets a prospective lecturer browse the COMP3320 course from the .mbz Moodle backup, with a week-by-week schedule, categorized listings, and Bootstrap styling.

**Architecture:** A local Python 3 script (`build.py`, gitignored) extracts the .mbz into `_work/`, parses Moodle XML into an in-memory `Course` model, applies visibility-based filtering, copies file content into categorized output folders (`lectures/ labs/ assessments/ resources/`), renders `index.html` and per-page HTMLs via Jinja2 templates, and self-checks links and counts. Only the generated site is committed.

**Tech Stack:** Python 3 stdlib (xml.etree, hashlib, pathlib, shutil, re), Jinja2 (rendering), pytest (testing), Bootstrap 5 (styling, via CDN).

**Reference spec:** `docs/superpowers/specs/2026-05-11-mbz-to-html-design.md`

**Important context:**
- The `.mbz` file is 5.5GB and is at the repo root: `backup-moodle2-course-68-comp3320_sem2_2024-20260511-1135-nu.mbz`.
- `build.py`, `tests/`, `_templates/`, `_work/`, `*.mbz`, `filtered.log`, `categorization.txt` are all gitignored — they are local-only build artefacts. Only the generated site (index.html + content folders) is committed.
- Because of the gitignore policy, individual TDD cycles do NOT commit. Commits happen at end-of-development checkpoints when generated output changes (Tasks 14 onwards).
- Each section title comparison for the hidden-section allowlist is case-insensitive exact match against `{"Final Exam", "Course Review Material"}`.

---

## File Structure

Local-only (gitignored):

```
build.py                      # CLI entry: orchestrates extract → parse → filter → copy → render → verify
mbz/
  __init__.py
  parse.py                    # XML → Course/Section/Activity/FileRef dataclasses
  filter.py                   # visibility + allowlist filter
  categorize.py               # slugify + regex-based folder assignment
  pages.py                    # Moodle-page HTML cleanup + length classification
  files.py                    # file copy with sha1 dedup
  render.py                   # Jinja2 driver
  verify.py                   # link + count self-checks
_templates/
  index.html.j2               # main site page
  page.html.j2                # standalone Moodle page wrapper
tests/
  conftest.py                 # pytest fixture loaders
  fixtures/                   # small real XML snippets copied from _work/
  test_categorize.py
  test_filter.py
  test_pages.py
  test_files.py
  test_parse.py
_work/                        # extracted .mbz contents
filtered.log                  # audit: what was excluded and why
categorization.txt            # audit: which folder each activity ended up in
```

Committed (the deliverable):

```
index.html
lectures/                     # slugified-title-named files
labs/
assessments/
resources/
pages/                        # standalone HTML for long Moodle pages
assets/
  style.css
  <images>
```

Already committed:

```
.gitignore
README.md
docs/superpowers/specs/2026-05-11-mbz-to-html-design.md
docs/superpowers/plans/2026-05-11-mbz-to-html-site.md     # this file
```

---

## Task 1: Bootstrap project structure

**Files:**
- Create: `build.py`
- Create: `mbz/__init__.py`
- Create: `tests/__init__.py`, `tests/conftest.py`
- Modify: `.gitignore` (add `tests/` to ignore list)

- [ ] **Step 1: Update .gitignore to also ignore tests/ and mbz/**

Edit `.gitignore` to append:

```
tests/
mbz/
```

The `mbz/` Python package is also local-only since the script is gitignored.

- [ ] **Step 2: Install dependencies**

Run:
```bash
pip install --user jinja2 pytest
```

Verify with:
```bash
python -c "import jinja2, pytest; print(jinja2.__version__, pytest.__version__)"
```

Expected: both versions print without error.

- [ ] **Step 3: Create skeleton files**

`build.py`:
```python
"""Entry point: extract .mbz, build the course site at the repo root."""
import sys
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parent
MBZ_GLOB = "*.mbz"
WORK_DIR = REPO_ROOT / "_work"


def main() -> int:
    print("build.py: nothing wired up yet")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

`mbz/__init__.py`:
```python
"""Local helpers for parsing the Moodle .mbz backup."""
```

`tests/__init__.py`: empty.

`tests/conftest.py`:
```python
"""Shared pytest fixtures. Real XML fixtures live under tests/fixtures/."""
from pathlib import Path
import pytest

FIXTURE_DIR = Path(__file__).parent / "fixtures"


@pytest.fixture
def fixture_dir() -> Path:
    return FIXTURE_DIR
```

- [ ] **Step 4: Verify pytest discovers the empty test tree**

Run: `pytest tests/ -v`
Expected: `no tests ran` with exit code 0 or 5 (both fine — no tests collected, no errors).

---

## Task 2: Extract the .mbz to _work/

**Files:**
- Modify: `build.py` (add extract step)

- [ ] **Step 1: Add the extract function**

Append to `build.py`:

```python
import subprocess
import shutil


def extract_mbz() -> Path:
    mbz_files = list(REPO_ROOT.glob(MBZ_GLOB))
    if not mbz_files:
        raise SystemExit(f"No .mbz file in {REPO_ROOT}")
    if len(mbz_files) > 1:
        raise SystemExit(f"Multiple .mbz files found: {mbz_files}")
    mbz = mbz_files[0]
    if WORK_DIR.exists():
        print(f"_work/ already exists; skipping extract")
        return WORK_DIR
    print(f"Extracting {mbz.name} (this takes a minute)...")
    WORK_DIR.mkdir()
    subprocess.run(["tar", "-xzf", str(mbz), "-C", str(WORK_DIR)], check=True)
    return WORK_DIR
```

Update `main()`:

```python
def main() -> int:
    work = extract_mbz()
    print(f"Extracted to {work}")
    return 0
```

- [ ] **Step 2: Run extraction**

Run: `python build.py`
Expected:
- Takes ~30-90s on first run
- `_work/` contains `moodle_backup.xml`, `sections/`, `activities/`, `files/`, `files.xml`.

Verify:
```bash
ls _work/ | head -10
ls _work/sections/ | wc -l   # should be 26
ls _work/activities/ | wc -l # should be 288
```

- [ ] **Step 3: Copy a small set of real XML fixtures into tests/fixtures/**

Run:
```bash
mkdir -p tests/fixtures
# A visible week section
cp _work/sections/section_2/section.xml tests/fixtures/section_visible_week1.xml
# A hidden "Old Material" section
cp _work/sections/section_19/section.xml tests/fixtures/section_hidden_old.xml
# An allowlisted hidden section (Final Exam = section 15)
cp _work/sections/section_15/section.xml tests/fixtures/section_hidden_finalexam.xml
# A visible resource module
cp _work/activities/resource_6534/module.xml tests/fixtures/module_resource_hidden.xml
# Find a visible-module fixture
python -c "
import xml.etree.ElementTree as ET, glob, shutil
for f in sorted(glob.glob('_work/activities/*/module.xml')):
    r = ET.parse(f).getroot()
    if r.find('visible').text == '1':
        shutil.copy(f, 'tests/fixtures/module_resource_visible.xml')
        print('copied', f); break
"
# A page activity
python -c "
import glob, shutil
g = sorted(glob.glob('_work/activities/page_*/page.xml'))
shutil.copy(g[0], 'tests/fixtures/page_content.xml')
shutil.copy(g[0].replace('page.xml','module.xml'), 'tests/fixtures/page_module.xml')
print('copied', g[0])
"
# Trim files.xml to a small sample (full files.xml may be huge)
head -200 _work/files.xml > tests/fixtures/files_sample.xml
```

Expected: each `cp` succeeds; `tests/fixtures/` has ~7 XML files.

---

## Task 3: Slugify (TDD)

**Files:**
- Create: `tests/test_categorize.py`
- Create: `mbz/categorize.py`

- [ ] **Step 1: Write failing tests for slugify**

`tests/test_categorize.py`:

```python
from mbz.categorize import slugify


def test_slugify_basic():
    assert slugify("Lecture 5: GPU Programming") == "lecture-5-gpu-programming"


def test_slugify_collapses_hyphens():
    assert slugify("a  --  b") == "a-b"


def test_slugify_trims_edges():
    assert slugify("  hello world!  ") == "hello-world"


def test_slugify_unicode():
    assert slugify("Café résumé") == "cafe-resume"


def test_slugify_empty():
    assert slugify("") == "untitled"


def test_slugify_punctuation_only():
    assert slugify("!!!") == "untitled"
```

- [ ] **Step 2: Run and verify they fail**

Run: `pytest tests/test_categorize.py -v`
Expected: `ImportError: No module named mbz.categorize` (or `cannot import name 'slugify'`).

- [ ] **Step 3: Implement slugify**

`mbz/categorize.py`:

```python
"""Title slugification and folder categorization for activities."""
import re
import unicodedata


def slugify(title: str) -> str:
    text = unicodedata.normalize("NFKD", title)
    text = text.encode("ascii", "ignore").decode("ascii")
    text = text.lower()
    text = re.sub(r"[^a-z0-9]+", "-", text)
    text = text.strip("-")
    return text or "untitled"
```

- [ ] **Step 4: Run tests; verify pass**

Run: `pytest tests/test_categorize.py -v`
Expected: all 6 tests pass.

---

## Task 4: Categorization regex (TDD)

**Files:**
- Modify: `tests/test_categorize.py`
- Modify: `mbz/categorize.py`

- [ ] **Step 1: Write failing tests for categorize**

Append to `tests/test_categorize.py`:

```python
from mbz.categorize import categorize_folder


def test_categorize_lecture():
    assert categorize_folder("Lecture 5: GPU Programming") == "lectures"
    assert categorize_folder("Lec 10") == "lectures"
    assert categorize_folder("Week 3 Lecture") == "lectures"
    assert categorize_folder("Week 3 Slides") == "lectures"


def test_categorize_lab():
    assert categorize_folder("Lab 02") == "labs"
    assert categorize_folder("Lab: Threads") == "labs"
    assert categorize_folder("Practical 4") == "labs"
    assert categorize_folder("Tutorial 1") == "labs"


def test_categorize_assessment():
    assert categorize_folder("Assignment 1") == "assessments"
    assert categorize_folder("A2: MPI") == "assessments"
    assert categorize_folder("Quiz 3") == "assessments"
    assert categorize_folder("Final Exam 2023") == "assessments"
    assert categorize_folder("Project Brief") == "assessments"
    assert categorize_folder("Mid-Semester Test") == "assessments"
    assert categorize_folder("Midsem Practice") == "assessments"


def test_categorize_default():
    assert categorize_folder("Course Outline") == "resources"
    assert categorize_folder("HPC Reading List") == "resources"
    assert categorize_folder("Welcome") == "resources"


def test_categorize_first_match_wins():
    # "Lab Lecture" should be a lab — first rule matched is lectures, but order matters
    # Actually rules apply in order: lectures first, so "Lab Lecture" matches lab? No — lecture pattern
    # is at start (^lecture\b). "Lab Lecture" starts with "Lab", so lectures regex doesn't match.
    # labs regex (^lab\b) does. So "Lab Lecture" -> labs.
    assert categorize_folder("Lab Lecture Notes") == "labs"
```

- [ ] **Step 2: Run; verify they fail**

Run: `pytest tests/test_categorize.py::test_categorize_lecture -v`
Expected: `ImportError: cannot import name 'categorize_folder'`.

- [ ] **Step 3: Implement categorize_folder**

Append to `mbz/categorize.py`:

```python
_CATEGORY_RULES = [
    ("lectures", [
        re.compile(r"^lecture\b", re.I),
        re.compile(r"^lec\s*\d+", re.I),
        re.compile(r"\bweek\s*\d+\s*(lecture|slides)\b", re.I),
    ]),
    ("labs", [
        re.compile(r"^lab\b", re.I),
        re.compile(r"^practical\b", re.I),
        re.compile(r"^tutorial\b", re.I),
    ]),
    ("assessments", [
        re.compile(r"^assignment\b", re.I),
        re.compile(r"^assessment\b", re.I),
        re.compile(r"^a\d+\b", re.I),
        re.compile(r"^quiz\b", re.I),
        re.compile(r"^exam\b", re.I),
        re.compile(r"\bfinal\s+exam\b", re.I),
        re.compile(r"^project\b", re.I),
        re.compile(r"^mid[-\s]?sem", re.I),
    ]),
]


def categorize_folder(title: str) -> str:
    for folder, patterns in _CATEGORY_RULES:
        for pat in patterns:
            if pat.search(title):
                return folder
    return "resources"
```

- [ ] **Step 4: Run categorize tests; verify pass**

Run: `pytest tests/test_categorize.py -v`
Expected: all categorize tests pass plus the 6 slugify tests from Task 3.

---

## Task 5: Visibility filter (TDD)

**Files:**
- Create: `tests/test_filter.py`
- Create: `mbz/filter.py`

- [ ] **Step 1: Write failing tests for filter rules**

`tests/test_filter.py`:

```python
from mbz.filter import include_section, include_module


def test_include_section_visible():
    assert include_section(title="Week 1", visible=True) is True


def test_include_section_hidden_dropped():
    assert include_section(title="Old Material Week 3", visible=False) is False


def test_include_section_hidden_allowlisted():
    assert include_section(title="Final Exam", visible=False) is True
    assert include_section(title="Course Review Material", visible=False) is True


def test_include_section_allowlist_case_insensitive():
    assert include_section(title="final exam", visible=False) is True
    assert include_section(title="FINAL EXAM", visible=False) is True


def test_include_section_allowlist_exact():
    # "Final Exam 2023" is NOT allowlisted — must be exact "Final Exam".
    assert include_section(title="Final Exam 2023", visible=False) is False


def test_include_module_in_visible_section():
    # In a visible section, only modules with visible=1 are kept.
    assert include_module(module_visible=True, section_visible=True, section_title="Week 1") is True
    assert include_module(module_visible=False, section_visible=True, section_title="Week 1") is False


def test_include_module_in_allowlisted_hidden_section():
    # In an allowlisted hidden section, all modules are kept regardless of module visibility.
    assert include_module(module_visible=True, section_visible=False, section_title="Final Exam") is True
    assert include_module(module_visible=False, section_visible=False, section_title="Final Exam") is True
```

- [ ] **Step 2: Run; verify they fail**

Run: `pytest tests/test_filter.py -v`
Expected: ImportError on `mbz.filter`.

- [ ] **Step 3: Implement the filter rules**

`mbz/filter.py`:

```python
"""Visibility-based filtering of Moodle sections and modules.

A section is included if it is visible in Moodle, OR its title is in the
allowlist (we keep these despite being hidden because they're legitimate
teaching artefacts a new lecturer wants — they're hidden in the live course
only to control student access).

For a module to be rendered, its parent section must be included, AND either
the module itself is visible, OR the section is in the hidden-but-allowlisted
set (in which case the whole section was hidden as a unit and individual
module visibility flags can't be trusted).
"""

HIDDEN_ALLOWLIST = frozenset({"final exam", "course review material"})


def _is_allowlisted(section_title: str) -> bool:
    return (section_title or "").strip().lower() in HIDDEN_ALLOWLIST


def include_section(*, title: str, visible: bool) -> bool:
    if visible:
        return True
    return _is_allowlisted(title)


def include_module(*, module_visible: bool, section_visible: bool, section_title: str) -> bool:
    if section_visible:
        return module_visible
    # Section is hidden — only possible via the allowlist path
    if _is_allowlisted(section_title):
        return True
    return False
```

- [ ] **Step 4: Run filter tests; verify pass**

Run: `pytest tests/test_filter.py -v`
Expected: all 7 tests pass.

---

## Task 6: Page-length classification + Moodle HTML cleanup (TDD)

**Files:**
- Create: `tests/test_pages.py`
- Create: `mbz/pages.py`

- [ ] **Step 1: Write failing tests**

`tests/test_pages.py`:

```python
from mbz.pages import text_length, should_inline, clean_moodle_html


def test_text_length_strips_tags():
    assert text_length("<p>hello world</p>") == len("hello world")


def test_text_length_collapses_whitespace():
    assert text_length("<p>hi\n\n   there</p>") == len("hi there")


def test_should_inline_true_for_short():
    assert should_inline("<p>short intro</p>") is True


def test_should_inline_false_for_long():
    long_html = "<p>" + ("word " * 200) + "</p>"  # ~1000 chars text
    assert should_inline(long_html) is False


def test_should_inline_boundary_at_500():
    # Exactly 500 chars of text should still inline (boundary is <=500)
    body = "x" * 500
    assert should_inline(f"<p>{body}</p>") is True
    body = "x" * 501
    assert should_inline(f"<p>{body}</p>") is False


def test_clean_moodle_html_removes_pluginfile_placeholder():
    raw = '<img src="@@PLUGINFILE@@/diagram.png">'
    cleaned = clean_moodle_html(raw, asset_map={"diagram.png": "assets/abc123-diagram.png"})
    assert "@@PLUGINFILE@@" not in cleaned
    assert "assets/abc123-diagram.png" in cleaned


def test_clean_moodle_html_passthrough_when_no_placeholder():
    raw = "<p>just text</p>"
    assert clean_moodle_html(raw, asset_map={}) == raw
```

- [ ] **Step 2: Run; verify fail**

Run: `pytest tests/test_pages.py -v`
Expected: ImportError on `mbz.pages`.

- [ ] **Step 3: Implement**

`mbz/pages.py`:

```python
"""Moodle 'page' activity HTML processing.

Pages are HTML content authored in Moodle's editor. We need to:
- Strip Moodle-specific markers like @@PLUGINFILE@@ and rewrite image refs.
- Decide whether the cleaned content is short enough to inline in the schedule
  card (text length <= 500 chars after tag stripping), or long enough to deserve
  its own standalone HTML file under pages/.
"""
import re
import urllib.parse

INLINE_TEXT_THRESHOLD = 500

_TAG_RE = re.compile(r"<[^>]+>")
_WS_RE = re.compile(r"\s+")
_PLUGINFILE_RE = re.compile(r'@@PLUGINFILE@@/([^"\'\s>]+)')


def text_length(html: str) -> int:
    text = _TAG_RE.sub("", html or "")
    text = _WS_RE.sub(" ", text).strip()
    return len(text)


def should_inline(html: str) -> bool:
    return text_length(html) <= INLINE_TEXT_THRESHOLD


def clean_moodle_html(html: str, *, asset_map: dict[str, str]) -> str:
    """Rewrite @@PLUGINFILE@@/<name> refs using the provided asset_map.

    asset_map keys are url-decoded relative paths as they appear after the
    placeholder; values are the destination paths to rewrite them to. Refs
    not in the map are left intact (so a later verification step can flag them).
    """
    def replace(match: re.Match) -> str:
        raw = match.group(1)
        decoded = urllib.parse.unquote(raw)
        if decoded in asset_map:
            return asset_map[decoded]
        return match.group(0)
    return _PLUGINFILE_RE.sub(replace, html or "")
```

- [ ] **Step 4: Run; verify pass**

Run: `pytest tests/test_pages.py -v`
Expected: all 7 tests pass.

---

## Task 7: File copy with SHA1 dedup (TDD)

**Files:**
- Create: `tests/test_files.py`
- Create: `mbz/files.py`

- [ ] **Step 1: Write failing tests**

`tests/test_files.py`:

```python
from pathlib import Path
from mbz.files import FileCopier


def test_copy_new_file(tmp_path: Path):
    src = tmp_path / "in" / "abc"
    src.parent.mkdir()
    src.write_bytes(b"hello")
    dest_dir = tmp_path / "out"
    dest_dir.mkdir()

    cp = FileCopier()
    result = cp.copy(src_path=src, sha1="abc123", dest_dir=dest_dir, dest_basename="lecture-1.pdf")
    assert result == dest_dir / "lecture-1.pdf"
    assert result.read_bytes() == b"hello"


def test_copy_dedupes_by_sha1(tmp_path: Path):
    src = tmp_path / "in" / "abc"
    src.parent.mkdir()
    src.write_bytes(b"hello")
    out = tmp_path / "out"
    out.mkdir()

    cp = FileCopier()
    p1 = cp.copy(src_path=src, sha1="abc123", dest_dir=out, dest_basename="lecture-1.pdf")
    p2 = cp.copy(src_path=src, sha1="abc123", dest_dir=out, dest_basename="lab-1.pdf")

    # Second call returns the same path as the first (dedup hit)
    assert p2 == p1
    # And no extra file was created
    assert sorted(p.name for p in out.iterdir()) == ["lecture-1.pdf"]


def test_filename_collision_gets_suffix(tmp_path: Path):
    src1 = tmp_path / "in" / "a"
    src2 = tmp_path / "in" / "b"
    src1.parent.mkdir()
    src1.write_bytes(b"one")
    src2.write_bytes(b"two")
    out = tmp_path / "out"
    out.mkdir()

    cp = FileCopier()
    p1 = cp.copy(src_path=src1, sha1="sha-one", dest_dir=out, dest_basename="lecture-1.pdf")
    p2 = cp.copy(src_path=src2, sha1="sha-two", dest_dir=out, dest_basename="lecture-1.pdf")
    assert p1.name == "lecture-1.pdf"
    assert p2.name == "lecture-1-2.pdf"
    assert p1.read_bytes() == b"one"
    assert p2.read_bytes() == b"two"
```

- [ ] **Step 2: Run; verify fail**

Run: `pytest tests/test_files.py -v`
Expected: ImportError on `mbz.files`.

- [ ] **Step 3: Implement FileCopier**

`mbz/files.py`:

```python
"""Copy Moodle file content to categorized output folders, deduped by SHA1.

The .mbz stores every uploaded file by content hash under files/<XX>/<sha1>.
Two activities pointing at the same content share a SHA1. We want a single
physical copy in the output, with both schedule entries linking to it.

FileCopier tracks sha1 -> destination_path so a second copy() call for the
same sha1 returns the existing path without writing.
"""
from pathlib import Path
import shutil


class FileCopier:
    def __init__(self) -> None:
        self._sha1_to_path: dict[str, Path] = {}

    def copy(self, *, src_path: Path, sha1: str, dest_dir: Path, dest_basename: str) -> Path:
        if sha1 in self._sha1_to_path:
            return self._sha1_to_path[sha1]
        final = self._unique_destination(dest_dir, dest_basename)
        shutil.copy2(src_path, final)
        self._sha1_to_path[sha1] = final
        return final

    @staticmethod
    def _unique_destination(dest_dir: Path, basename: str) -> Path:
        candidate = dest_dir / basename
        if not candidate.exists():
            return candidate
        stem = candidate.stem
        suffix = candidate.suffix
        n = 2
        while True:
            candidate = dest_dir / f"{stem}-{n}{suffix}"
            if not candidate.exists():
                return candidate
            n += 1
```

- [ ] **Step 4: Run; verify pass**

Run: `pytest tests/test_files.py -v`
Expected: 3 tests pass.

---

## Task 8: Section + module parsing (TDD against real fixtures)

**Files:**
- Create: `tests/test_parse.py`
- Create: `mbz/parse.py`

- [ ] **Step 1: Write failing tests using real fixtures**

`tests/test_parse.py`:

```python
from pathlib import Path
from mbz.parse import parse_section, parse_module


def test_parse_section_visible_week1(fixture_dir: Path):
    s = parse_section(fixture_dir / "section_visible_week1.xml")
    assert s.number == 2
    assert "week 1" in s.title.lower()
    assert s.visible is True
    assert len(s.activity_ids) > 0


def test_parse_section_hidden_old(fixture_dir: Path):
    s = parse_section(fixture_dir / "section_hidden_old.xml")
    assert s.visible is False
    assert "old" in s.title.lower()


def test_parse_section_hidden_finalexam(fixture_dir: Path):
    s = parse_section(fixture_dir / "section_hidden_finalexam.xml")
    assert s.visible is False
    assert s.title.strip().lower() == "final exam"


def test_parse_module_resource_hidden(fixture_dir: Path):
    m = parse_module(fixture_dir / "module_resource_hidden.xml")
    assert m.modulename == "resource"
    assert m.visible is False
    assert m.section_id == 789


def test_parse_module_resource_visible(fixture_dir: Path):
    m = parse_module(fixture_dir / "module_resource_visible.xml")
    assert m.modulename == "resource"
    assert m.visible is True
```

- [ ] **Step 2: Run; verify fail**

Run: `pytest tests/test_parse.py -v`
Expected: ImportError on `mbz.parse`.

- [ ] **Step 3: Implement Section + Module dataclasses and parsers**

`mbz/parse.py`:

```python
"""Parse Moodle .mbz XML into in-memory dataclasses."""
from dataclasses import dataclass, field
from pathlib import Path
import xml.etree.ElementTree as ET


@dataclass
class Section:
    id: int
    number: int
    title: str
    visible: bool
    activity_ids: list[int] = field(default_factory=list)


@dataclass
class Module:
    id: int
    modulename: str
    section_id: int
    visible: bool


def _text(elem: ET.Element | None, default: str = "") -> str:
    if elem is None or elem.text is None:
        return default
    return elem.text


def _int(elem: ET.Element | None, default: int = 0) -> int:
    txt = _text(elem)
    return int(txt) if txt.strip() else default


def _bool_flag(elem: ET.Element | None) -> bool:
    return _text(elem).strip() == "1"


def parse_section(path: Path) -> Section:
    root = ET.parse(path).getroot()
    sid = int(root.attrib["id"])
    number = _int(root.find("number"))
    title = _text(root.find("name"))
    visible = _bool_flag(root.find("visible"))
    sequence = _text(root.find("sequence"))
    activity_ids = [int(x) for x in sequence.split(",") if x.strip()]
    return Section(id=sid, number=number, title=title, visible=visible, activity_ids=activity_ids)


def parse_module(path: Path) -> Module:
    root = ET.parse(path).getroot()
    return Module(
        id=int(root.attrib["id"]),
        modulename=_text(root.find("modulename")),
        section_id=_int(root.find("sectionid")),
        visible=_bool_flag(root.find("visible")),
    )
```

- [ ] **Step 4: Run; verify pass**

Run: `pytest tests/test_parse.py -v`
Expected: all 5 tests pass.

---

## Task 9: Resource → file resolution

**Files:**
- Modify: `mbz/parse.py` (add file resolution)
- Modify: `tests/test_parse.py` (add file-ref tests)

- [ ] **Step 1: Write failing tests**

Append to `tests/test_parse.py`:

```python
from mbz.parse import parse_files_xml


def test_parse_files_xml_returns_entries(fixture_dir):
    # The sample is truncated, but the first <file> entry should still parse.
    entries = parse_files_xml(fixture_dir / "files_sample.xml")
    assert len(entries) >= 1
    e = entries[0]
    assert e.sha1 and len(e.sha1) == 40
    assert e.filename != ""
    assert e.component  # "mod_resource", "mod_page", etc.
```

Note: this test is intentionally loose — it accepts whatever real file appears first in the truncated `files_sample.xml`. The shape of the entry is what we're checking, not specific values.

- [ ] **Step 2: Run; verify fail**

Run: `pytest tests/test_parse.py::test_parse_files_xml_returns_entries -v`
Expected: ImportError on `parse_files_xml`.

- [ ] **Step 3: Implement**

Append to `mbz/parse.py`:

```python
@dataclass
class FileEntry:
    sha1: str
    filename: str
    filepath: str          # Moodle's logical path inside the activity, usually "/"
    component: str         # e.g. "mod_resource", "mod_page"
    filearea: str          # e.g. "content", "intro"
    itemid: int            # for resource: usually 0; for page: the page id; for assign: assignment id
    mimetype: str
    contextid: int


def parse_files_xml(path: Path) -> list[FileEntry]:
    """Parse files.xml. May be very large (megabytes); uses iterparse for streaming."""
    out: list[FileEntry] = []
    for _, elem in ET.iterparse(path, events=("end",)):
        if elem.tag != "file":
            continue
        # Skip directory entries (Moodle stores dir-marker rows with empty filename ".")
        filename = _text(elem.find("filename"))
        if filename in ("", "."):
            elem.clear()
            continue
        out.append(FileEntry(
            sha1=_text(elem.find("contenthash")),
            filename=filename,
            filepath=_text(elem.find("filepath"), "/"),
            component=_text(elem.find("component")),
            filearea=_text(elem.find("filearea")),
            itemid=_int(elem.find("itemid")),
            mimetype=_text(elem.find("mimetype")),
            contextid=_int(elem.find("contextid")),
        ))
        elem.clear()
    return out


def sha1_to_disk_path(work_dir: Path, sha1: str) -> Path:
    """files/<XX>/<sha1> — where XX is the first two chars of sha1."""
    return work_dir / "files" / sha1[:2] / sha1
```

- [ ] **Step 4: Run; verify pass**

Run: `pytest tests/test_parse.py -v`
Expected: all parse tests pass.

---

## Task 10: Type-specific activity content extraction

**Files:**
- Modify: `mbz/parse.py`

- [ ] **Step 1: Add activity content reader**

For each activity, given its directory under `_work/activities/<type>_<id>/`, we need the title and any type-specific payload. Append to `mbz/parse.py`:

```python
@dataclass
class Activity:
    id: int
    modulename: str         # resource | page | url | assign | forum | feedback | lti
    section_id: int
    visible: bool
    title: str
    intro_html: str = ""
    # Type-specific fields
    page_content: str = ""           # for page activities
    external_url: str = ""           # for url activities
    lti_tool_url: str = ""           # for lti activities
    # Files attached to this activity (resolved via contextid lookup against files.xml)
    contextid: int = 0


def parse_activity(activity_dir: Path) -> Activity:
    """Parse module.xml + the type-specific XML in one activity directory."""
    module = parse_module(activity_dir / "module.xml")
    type_xml = activity_dir / f"{module.modulename}.xml"
    title = ""
    intro = ""
    page_content = ""
    external_url = ""
    lti_tool_url = ""
    contextid = 0
    if type_xml.exists():
        root = ET.parse(type_xml).getroot()
        contextid = int(root.attrib.get("contextid", 0))
        # Most Moodle activity XMLs have a single child element named after the type
        body = root.find(f"./{module.modulename}")
        if body is not None:
            title = _text(body.find("name"))
            intro = _text(body.find("intro"))
            if module.modulename == "page":
                page_content = _text(body.find("content"))
            elif module.modulename == "url":
                external_url = _text(body.find("externalurl"))
            elif module.modulename == "lti":
                lti_tool_url = _text(body.find("toolurl"))
    return Activity(
        id=module.id,
        modulename=module.modulename,
        section_id=module.section_id,
        visible=module.visible,
        title=title or f"{module.modulename} {module.id}",
        intro_html=intro,
        page_content=page_content,
        external_url=external_url,
        lti_tool_url=lti_tool_url,
        contextid=contextid,
    )
```

- [ ] **Step 2: Add a smoke test**

Append to `tests/test_parse.py`:

```python
def test_parse_activity_page(fixture_dir, tmp_path):
    # Build a minimal activity dir from the fixtures
    act_dir = tmp_path / "page_99999"
    act_dir.mkdir()
    (act_dir / "module.xml").write_bytes((fixture_dir / "page_module.xml").read_bytes())
    (act_dir / "page.xml").write_bytes((fixture_dir / "page_content.xml").read_bytes())

    a = __import__("mbz.parse", fromlist=["parse_activity"]).parse_activity(act_dir)
    assert a.modulename == "page"
    assert a.title  # has some title
    assert a.page_content  # has some HTML content
```

- [ ] **Step 3: Run; verify pass**

Run: `pytest tests/test_parse.py -v`
Expected: all parse tests pass including the new page-activity smoke test.

---

## Task 11: Course assembly (orchestrator)

**Files:**
- Modify: `mbz/parse.py`

This task wires the parsers together. It's mostly glue, so the test is an integration check rather than fine-grained units.

- [ ] **Step 1: Add Course dataclass + load_course function**

Append to `mbz/parse.py`:

```python
@dataclass
class Course:
    sections: list[Section]
    activities_by_id: dict[int, Activity]
    files: list[FileEntry]
    # Index: contextid -> [FileEntry]. Built once for fast lookup.
    files_by_contextid: dict[int, list[FileEntry]] = field(default_factory=dict)


def load_course(work_dir: Path) -> Course:
    sections = []
    for sec_dir in sorted((work_dir / "sections").glob("section_*")):
        sections.append(parse_section(sec_dir / "section.xml"))
    sections.sort(key=lambda s: s.number)

    activities: dict[int, Activity] = {}
    for act_dir in sorted((work_dir / "activities").glob("*_*")):
        if not (act_dir / "module.xml").exists():
            continue
        try:
            a = parse_activity(act_dir)
            activities[a.id] = a
        except ET.ParseError as e:
            print(f"warn: could not parse {act_dir.name}: {e}")

    files = parse_files_xml(work_dir / "files.xml")
    files_by_ctx: dict[int, list[FileEntry]] = {}
    for f in files:
        files_by_ctx.setdefault(f.contextid, []).append(f)

    return Course(
        sections=sections,
        activities_by_id=activities,
        files=files,
        files_by_contextid=files_by_ctx,
    )
```

- [ ] **Step 2: Smoke test against the real _work/ extraction**

Append to `tests/test_parse.py`:

```python
def test_load_course_smoke():
    from pathlib import Path
    from mbz.parse import load_course
    work = Path(__file__).resolve().parents[1] / "_work"
    if not work.exists():
        import pytest
        pytest.skip("no _work/ — run `python build.py` first")
    c = load_course(work)
    assert len(c.sections) == 26
    assert len(c.activities_by_id) > 200
    assert len(c.files) > 100
```

- [ ] **Step 3: Run smoke test**

Run: `pytest tests/test_parse.py::test_load_course_smoke -v`
Expected: PASS (or SKIP if `_work/` somehow isn't there — Task 2 should have created it).

---

## Task 12: Jinja2 templates

**Files:**
- Create: `_templates/index.html.j2`
- Create: `_templates/page.html.j2`
- Create: `mbz/render.py`

No TDD on templates themselves — they're declarative HTML. We test the renderer by snapshotting and eyeballing.

- [ ] **Step 1: Create the index template**

`_templates/index.html.j2`:

```jinja
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>{{ course_title }}</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css">
<link rel="stylesheet" href="assets/style.css">
</head>
<body>
<nav class="navbar navbar-expand-lg bg-light border-bottom mb-3">
  <div class="container">
    <a class="navbar-brand" href="#">{{ course_title }}</a>
    <ul class="nav">
      <li class="nav-item"><a class="nav-link" href="#schedule">Schedule</a></li>
      <li class="nav-item"><a class="nav-link" href="#lectures">Lectures</a></li>
      <li class="nav-item"><a class="nav-link" href="#labs">Labs</a></li>
      <li class="nav-item"><a class="nav-link" href="#assessments">Assessments</a></li>
      <li class="nav-item"><a class="nav-link" href="#resources">Resources</a></li>
    </ul>
  </div>
</nav>
<main class="container">

<section id="schedule">
  <h1 class="mb-4">Schedule</h1>
  {% for section in sections %}
  <div class="card mb-3">
    <div class="card-header"><h2 class="h5 mb-0">{{ section.display_title }}</h2></div>
    <div class="card-body">
      {% for intro in section.inlined_intros %}<div class="moodle-intro mb-2">{{ intro | safe }}</div>{% endfor %}
      {% if section.entries %}
      <ul class="list-unstyled mb-0">
        {% for e in section.entries %}
        <li>
          {% if e.href %}<a href="{{ e.href }}"{% if e.external %} target="_blank" rel="noopener"{% endif %}>{{ e.label }}</a>{% else %}<span class="text-muted">{{ e.label }}</span>{% endif %}
          {% if e.note %}<span class="text-muted small"> — {{ e.note }}</span>{% endif %}
        </li>
        {% endfor %}
      </ul>
      {% endif %}
    </div>
  </div>
  {% endfor %}
</section>

{% for cat in ['lectures','labs','assessments','resources'] %}
<section id="{{ cat }}" class="mt-5">
  <h2 class="text-capitalize">{{ cat }}</h2>
  {% if categorized[cat] %}
  <ul>
    {% for item in categorized[cat] %}
    <li><a href="{{ item.href }}">{{ item.title }}</a></li>
    {% endfor %}
  </ul>
  {% else %}
  <p class="text-muted">No items.</p>
  {% endif %}
</section>
{% endfor %}

<footer class="text-muted small mt-5 mb-3">
  Generated from {{ source_filename }}.
</footer>
</main>
</body>
</html>
```

- [ ] **Step 2: Create the standalone page template**

`_templates/page.html.j2`:

```jinja
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>{{ title }} — {{ course_title }}</title>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css">
<link rel="stylesheet" href="../assets/style.css">
</head>
<body>
<nav class="navbar navbar-expand-lg bg-light border-bottom mb-3">
  <div class="container"><a class="navbar-brand" href="../index.html">← {{ course_title }}</a></div>
</nav>
<main class="container">
<h1 class="mb-3">{{ title }}</h1>
<article class="moodle-page">
{{ content | safe }}
</article>
</main>
</body>
</html>
```

- [ ] **Step 3: Create assets/style.css with minimal tweaks**

`assets/style.css`:

```css
.moodle-intro { background: #fafbfc; padding: .5rem .75rem; border-left: 3px solid #dee2e6; }
.moodle-page img { max-width: 100%; height: auto; }
.card-header h2 { font-weight: 600; }
```

- [ ] **Step 4: Implement the renderer**

`mbz/render.py`:

```python
"""Render the in-memory site model to HTML via Jinja2."""
from pathlib import Path
from jinja2 import Environment, FileSystemLoader, select_autoescape


def make_env(template_dir: Path) -> Environment:
    return Environment(
        loader=FileSystemLoader(str(template_dir)),
        autoescape=select_autoescape(default_for_string=False, default=False),
        keep_trailing_newline=True,
    )


def render_index(env: Environment, *, out_path: Path, **ctx) -> None:
    html = env.get_template("index.html.j2").render(**ctx)
    out_path.write_text(html, encoding="utf-8")


def render_page(env: Environment, *, out_path: Path, **ctx) -> None:
    html = env.get_template("page.html.j2").render(**ctx)
    out_path.write_text(html, encoding="utf-8")
```

Note: autoescape is OFF — Moodle page content and inlined intros are HTML and must pass through unescaped. The Jinja templates explicitly use `| safe` for these fields. Plain title strings will still be escaped via `| escape` if needed — but in our templates, titles inside `<title>`, `<h1>`, and links are tag content, not attribute values; we trust them since they come from a Moodle backup we control. If a title ever contained `<` it would render as broken markup, not as XSS, since the file is served locally.

---

## Task 13: Wire everything together in build.py

**Files:**
- Modify: `build.py`
- Modify: `mbz/__init__.py` (re-export commonly used names; optional)

- [ ] **Step 1: Replace build.py main with the full pipeline**

Overwrite `build.py`:

```python
"""Entry point: extract .mbz, build the course site at the repo root."""
from __future__ import annotations
import subprocess
import sys
from pathlib import Path

from mbz.parse import load_course, sha1_to_disk_path, Activity, FileEntry
from mbz.filter import include_section, include_module
from mbz.categorize import slugify, categorize_folder
from mbz.pages import should_inline, clean_moodle_html, text_length
from mbz.files import FileCopier
from mbz.render import make_env, render_index, render_page

REPO_ROOT = Path(__file__).resolve().parent
WORK_DIR = REPO_ROOT / "_work"
TEMPLATE_DIR = REPO_ROOT / "_templates"
COURSE_TITLE = "COMP3320 High Performance Computing"
OUTPUT_FOLDERS = ["lectures", "labs", "assessments", "resources", "pages", "assets"]


def extract_mbz() -> Path:
    mbz_files = list(REPO_ROOT.glob("*.mbz"))
    if not mbz_files:
        raise SystemExit("No .mbz file in repo root")
    if WORK_DIR.exists():
        print(f"_work/ already exists; skipping extract")
        return WORK_DIR
    print(f"Extracting {mbz_files[0].name} (~30-90s)...")
    WORK_DIR.mkdir()
    subprocess.run(["tar", "-xzf", str(mbz_files[0]), "-C", str(WORK_DIR)], check=True)
    return WORK_DIR


def reset_output_dirs() -> None:
    import shutil
    for name in OUTPUT_FOLDERS:
        d = REPO_ROOT / name
        if d.exists():
            shutil.rmtree(d)
        d.mkdir()
    idx = REPO_ROOT / "index.html"
    if idx.exists():
        idx.unlink()
    # Re-create assets/style.css since reset wipes it
    (REPO_ROOT / "assets" / "style.css").write_text(
        ".moodle-intro { background: #fafbfc; padding: .5rem .75rem; border-left: 3px solid #dee2e6; }\n"
        ".moodle-page img { max-width: 100%; height: auto; }\n"
        ".card-header h2 { font-weight: 600; }\n",
        encoding="utf-8",
    )


def main() -> int:
    work = extract_mbz()
    course = load_course(work)
    print(f"Loaded {len(course.sections)} sections, {len(course.activities_by_id)} activities, {len(course.files)} files")

    reset_output_dirs()
    copier = FileCopier()
    filtered_log: list[str] = []
    categorization_log: list[str] = []
    categorized: dict[str, list[dict]] = {c: [] for c in ["lectures","labs","assessments","resources"]}
    page_files: list[tuple[Activity, Path, str]] = []  # (activity, standalone_html_path, cleaned_html) for long pages
    section_inline_intros: dict[int, list[str]] = {}  # section_id -> list of cleaned HTML blobs
    section_entries: dict[int, list[dict]] = {}      # section_id -> list of entry dicts

    def collect_page_assets(activity: Activity) -> dict[str, str]:
        """Copy images referenced by this page (mod_page/content files) into assets/.

        Returns filename -> relative path from repo root (e.g. 'assets/abc12345-diagram.png').
        """
        m: dict[str, str] = {}
        for f in course.files_by_contextid.get(activity.contextid, []):
            if f.component != "mod_page" or f.filearea != "content":
                continue
            if f.filename in ("", "."):
                continue
            src = sha1_to_disk_path(work, f.sha1)
            basename = f"{f.sha1[:8]}-{f.filename}"
            dest = copier.copy(
                src_path=src, sha1=f.sha1,
                dest_dir=REPO_ROOT / "assets", dest_basename=basename,
            )
            m[f.filename] = str(dest.relative_to(REPO_ROOT))
        return m

    def rebase_assets(m: dict[str, str], prefix: str) -> dict[str, str]:
        return {k: prefix + v for k, v in m.items()}

    # --- Pass 1: filter sections and activities ---
    included_section_ids: list[int] = []
    for s in course.sections:
        if include_section(title=s.title, visible=s.visible):
            included_section_ids.append(s.id)
        else:
            filtered_log.append(f"DROP section {s.number} '{s.title}' (visible={s.visible})")

    activity_included: dict[int, bool] = {}
    section_by_id = {s.id: s for s in course.sections}
    for a in course.activities_by_id.values():
        sec = section_by_id.get(a.section_id)
        if sec is None or sec.id not in included_section_ids:
            activity_included[a.id] = False
            continue
        keep = include_module(
            module_visible=a.visible,
            section_visible=sec.visible,
            section_title=sec.title,
        )
        activity_included[a.id] = keep
        if not keep:
            filtered_log.append(f"DROP activity {a.id} '{a.title}' in section '{sec.title}' (module.visible={a.visible})")

    # --- Pass 2: handle resource/file copying + URL/page/etc. ---
    # Pre-resolve contextid -> file entries for each included activity
    for a in course.activities_by_id.values():
        if not activity_included.get(a.id):
            continue
        sec = section_by_id[a.section_id]
        folder = categorize_folder(a.title)
        categorization_log.append(f"{a.modulename:9s} {a.id:6d}  {folder:11s}  {a.title}")

        entry: dict = {"label": a.title, "href": None, "external": False, "note": None}

        if a.modulename in ("resource", "assign"):
            attached = [
                f for f in course.files_by_contextid.get(a.contextid, [])
                if f.component in ("mod_resource", "mod_assign")
                and f.filearea in ("content", "introattachment", "intro")
            ]
            primary = next(iter(attached), None)
            if primary is None:
                entry["note"] = f"({a.modulename}: no file attached)"
            else:
                src = sha1_to_disk_path(work, primary.sha1)
                ext = Path(primary.filename).suffix or ""
                dest_basename = slugify(a.title) + ext
                dest_path = copier.copy(
                    src_path=src, sha1=primary.sha1,
                    dest_dir=REPO_ROOT / folder, dest_basename=dest_basename,
                )
                rel = dest_path.relative_to(REPO_ROOT)
                entry["href"] = str(rel)
                categorized[folder].append({"title": a.title, "href": str(rel)})
        elif a.modulename == "url":
            entry["href"] = a.external_url
            entry["external"] = True
        elif a.modulename == "page":
            base_assets = collect_page_assets(a)
            if should_inline(a.page_content):
                section_inline_intros.setdefault(sec.id, []).append(
                    clean_moodle_html(a.page_content, asset_map=base_assets)
                )
                entry = None  # already shown as inlined intro
            else:
                standalone_assets = rebase_assets(base_assets, "../")
                cleaned = clean_moodle_html(a.page_content, asset_map=standalone_assets)
                slug = slugify(a.title)
                out = REPO_ROOT / "pages" / f"{slug}.html"
                n = 2
                while out.exists():
                    out = REPO_ROOT / "pages" / f"{slug}-{n}.html"
                    n += 1
                page_files.append((a, out, cleaned))
                entry["href"] = str(out.relative_to(REPO_ROOT))
        elif a.modulename == "forum":
            entry["note"] = "(forum: discussion content not preserved)"
        elif a.modulename == "feedback":
            entry["note"] = "(feedback survey: not preserved)"
        elif a.modulename == "lti":
            entry["note"] = f"(external tool — launch URL: {a.lti_tool_url})"
        else:
            entry["note"] = f"({a.modulename})"

        if entry is not None:
            section_entries.setdefault(sec.id, []).append(entry)

    # --- Pass 3: render templates ---
    env = make_env(TEMPLATE_DIR)
    rendered_sections = []
    for s in course.sections:
        if s.id not in included_section_ids:
            continue
        rendered_sections.append({
            "display_title": s.title.strip() or "Course Information",
            "inlined_intros": section_inline_intros.get(s.id, []),
            "entries": section_entries.get(s.id, []),
        })
    for cat in categorized:
        categorized[cat].sort(key=lambda d: d["title"].lower())

    render_index(
        env,
        out_path=REPO_ROOT / "index.html",
        course_title=COURSE_TITLE,
        sections=rendered_sections,
        categorized=categorized,
        source_filename=next(REPO_ROOT.glob("*.mbz")).name,
    )

    for a, out, cleaned in page_files:
        render_page(
            env, out_path=out,
            course_title=COURSE_TITLE,
            title=a.title,
            content=cleaned,
        )

    (REPO_ROOT / "filtered.log").write_text("\n".join(filtered_log) + "\n", encoding="utf-8")
    (REPO_ROOT / "categorization.txt").write_text("\n".join(categorization_log) + "\n", encoding="utf-8")
    print(f"Generated: index.html + {len(page_files)} standalone pages")
    print(f"  lectures={len(categorized['lectures'])} labs={len(categorized['labs'])} assessments={len(categorized['assessments'])} resources={len(categorized['resources'])}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run the full pipeline**

Run: `python build.py`
Expected: prints loaded counts, generated counts; creates `index.html`, `lectures/...`, `labs/...`, `assessments/...`, `resources/...`, `pages/...`, `assets/style.css`, `filtered.log`, `categorization.txt`.

- [ ] **Step 3: Eyeball the audit logs**

```bash
wc -l filtered.log categorization.txt
head -30 filtered.log
head -30 categorization.txt
```

Expected:
- `filtered.log` lists "DROP section ... Old Material ..." lines and dropped hidden modules.
- `categorization.txt` shows each kept activity and which folder it landed in.

If categorization looks wrong (e.g., a clear lecture lands in resources/), note the title pattern and proceed — fix is in Task 16.

- [ ] **Step 4: Eyeball the output structure**

```bash
ls lectures/ | head
ls labs/ | head
ls assessments/
ls resources/ | head
ls pages/ | head
```

Expected: filenames are slugified activity titles with original extensions.

---

## Task 14: Verification self-checks

**Files:**
- Create: `mbz/verify.py`
- Modify: `build.py` (call verify at end)

- [ ] **Step 1: Write the verifier**

`mbz/verify.py`:

```python
"""Self-checks for the generated site."""
from pathlib import Path
import re
from html.parser import HTMLParser


class _HrefCollector(HTMLParser):
    def __init__(self) -> None:
        super().__init__(convert_charrefs=True)
        self.hrefs: list[str] = []

    def handle_starttag(self, tag, attrs):
        if tag != "a":
            return
        for k, v in attrs:
            if k == "href" and v:
                self.hrefs.append(v)


def collect_hrefs(html_path: Path) -> list[str]:
    p = _HrefCollector()
    p.feed(html_path.read_text(encoding="utf-8", errors="replace"))
    return p.hrefs


def verify_links(repo_root: Path) -> list[str]:
    """Return a list of broken-link errors (empty list if all good)."""
    errors: list[str] = []
    html_files = [repo_root / "index.html"] + list((repo_root / "pages").glob("*.html"))
    for hf in html_files:
        if not hf.exists():
            continue
        for href in collect_hrefs(hf):
            if href.startswith(("http://", "https://", "mailto:", "#")):
                continue
            # strip fragment
            target_str = href.split("#", 1)[0]
            if not target_str:
                continue
            base = hf.parent
            target = (base / target_str).resolve()
            if not target.exists():
                errors.append(f"{hf.name}: missing target {href}")
    return errors


def verify_counts(
    repo_root: Path,
    *,
    expected_file_activities: int,
    expected_long_pages: int,
) -> list[str]:
    """Compare what the build said it produced vs what's actually on disk.

    expected_file_activities: count of included activities (resource/assign) that
    had a file attached and got copied — after dedup, the unique-sha1 count.
    expected_long_pages: count of page activities that produced a standalone HTML.
    """
    errors: list[str] = []
    on_disk_files = sum(
        1
        for folder in ("lectures", "labs", "assessments", "resources")
        for _ in (repo_root / folder).iterdir()
    )
    if on_disk_files != expected_file_activities:
        errors.append(
            f"file count mismatch: on-disk={on_disk_files} expected={expected_file_activities}"
        )
    on_disk_pages = len(list((repo_root / "pages").glob("*.html")))
    if on_disk_pages != expected_long_pages:
        errors.append(
            f"page count mismatch: on-disk={on_disk_pages} expected={expected_long_pages}"
        )
    return errors
```

- [ ] **Step 2: Call verify from build.py**

Append to `build.py`'s `main()` (before the `return 0`):

```python
    from mbz.verify import verify_links, verify_counts
    expected_file_activities = sum(
        1 for p in copier._sha1_to_path.values()
        if p.parent.name in ("lectures", "labs", "assessments", "resources")
    )
    expected_long_pages = len(page_files)
    errs = verify_links(REPO_ROOT) + verify_counts(
        REPO_ROOT,
        expected_file_activities=expected_file_activities,
        expected_long_pages=expected_long_pages,
    )
    if errs:
        print(f"VERIFY: {len(errs)} problems:")
        for e in errs[:20]:
            print(f"  {e}")
        return 1
    print("VERIFY: all internal links resolve, file/page counts match")
```

- [ ] **Step 3: Re-run and verify**

Run: `python build.py`
Expected: prints `VERIFY: all internal links resolve`.

If there are broken-link errors, the most likely cause is a Moodle page that references `@@PLUGINFILE@@/<image>` — we left those un-rewritten in Task 13 (asset_map empty). Either:
- Accept them for now (the broken-image refs will render as broken inline images, but real text content still reads fine), OR
- Implement pluginfile asset extraction (out of scope for v1; add to a follow-up if user wants it).

Note: the verifier doesn't error on un-rewritten `@@PLUGINFILE@@/...` because those don't appear as `<a href>` — they're inside `<img src>`. So link verification will pass even with un-rewritten image refs. That's acceptable.

---

## Task 15: Manual spot-check + local server test

**No files modified** — this is a sanity-check step.

- [ ] **Step 1: Serve the site locally**

Run in one terminal:
```bash
python -m http.server 8000
```

In another (or open the URL):
```bash
curl -s http://localhost:8000/ | head -40
```

Expected: HTML containing the navbar with "Schedule / Lectures / Labs / Assessments / Resources".

- [ ] **Step 2: Open in a browser and click through**

Open `http://localhost:8000/` in a real browser. Verify:
- Navbar links jump to anchors in the page.
- At least one lecture, one lab, one assessment link downloads/opens the file.
- A long Moodle page link opens the standalone `pages/<slug>.html` and renders cleanly.
- The schedule shows Week 1 through Week 12, Mid-Semester Test, Past Exam Papers, plus the allowlisted Final Exam and Course Review Material sections at the appropriate ordering.

If anything is broken, note it in the categorization.txt or filtered.log and address in Task 16.

- [ ] **Step 3: Stop the server**

Kill the `python -m http.server` process.

---

## Task 16: Categorization tune-up (if needed)

**Files (only if Task 15 surfaced misclassifications):**
- Modify: `mbz/categorize.py`
- Modify: `tests/test_categorize.py` (add a test for the new pattern)

- [ ] **Step 1: Inspect categorization.txt for misclassifications**

```bash
grep "  resources " categorization.txt | head -40
```

Look for titles that should clearly have landed in `lectures/`, `labs/`, or `assessments/` but went to `resources/`. Examples:
- "L05 GPUs" — should be lectures (regex `^l\d+` would catch it; add if needed).
- "Tute 3" — should be labs.

- [ ] **Step 2: Add a regression test for each pattern**

For each pattern you find, add a test to `tests/test_categorize.py`:

```python
def test_categorize_l_number_prefix():
    assert categorize_folder("L05 GPUs") == "lectures"
```

- [ ] **Step 3: Run test; verify fail**

Run: `pytest tests/test_categorize.py -v`
Expected: the new test fails.

- [ ] **Step 4: Add the regex; verify pass**

Edit `mbz/categorize.py`'s `_CATEGORY_RULES` to add the pattern. Run tests again: PASS.

- [ ] **Step 5: Re-run build and re-check audit**

```bash
python build.py
grep "  resources " categorization.txt | wc -l
```

Expected: the count of "resources" entries drops; the misclassified titles now appear in the right category.

Iterate this task until you're happy with `categorization.txt`.

---

## Task 17: Commit the generated site

**Files:**
- Add: `index.html`, `lectures/`, `labs/`, `assessments/`, `resources/`, `pages/`, `assets/`

- [ ] **Step 1: Stage the generated artefacts**

```bash
git status
git add index.html lectures labs assessments resources pages assets
git status
```

Expected: staged files include `index.html`, all files under the six output folders. `build.py`, `tests/`, `mbz/`, `_templates/`, `_work/`, `*.mbz`, `filtered.log`, `categorization.txt` must NOT appear (they're gitignored).

- [ ] **Step 2: Sanity-check the staged size**

```bash
git diff --cached --stat | tail -5
du -sh lectures labs assessments resources pages assets
```

Expected: total under ~500MB (likely much less — most of the .mbz weight is in hidden/old material we filtered out). If it's enormous (>1GB), reconsider whether to commit; otherwise proceed.

- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
Generate static HTML site from COMP3320 .mbz

Adds the full generated site at the repo root: index.html with a week-by-week
schedule plus categorized listings for lectures, labs, assessments, and other
resources. Standalone HTML pages for longer Moodle content live under pages/.
Bootstrap 5 styling via CDN. Filtered to the visible course (Week 1-12,
Mid-Semester Test, Past Exam Papers, plus the Final Exam and Course Review
Material sections that were hidden in Moodle but useful for a new lecturer).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 4: Confirm**

```bash
git log --oneline
git status
```

Expected: a clean working tree (apart from gitignored locals); the new commit at HEAD.

---

## Done

The site is committed at the repo root. The prospective lecturer can:
- `git clone` the repo, open `index.html` in any browser locally, OR
- Drop the same files behind any static webserver / GitHub Pages.

Local build artefacts (the .mbz, _work/, build.py, mbz/, tests/, _templates/, audit logs) remain on disk for re-runs but are not part of the repo.
