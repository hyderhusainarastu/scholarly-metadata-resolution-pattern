# Prompt: resolve citations to verified metadata, never invented

Paste this into ChatGPT, Claude.ai, Gemini or a local model, then add your citation list or your project description.

---

You help me build or fix a pipeline that turns noisy free-text citations (OCR footnotes, extracted reference lists, manual entry) into verified bibliographic metadata.

Hard rule: a bibliographic fact (title, author, year, DOI, ISBN, URL) may only come from a real API lookup result. Never fill one from memory or guess. If no lookup confirms it, the citation stays "unresolved" (null).

Design the resolver with these parts.

1. Provider interface. Every source (Crossref, OpenAlex, Google Books, Open Library, or a test stub) implements the same small interface: search(query, limit) returning normalized candidates with provider, identifiers {doi, isbn, providerWorkId}, title, authors, year, url, and the original provider response kept for audit. The rest of the pipeline never sees vendor-specific shapes.

2. Form-aware ordering. Classify each citation as book-like (publisher words, Press, edition), journal-like (venue words such as Journal, Review, volume/issue, a DOI) or unresolvable by metadata APIs (for example a classical locus citation like a numbered passage reference). Book-like: put book catalogs first. Journal-like: put the DOI-centric index first. Reorder providers, never drop any. For unresolvable shapes, return unresolved without calling anything.

3. Query. Clean the text into a short query (author surname, title words, year). Ask each provider for its top 5 candidates, not only the first.

4. Confidence guard. Score every candidate against the ORIGINAL citation text by title-word overlap (ignore stopwords, case and punctuation). Accept the best candidate only if it clears a fixed threshold. The provider's own rank 1 is not trusted; a better match at rank 3 wins. Widen the window, keep the bar.

5. Etiquette and resilience. Send a contact identifier where the provider has a polite pool, set timeouts, isolate failures per provider (one provider failing must not fail the run), stop at the first confident match, and cache resolved records by canonical identifier (DOI, ISBN or provider id).

6. Output. One normalized record per resolved citation, or null with a reason code. Never a partial guess.

When I give you citations, show for each: the detected form, the provider order you would use, the query, and the guard decision. If you cannot call the APIs yourself, give me the code or pseudocode and clearly mark every field you have not verified as unverified. Do not invent identifiers.

Worked example (synthetic): "Fenwick, Alden. The Grammar of Ordinary Belief. Cambridge: Cambridge University Press, 1988." is book-like, so book catalogs go first. A candidate titled "The Grammar of Ordinary Belief", author Alden Fenwick, 1988, found at rank 3, clears the overlap threshold and is accepted. If nothing clears it, the answer is null.
