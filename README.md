# Scholarly Metadata Resolution Pattern

A methodology — not a library — for resolving noisy, free-text citations
(the kind you get from OCR'd footnotes, reference-list extraction, or
manual entry) into verified bibliographic metadata by querying several
free, keyless scholarly-metadata APIs behind one shared interface, in an
order that adapts to the citation's own shape, with a confidence guard
that refuses to attach a wrong record.

It was distilled from a working citation-resolution stage inside an
academic-reading application, where the standing rule is that a
bibliographic fact (title, author, year, identifier) may only ever come
from a real API lookup — never from a language model's guess. This
package generalizes that stage into a reusable pattern, described here in
language-neutral pseudocode written to teach the underlying technique —
read it for the approach, not as a drop-in implementation — so you can
apply the same discipline in your own stack and language.

## What it does

- Defines one small **provider interface** every metadata source (a real
  HTTP-backed catalog, or a stub for testing) implements the same way, so
  the rest of your pipeline never touches a vendor-specific response
  shape.
- Describes **citation-form-aware fallback ordering**: classify a
  citation as book-like, journal-like, or a shape a metadata API can
  never resolve (e.g. a classical locus citation), and reorder — never
  drop — your provider list accordingly.
- Describes a **confidence guard** (title-word overlap over a widened
  top-N candidate window, not blind trust in a provider's #1 result) that
  keeps false matches out without narrowing the search.
- Covers **etiquette and resilience**: polite-pool identification,
  timeouts, per-provider failure isolation, short-circuiting on the first
  confident match, and caching resolved records by canonical identifier.

It does not ship a client library for Crossref/OpenAlex/Google
Books/Open Library specifically — it teaches the pattern so you can wire
up whichever metadata providers your own project actually needs.

## Works with

| Surface | How | Status |
|---|---|---|
| Claude Code | native skill (`SKILL.md`) | not tested |
| Codex | native skill via `.agents/skills` | not tested |
| Gemini CLI | native skill via `.agents/skills`, pointer in `GEMINI.md` | not tested |
| Cursor | native skill via `.agents/skills`, `AGENTS.md` | not tested |
| Copilot | pointer to `AGENTS.md` | not tested |
| ChatGPT / Claude.ai / Gemini (chat) | prompt (`PROMPT.md`) | not tested |

## Quick start

No dependencies and no build step: the skill is methodology and reference
material.

```sh
git clone https://github.com/hyderhusainarastu/scholarly-metadata-resolution-pattern
cd scholarly-metadata-resolution-pattern
./install.sh --claude              # ~/.claude/skills/scholarly-metadata-resolution-pattern
./install.sh --agents --project .  # .agents/skills (Codex, Cursor, Gemini CLI)
./install.sh --gemini --project .  # pointer block in GEMINI.md
./install.sh --copilot --project . # pointer block in .github/copilot-instructions.md
```

Run `./install.sh` with no flags to see help and detected tools. Add
`--dry-run` to preview, `--copy` to copy instead of symlink, and
`--uninstall` to remove only what the installer created. For chat UIs, paste
`PROMPT.md`.

## Worked example (synthetic data)

Say your pipeline extracted this raw footnote text (synthetic example,
not a real citation from any project's corpus):

```
Fenwick, Alden. The Grammar of Ordinary Belief. Cambridge: Cambridge
University Press, 1988.
```

Applying the pattern:

1. **Classify the shape.** The string carries `Press` and a numbered
   edition-adjacent structure with no quoted title and no venue word
   ("Journal", "Review", …) — it classifies as **book-form**.
2. **Reorder providers.** Book-catalog providers go first: a books
   catalog, then a second books catalog, then the broad open-works
   index, then the DOI-centric index — instead of your default
   DOI-index-first order.
3. **Query the first provider** with the cleaned query text
   (`"Fenwick, Alden, The Grammar of Ordinary Belief 1988"`), asking for
   its top 5 candidates rather than just its top 1.
4. **Score each candidate** against the *original* citation text with the
   title-overlap guard. Suppose candidate #3 in that provider's result
   list is:
   ```
   { title: "The Grammar of Ordinary Belief",
     authors: ["Alden Fenwick"],
     year: 1988,
     identifiers: { providerWorkId: "OL-SYN-001W" } }
   ```
   Its title overlaps strongly enough with the query to clear the
   threshold, and it's the highest-scoring candidate in the window — it
   wins, even though it wasn't the provider's own #1 pick.
5. **Return a normalized record**, short-circuiting before any of the
   remaining three providers are ever called:
   ```json
   {
     "provider": "example-book-catalog",
     "identifiers": { "doi": null, "isbn": null, "providerWorkId": "OL-SYN-001W" },
     "title": "The Grammar of Ordinary Belief",
     "authors": ["Alden Fenwick"],
     "year": 1988,
     "url": "https://example-catalog.test/works/OL-SYN-001W",
     "availability": "metadata-only",
     "providerResponse": { "...": "provider's original response, kept for audit" }
   }
   ```

If none of the four providers had produced a candidate clearing the
overlap threshold, the correct outcome is `null` — the citation stays
honestly unresolved rather than being guessed at.

See `SKILL.md` for the full pattern (interface, ordering rules, guard
logic, etiquette, and orchestration pseudocode) and `examples/` for the
synthetic input/output pair above in file form.

## Limits

Pseudocode and methodology only; there is no client library. Results depend
on the coverage and rate limits of the providers you wire up. Locus-style
citations (numbered classical passages) are deliberately left unresolved.

## Related

- [scholarly-citation-resolver](https://github.com/hyderhusainarastu/scholarly-citation-resolver): runnable reference implementation of this pattern.
- [hallucinated-citation-checker](https://github.com/hyderhusainarastu/hallucinated-citation-checker): flags citations that do not resolve to a real record.
- [quote-verification-toolkit](https://github.com/hyderhusainarastu/quote-verification-toolkit): verifies quoted text against sources.

## Keywords

crossref, openalex, google books, open library, citation resolution, DOI, ISBN, bibliographic metadata, hallucination prevention, agent skill

## License

MIT — see `LICENSE`.

---
Part of palimnote-skills - more at https://github.com/hyderhusainarastu
