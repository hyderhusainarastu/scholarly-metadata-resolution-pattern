# scholarly-metadata-resolution-pattern

A methodology skill (no code): resolve noisy free-text citations to verified metadata through several keyless scholarly APIs behind one adapter interface, with a title-overlap guard.

## Using this in your project

- Read `SKILL.md` when a task asks you to attach title, author, year or identifier metadata to a raw citation string or reference list.
- Follow the pattern: one provider interface, form-aware provider ordering (reorder, never drop), top-N candidate window scored against the original citation text, short-circuit on the first confident match, cache by canonical identifier.
- Bibliographic facts (title, author, year, DOI, ISBN) come only from an actual API response. Never fill them from model memory. If no candidate clears the overlap threshold, return null and leave the citation unresolved.
- Keep the raw provider response for audit. Send a polite-pool contact identifier, set timeouts and isolate per-provider failures.
- `examples/synthetic-resolution.json` shows the expected input and output shape (synthetic data).
- The skill writes nothing by itself; any files come from the code you write in your own project.
- Runnable reference implementation: `scholarly-citation-resolver`. To check already-resolved citations for invented references, use `hallucinated-citation-checker` (both under github.com/hyderhusainarastu).
- Paste-in version for chat UIs: `PROMPT.md`.

## Working on this repo

- Install locally: `./install.sh` (prints help and detected tools; changes nothing without a target flag).
- Test: `bash test_install.sh` (offline, uses a temporary HOME).
- Keep examples synthetic. Do not add real citations from private corpora, secrets or local paths.
