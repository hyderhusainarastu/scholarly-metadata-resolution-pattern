---
name: scholarly-metadata-resolution-pattern
description: Resolves a noisy free-text citation into verified metadata via Crossref, OpenAlex, Google Books and Open Library behind one adapter interface, with form-aware provider ordering and a title-overlap guard. Use when a pipeline must attach real bibliographic facts, never model-invented ones.
---

# Scholarly Metadata Resolution Pattern

## What this skill is for

A recurring problem in any pipeline that processes citations, reference
lists, or footnotes: you have a noisy free-text string (e.g. "Fenwick,
Alden. The Grammar of Ordinary Belief. Cambridge: Cambridge University
Press, 1988." or an OCR-garbled fragment of the same — synthetic examples
throughout this skill) and you need to attach *real* canonical metadata to
it — a title, authors, a year, a stable identifier (DOI/ISBN/work id), and
a URL — without ever letting a language model guess at that metadata.
Bibliographic facts must come from an actual lookup against a real
catalog, or the citation stays honestly unresolved.

This pattern was extracted from a working implementation that resolves
citations for an academic-reading application, querying four free, keyless
public metadata APIs — **Crossref** (DOI-backed journal/article metadata),
**OpenAlex** (broad open scholarly-works index, strong on books and older
work), **Google Books** (publisher-supplied book metadata: ISBN, imprint,
edition), and **Open Library** (book catalog with public-domain full-text
links) — but the pattern generalizes to any set of metadata providers,
free or paid, keyless or keyed.

Use this skill when:
- You're building a "resolve this citation to a real record" step in an
  ingestion, extraction, or research pipeline.
- An existing resolver only tries one provider (or always tries providers
  in the same fixed order) and you need it to degrade gracefully across
  several.
- You're worried about (or have observed) a resolver either matching the
  wrong record with false confidence, or matching nothing because it only
  ever looks at a provider's #1 search result.
- You need a defensible reason to say "this citation could not be
  resolved" rather than fabricate a plausible-looking answer.

## The four pieces of the pattern

1. **One interface, many providers.** Every metadata source implements the
   same tiny contract and returns the same normalized record shape, so the
   orchestrator (and everything downstream of it) never touches a
   vendor-specific JSON shape.
2. **Fallback ordering that adapts to the citation's own shape**, not one
   fixed provider sequence — reordered, never pruned. A provider that isn't
   first for this citation is still tried if the ones ahead of it miss.
3. **A confidence guard over a widened candidate window**, not blind trust
   in a provider's own #1-ranked result. A citation only resolves when a
   candidate's title genuinely overlaps the query; nothing is attached on
   thin evidence.
4. **Etiquette and resilience**: identify yourself politely to free APIs,
   bound every call with a timeout, swallow a single provider's failure
   without sinking the whole resolution, and avoid re-querying the same
   thing forever.

Each piece is detailed below, with pseudocode. The pseudocode is
illustrative and language-neutral — adapt it to whatever language and
HTTP client your project actually uses; do not treat the function/type
names as an API you must match exactly.

---

## 1. The unified provider interface

Define one narrow contract every provider adapter implements. The exact
field names below are illustrative — pick whatever shape fits your own
pipeline; what matters is that every adapter returns the *same* shape, so
nothing downstream branches on which provider happened to answer:

```
interface MetadataProvider:
  name: string                      # stable identifier, e.g. "crossref"
  search(query: string, signal?) -> ResolvedMetadata | null

interface ResolvedMetadata:
  provider: string                  # which adapter produced this match
  title: string
  authors: string[] | null          # a normalized list, not a raw byline
  year: number | null
  identifiers: {                    # every stable id the provider gave
    doi: string | null,             # you, not just whichever one field
    isbn: string | null,            # you happened to reach for first
    providerWorkId: string | null,
  }
  url: string | null
  availability: "open" | "paywalled" | "metadata-only" | "unknown"
  providerResponse: unknown         # the provider's original payload,
                                     # kept for audit/debugging — never
                                     # read directly for a fact that
                                     # belongs in a normalized field above
```

Design choices worth keeping if you adapt this:

