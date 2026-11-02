# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A CLI (`paprika-recipes`) that checks a Paprika account's recipes out into a directory of markdown
files, tracks local edits git-style, and syncs them back. Paprika has no public API; this speaks the
same private v2 sync protocol the apps use. `readme.md` is the user-facing manual and explains the
sync semantics in depth.

## Commands

```bash
pip install -e .[test]        # dev install (uv works too)
pytest                        # run the whole suite
pytest tests/test_sync.py     # one file
pytest tests/test_sync.py::test_pull_adds_new_recipe   # one test
pytest -k merge               # by name pattern
pre-commit run --all-files    # lint: isort, black, flake8 (bugbear + pytest-style), pyupgrade, mypy
```

CI (`.github/workflows/ci.yml`) runs `pre-commit` and `pytest` on Python 3.10–3.14. `requires-python`
is `>=3.10`; `pyupgrade` is pinned to `--py310-plus`.

## Architecture

### Commands are discovered, not imported

`pyproject.toml` `[project.entry-points."paprika_recipes.commands"]` maps each subcommand name to a
`Command` class. `get_installed_commands()` in `command.py` loads them; `cmdline.py` builds the
argparse subparsers from that. To add a subcommand: add a module under `paprika_recipes/commands/`
whose `Command` subclasses one of the bases in `command.py`, then register it in `pyproject.toml`.

Command base hierarchy (`command.py`):
- `BaseCommand` — argument plumbing, `--json`, the `console` (stderr under `--json`, non-interactive
  so a missing password errors instead of hanging).
- `RemoteCommand` — resolves an account + credentials and hands back a `Remote`.
- `RepositoryCommand` — finds the recipe directory (`Repository.find`, searches upward).
- `RepositorySyncCommand` — both; the account defaults to the one recorded in the directory.

### The git-like model (`repository.py`)

A working directory holds `<Recipe Name>.md` files plus a `.paprika/` directory with a pristine
**base copy** of each recipe as last pulled (`recipes/<uid>.json`), photo bookkeeping
(`photos/<uid>.json`), and `config.yaml` (which account, domain, frontmatter prefix).

Load-bearing invariant, spelled out in the module docstring: **local-edit detection compares
*rendered text*** — the base copy rendered to markdown vs. the bytes on disk — **not parsed fields.**
Both sides pass through the identical markdown transform, so round-trip normalisation cancels out and
an imperfect round-trip can never manufacture a spurious upload. Parsing only happens for files
already known to be edited. Recipes are tracked by the `uid` in their frontmatter, so files can be
renamed and foldered freely. Paprika's `hash` folds in the photo and cannot be recomputed locally;
it is only ever an opaque "has the server's copy moved" token.

### Reconciliation (`sync.py`)

`Syncer.pull` / `push` ask three questions per recipe: has the server moved (base hash vs. index
hash), has the local file moved (`WorkingRecipe.has_local_changes`), did both (→ merge). A merged
file holds the merge while its base copy holds the server's copy, which makes the result an ordinary
local change. Deletion travels both ways but is never destructive — a locally deleted file is pushed
as `in_trash=True`; the base copy is kept until the trashing succeeds so `restore` still works.
`refuse_unidentifiable()` aborts a push when the directory looks like every file stopped being
readable (e.g. a `frontmatter_prefix` mismatch) rather than trashing the account.

### Merge (`merge.py`)

Three-way, base copy as the third input. Prose fields (`MERGEABLE_FIELDS`) merge line-by-line with
git-style conflict markers (`<<<<<<< yours` / `>>>>>>> paprika`) that live in the file until edited
out; `push` refuses a file that still has them. Scalar fields — name, rating, times, and the photo
embed line — cannot hold "either answer", so if one changed on both sides the whole recipe is left
untouched and the user is told which field disagreed.

### Markdown format (`markdown.py`, `DocumentFormat`)

Recipe ⇄ YAML frontmatter + body. Prose fields (`ingredients`, `directions`, `notes`,
`nutritional_info`, description) are passed through verbatim — no ingredient-amount parsing.
`## ` headings not in `SECTIONS` and frontmatter keys not owned by Paprika become `Extras`: carried
through every rewrite untouched, never uploaded, reported to the user. Prose lines that look like a
heading or a photo embed are backslash-escaped (`\##`) so an *unescaped* `## ` always means a section
boundary. `frontmatter_prefix` (chosen at clone time, immutable after) namespaces Paprika's fields so
they can't collide with a vault's own `rating:`/`tags:`; with a prefix set, an unprefixed field is
the user's. `normalize_recipe()` flattens the handful of differences markdown can't represent
(trailing newlines, `photo: null` vs. absent embed).

### Remote (`remote.py`)

`Remote` speaks the v2 sync API. `USER_AGENT` (`constants.py`) **must** be prefixed with Paprika's
iOS app string — the API answers unrecognised agents with `{"error": ...}` and HTTP 200. Auth: POST
to the v1 login endpoint for a bearer token; the password comes from the system keyring under the
account's email (`credentials.py`), never a file we write. `RemoteRecipe` is the recipe's own fields
(including the square-thumbnail `photo`); `RemotePhoto` is a separate gallery-photo object
(`photo_large` on the recipe names the gallery twin). Photo download URLs are short-lived signed
object-storage links, re-minted via `photo_download_url()` when stale.

### Recipes and archives

`recipe.py` `BaseRecipe` → `RemoteRecipe` (sync) and `archive.py` `ArchiveRecipe` (the
`.paprikarecipes` zip used by `extract-archive` / `create-archive`). The archive path has no change
tracking — straight unpack/repack — and always reads/writes unprefixed frontmatter. Archives store
photos inline as base64; `detach_photo` / `attach_photo` move them to/from `attachments/`.

### Output contract (`reporting.py`, `constants.py`)

Every command takes `--json`: a versioned document (`JSON_VERSION`) to stdout, everything human
(report, progress, prompts) to stderr. `ExitCode` (IntEnum): 0 success, 1 attention (conflict / a
recipe `push` refused / `status --exit-code` with changes), 2 argparse usage, 3 auth, 4 remote,
5 user/directory error, 70 internal bug. `cmdline.py` maps exception classes to codes;
`PaprikaUserError` → 5, `AuthenticationError` → 3, other `PaprikaError` → 4.

## Tests

`tests/` mirrors the module layout. There is no `conftest.py`. Sync tests use `FakeAccount` in
`test_sync.py` — an in-memory stand-in for `Remote` that stores recipes as `as_dict()` copies. flake8
runs `flake8-pytest-style`, so follow its conventions (e.g. `@pytest.fixture()` with parens).
