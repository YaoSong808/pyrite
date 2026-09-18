Theme:       CLI write-and-report correctness — the update path stops rewriting frontmatter, health tells the truth by exit code, backup lands in the data dir
Branch:      fix/cli-write-and-report-correctness
Worktree:    /Users/markr/pyrite-wt/fix-cli-write-and-report-correctness
Closes:      #46, #18, #21   (all GitHub milestone 0.24.2)
Model:       opus (the #46 root cause is in the model/storage write path, not the CLI)

Three milestone-0.24.2 CLI bugs that a reviewer reads as one change: the CLI
lies about what it wrote (#46), lies about what it found (#18), and writes
where it was not asked to (#21). #46 is the one that matters; the other two
are small and ride along so the PR is one reviewable unit.

## #46 — `update --tags` rewrites a backlog item's frontmatter (DATA LOSS, do this first)

Repro (run it before you change anything, and paste the diff into your report):

```bash
cd /Users/markr/pyrite-wt/fix-cli-write-and-report-correctness
.venv/bin/pyrite update regression-test-links-fts-quoting -k pyrite \
  --tags "testing,links,fts,audit-2026-07,quality"
git diff kb/backlog/regression-test-links-fts-quoting.md
git checkout kb/backlog/regression-test-links-fts-quoting.md   # put it back
```

Observed on dev at 490603e: the declared fields `kind`, `status`, `priority`,
`effort` are DROPPED, and `body:` (the whole body as a YAML string),
`file_path:` (an absolute path) and `importance: 5` are ADDED. Six KB items
were corrupted this way in one loop. `pyrite sw backlog` silently loses an
item whose `status` is gone — the same failure shape as the 75 stranded items
in the pyrite-dev gotchas.

**Root-cause it; do not patch the symptom.** What I traced before dispatching,
so you do not repeat it — verify each step rather than trusting it:

- `pyrite/cli/entry_commands.py:388` is thin: it just puts `tags` into
  `updates` and calls the service. The bug is below the CLI.
- `pyrite/services/kb_service.py:530 update_entry` does
  `KBRepository.load(entry_id)` → `setattr` per update → `_doc_mgr.save_entry`.
- So the question is what `load` returns for a `backlog_item` (a software-kb
  extension type) and what `to_frontmatter` then emits. `pyrite/models/base.py:166
  _base_frontmatter` starts from `self.extra_frontmatter`, and PR #35 (#15)
  added `capture_extra_frontmatter` (`pyrite/models/base.py:50`,
  `pyrite/models/core_types.py:445`) for exactly this. Find out why that
  protection does not hold here: is the entry loading as a generic Entry
  instead of the typed model, is the extension type's `to_frontmatter` not
  calling the base, or is something serializing the model's own attributes
  (`body`/`file_path`/`importance` are model internals, not frontmatter)?
  Name the mechanism in your report.
- Check `pyrite/storage/repository.py:313` and `:649` (two `load`s) and
  `pyrite/storage/document_manager.py:33 save_entry`.

Acceptance (#46):
- A test that updates tags on a typed entry which has BOTH declared fields
  (`kind`/`status`/`priority`/`effort`) and undeclared keys, and asserts the
  on-disk frontmatter differs ONLY in `tags` — and never emits `body`,
  `file_path` or `importance`.
- `update --title` and `update -b` checked for the same leak, with tests.
  If they leak too, fix them in the same change; if they do not, say why in
  the report (that asymmetry is a clue to the root cause).
- Cover the extension type specifically (`backlog_item` from software-kb),
  not only a core type — the bug was found on a backlog item.
- The repro above produces an empty `git diff` except the tags line.

## #18 — `index health` exits 0 while reporting "unhealthy"

Measured 2026-09-17: printed `"status": "unhealthy"` (22 missing files, 33
stale, 52 malformed frontmatter) and exited 0. Any script, CI step or agent
gating on the exit code reads that as healthy — a failure converted to success
at a boundary. Second gap: no `-k/--kb`, so on a machine with many KBs the
report is dominated by other KBs and cannot serve as a per-project gate (the
pyrite-dev skill uses it as exactly that).

Acceptance (#18):
- Exit 1 when status is unhealthy; `--no-fail` keeps the old behaviour.
- `-k/--kb` scopes every check to one KB.
- Tests: unhealthy fixture → exit 1; `-k` excludes another KB's problems.
- **Do not regress**: stdout stays clean JSON, warnings go to stderr.

## #21 — `db backup` writes into the current directory

`pyrite/cli/db_commands.py:46` writes `pyrite-backup-*.db` into the cwd. The
repo root had accumulated 125 of them (58 MB), gitignored so nobody noticed.

Acceptance (#21):
- Default the backup path to `<data dir>/backups/`; `--output` still takes an
  explicit location.
- Test: `pyrite db backup` from any cwd leaves the cwd untouched.

## Touches

Existing, patch minimally — name any file you touch outside this list in your
report and say why:
  pyrite/cli/entry_commands.py, pyrite/cli/index_commands.py,
  pyrite/cli/db_commands.py, pyrite/services/kb_service.py,
  pyrite/models/base.py, pyrite/models/core_types.py,
  pyrite/storage/repository.py, pyrite/storage/document_manager.py,
  CHANGELOG.md ([Unreleased], one line per issue)
New: tests under tests/ for each of the three.

## Out of scope — hard boundaries, other workers own these files

- `tests/e2e/`, `.github/workflows/ci.yml`, `scripts/run_tutorial*`,
  `pyproject.toml`, `tests/test_dev_process_config.py` — worker on #39.
- `web/` of any kind, `pyrite/server/endpoints/daily.py`, `kb/adrs/0032-*.md`
  — worker on #38.
- Do NOT repair the already-corrupted KB items (#51 tracks the index side);
  do not run bulk `pyrite update` over `kb/`. Fix the code, add the tests.
- Do not open a PR. A draft PR already exists as the claim; I flip it.

## Process

Use this worktree's `.venv`. Load the pyrite-dev skill and follow it: TDD (the
failing test first, and show it failing), root-cause over symptom, evidence
before claims. Run `.venv/bin/pytest tests/ extensions/ -n auto` before you
report. Commit at whatever pace the work needs; `fix:` commits must touch
tests/. When the theme is complete, reply with the pyrite-dev report format,
including the verbatim before/after of the #46 repro diff.