- **`availability` is a first-class field**, not an afterthought — each
  provider skews differently (a DOI-centric index skews `paywalled`; an
  open-works index can genuinely report `open` when it knows; a books-only
  catalog usually can only claim `metadata-only`). Whatever decides "can I
  show this content" downstream should read this field, not infer it from
  which provider happened to match.
- **`identifiers` is a bag, not a single field.** A record can legitimately
  carry a DOI, an ISBN, and a provider-native work id at once (a book
  indexed by both a books catalog and an open-works index, say) —
  collapsing that down to one "the identifier" field throws away real
  information a caller might need.
- **`providerResponse` is kept, never trusted directly.** It exists so a
  human (or a later repair pass) can see exactly what the provider
  returned, but no downstream code should reach into it for a fact that
  belongs in a normalized field above — that reintroduces the
  vendor-shape coupling this interface exists to remove.
- Every adapter owns its own network call, its own response parsing, and
  applying the confidence guard (piece 3) before returning anything. The
  orchestrator never parses a provider's JSON itself.

## 2. Fallback ordering, shaped by the citation itself

Don't always try the same provider first. A journal-article citation and a
19th-century critical edition of a classical text resolve best against
very different catalogs — a DOI-centric index is thin on pre-DOI-era
monographs and reference works; a book catalog is thin on journal
articles. Classify the citation's *form* cheaply (no network call) and use
that to pick an order:

```
function classifyCitationShape(text) -> "book" | "journal" | "unresolvable-form" | "unknown":
  # Check the narrowest, most specific signal FIRST. A citation whose
  # own shape rules out provider lookup entirely (see the classical-locus
  # note below) must never fall through to a book/journal guess.
  if looks_like_a_locus_citation(text):        # e.g. a classical work cited
    return "unresolvable-form"                  # by section/line number,
                                                  # not by title — resolving
                                                  # against a metadata API
                                                  # is a category error, not
                                                  # a provider-ordering
                                                  # problem; route it to a
                                                  # different, non-provider
                                                  # resolution path instead.
  if has_quoted_title(text) or has_venue_markers(text):  # "Journal",
    return "journal"                                      # "Review", "Quarterly", …
  if has_publisher_markers(text) or has_editor_marker(text):  # "Press",
    return "book"                                               # "ed.", "trans.", edition ordinals
  return "unknown"

ORDER_BY_SHAPE = {
  "book":    [googlebooks, openlibrary, openalex, crossref],
  "journal": [crossref, openalex, openlibrary, googlebooks],
  "unknown": [crossref, openalex, openlibrary, googlebooks],
}
# "journal" and "unknown" happen to land on the same sequence in this
# illustration — spell both rows out anyway rather than letting "journal"
# fall through to a shared default entry. That way, if a journal-tuned
# provider ever needs to jump the queue for venue-heavy queries, it's a
# one-line edit to the "journal" row, not a refactor to first separate an
# implicit fallthrough that nobody remembers is there.

function orderProviders(shape, providers):
  priority = ORDER_BY_SHAPE.get(shape, ORDER_BY_SHAPE["unknown"])
  return sort(providers, by = priority.index_of(provider.name))
```

Rules that matter more than the exact regex/keyword list above:

- **Reorder, never drop.** Every provider stays in the loop regardless of
  classification — a book-form citation that misses every book catalog
  still falls through to the DOI-centric providers, because a wrong guess
  about form shouldn't cost real coverage.
- **Classify from the least-processed text available**, not from a
  heavily normalized search query. If an upstream cleaning step strips
  publisher names or edition markers out of the string used for the
  actual provider query (a defensible normalization — those tokens can
  swamp title-matching), classification should still see the *original*
  text so the book/journal signal isn't lost along with the noise. Keep
  the two uses of the text — "what do I classify on" vs. "what do I
  literally send to each provider" — deliberately separate.
- **Give the "doesn't belong in this resolver at all" case its own branch,
  checked first**, ahead of book/journal detection. A classical-work
  locus citation (book/chapter/section numbering, not a title) is not a
  journal article that failed to classify — sending it through four API
  calls it can never match wastes quota and time for a result you can
  already predict is null. Recognize the shape and skip the provider loop
  entirely, routing it to whatever specialized resolution path (if any)
  actually handles that citation family.
- A citation form taxonomy is inherently incomplete. Keep an explicit
  `"unknown"` bucket that preserves whatever your default order is, rather
  than forcing every citation into "book" or "journal."

## 3. The confidence guard: widen the window, keep the bar

Two independent mistakes are easy to make here, and the fix for one must
not reintroduce the other:

- **Trusting a provider's #1 result blindly** is fragile for a noisy,
  OCR-derived, or full-citation-text query: a provider's own relevance
  ranking can put the true match a few rows down, especially when the
  query text is dominated by venue/publisher words rather than the title.
- **Loosening the acceptance threshold** to compensate would fix that at
  the cost of accepting an unrelated record — a worse failure mode than
  leaving a citation unresolved, because it silently attaches a wrong
  fact.

The fix is to widen *which candidates you look at*, not *how permissive
you are about accepting one*:

```
function scoreTitleOverlap(query, title) -> number in [0, 1]:
  # bag-of-words overlap after lowercasing and stripping punctuation,
  # dropping short connector words first (tune the length cutoff to your
  # own corpus — there's nothing special about any particular number here)
  q = significant_words(query)
  t = significant_words(title)
  if q is empty or t is empty: return 0
  return count(w in q if w in t) / size(q)

function pickBestCandidate(query, candidates, get_title, threshold):
  # threshold is a tuning knob you set from your own measurements — see
  # the notes below. Treat any starting number you see here as a
  # placeholder to validate, not a constant to trust as-is.
  best = null
  best_score = -1
  for candidate in candidates:          # candidates = a provider's top-N
    title = get_title(candidate)        # results (e.g. N=5), NOT just #1
    if title is missing: continue
    score = scoreTitleOverlap(query, title)
    if score >= threshold and score > best_score:
      best = candidate
      best_score = score
  # `>`, not `>=`, is deliberate: candidates arrive in the provider's own
  # result order, so when two candidates in the window tie on score, this
  # keeps whichever one the loop reached first instead of letting a later,
  # equally-scored row silently overwrite an earlier one for no reason.
  return best
```

Notes worth preserving if you adapt this:

- Request a small top-N window from each provider (5 is a reasonable
  default) instead of `limit=1`. The guard itself is unchanged — every
  candidate, first or fifth, is held to the exact same threshold — only
  the *search space* widens.
- **Don't borrow a threshold number from someone else's pipeline.** The
  right cutoff depends on your title lengths, your language, and your
  candidate pool, so measure your own false-accept/false-reject rate on a
  real (or synthetic-and-labeled) sample before settling on one, and
  revisit it whenever your citation population changes shape (much
  shorter titles, non-English text where word-count overlap behaves
  differently, etc.). Treat any specific number you encounter in someone
  else's writeup of this pattern — including this one — as a starting
  guess to validate, never as a constant to copy in blind.
- Compute similarity against the citation's full original text, not the
  narrowed provider-query text, if the two differ (see the "least-
  processed text" note above) — a stripped year or publisher token
  shouldn't stop it from counting toward the match evidence, only from
  polluting the outbound search request. Providers can be picky about a
  bare trailing year in a full-text search query in ways they are not
  about the same text used purely for scoring — worth testing per
  provider before assuming it's harmless either way.
- Never fall back to "just take the top result if nothing clears the
  threshold." No match clearing the bar means the citation is
  unresolved by this provider — that's a correct, honest outcome, not a
  failure to paper over.

## 4. Provider etiquette, timeouts, and failure isolation

Free, keyless scholarly APIs generally ask for identification rather than
an API key, and reward it with better rate limits:

- **Send a descriptive `User-Agent`** and, where a provider supports a
  "polite pool" contact parameter (Crossref and OpenAlex both do — an
  email address passed as a query param), send one when you have it
  configured. This is a courtesy, not an auth mechanism; it costs nothing
  and measurably improves reliability under load.
- **Support an optional API key that raises quota without being
  required.** A provider like Google Books works keyless at a modest
  quota; if a key is configured, attach it — but the resolver must still
  function with none of these keys present. A resolution pipeline that
  can't do anything in a fresh environment because no secrets are
  configured yet is a bad default; free-and-keyless-first is a deliberate
  design choice worth keeping.
- **Bound every provider call with a timeout** (an abort signal /
  cancellation token passed down to the HTTP call, cleared once the call
  settles). A slow provider must not stall the whole resolution — it
  should simply lose its turn.
- **Catch and log a single provider's failure; keep going.** One provider
  being down, rate-limited, or returning an unexpected shape should cost
  you that provider's contribution for this citation, not the whole
  resolution. Log enough to diagnose later (provider name + error
  message), then move to the next provider in the ordered list.
- **Short-circuit on the first confident match.** Once a provider clears
  the confidence guard, stop — don't spend calls on the remaining
  providers "just to compare." This keeps typical-case latency and quota
  usage low; the full provider list is a fallback path, not something you
  expect to walk every time.
- **Reject obviously-too-short queries before making any network call at
  all** (e.g. a bare "ibid." or a two-character fragment). This is a free
  win: no provider can usefully match a handful of characters, so don't
  spend a round trip finding that out four times. (The orchestration
  pseudocode below uses a placeholder minimum-length number for this —
  tune it to your own corpus rather than trusting it as-is.)
- **Cache/dedupe by canonical identifier, not by raw query text.** Once a
  citation resolves, persist the resulting record keyed by its stable
  identifier (DOI, ISBN, or provider work-id) in a shared store, and check
  that store before re-querying providers for a citation you have very
  likely already resolved (e.g. the same work cited by two different
  documents, or re-processing after a pipeline re-run). This is standard
  courtesy toward free public APIs as much as it is a performance
  optimization — don't make the same lookup twice just because two
  different raw strings both resolve to the same underlying work.

## 5. Orchestration: the resolve function itself

Putting the four pieces together:

```
function resolveCitation(query, options = {}):
  providers = options.providers ?? DEFAULT_PROVIDERS
  cleaned = trim(query)
  # 4 is a placeholder — tune the minimum length for your workload; it
  # depends on your language and citation style, not on anything universal.
  if length(cleaned) < 4: return null              # too short to be worth a call

  classification_text = options.rawText ?? cleaned  # classify on the least-
                                                      # processed text available
  shape = classifyCitationShape(classification_text)
  if shape == "unresolvable-form": return null      # route elsewhere; zero
                                                      # provider calls spent

  for provider in orderProviders(shape, providers):
    try:
      # 10000ms is a placeholder — tune for your workload based on the
      # latency of the providers you actually call, not on this number.
      record = await with_timeout(
        provider.search(cleaned, timeout = options.timeoutMs ?? 10000)
      )
      if record is not null: return record          # first confident match wins
    except error:
      log(f"[metadata-resolver] {provider.name} lookup failed: {error}")
      # fall through to the next provider — one failure isn't fatal
  return null    # nothing matched: stays unresolved, not guessed
```

The contract this function upholds, end to end:

- **Returns a real record or `null` — never a guess.** Nothing calls an
  LLM to fill in a title/author/year when providers come up empty. If
  your pipeline has an AI stage downstream, it should classify or explain
  a *resolved* citation's relationship to other material; it should never
  be the source of the citation's own factual metadata.
- **A single call to this function is enough** — callers don't need to
  know which providers exist, what order they run in, or how many were
  tried. That knowledge lives entirely inside the orchestrator and the
  provider list.
- **Deterministic and swappable providers** make this trivially testable
  without live network calls: inject fake providers (fixed responses,
  responses that throw, responses that require the real matching logic)
  and assert on call order/short-circuiting/error isolation, all with
  synchronous fakes — the resolution *logic* has no direct dependency on
  the network beyond what each provider adapter itself needs.

## Applying this pattern outside its origin domain

Nothing here is specific to citations of scholarly works. The same four
pieces apply anywhere you need to resolve a noisy free-text reference
against multiple external catalogs and refuse to guess when none match:
product-lookup against several retailer APIs, company-name resolution
against several registries, person/entity resolution against several
directories. Swap the provider list, swap the shape-classification
signals, keep the interface, the confidence guard, the etiquette, and the
"never guess" contract.
