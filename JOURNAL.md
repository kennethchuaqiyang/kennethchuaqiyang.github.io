---
title: "Spec-to-Test-Case Agent - Fix & Enhancement Journal"
render_with_liquid: false
---

# Spec-to-Test-Case Agent — Fix & Enhancement Journal

A dated log of every failure mode found and fixed.

Pairs with `FINDINGS_AND_FIXES.md` in the repo, which has the full technical detail per fix; this is the dated, narrative version.

**The core pattern, throughout:** anything mechanically detectable moves out of the LLM's hands into deterministic Python post-processing, each paired with a unit test built from a real captured failure case — never a synthetic example. A second rule once enough fixes had accumulated: flag broken-but-recoverable content for manual review rather than silently guessing or discarding it; only strip content that's genuinely redundant noise.

## 2026-09-22 (or earlier) — Foundational fixes

*Exact date approximate — taken from the findings doc's own last-recorded metadata, not a commit history.*

1. **Leaked schema field names in `steps`** — the model's own JSON field names leaking into a step. Stripped as pure noise.
2. **Duplicate candidate answers in `expected_result`** — two alternative phrasings joined by a bare comma instead of one being picked. Split and kept the first.
3. **Self-contradicting hedge clause** — "No error message is shown, however, an error message is presented." Trailing contradiction dropped.
4. **Non-text quoted "message"** — a quoted error string that collapsed to a bare comma. Flagged rather than silently deleted.
5. **Unresolved `{name}` template placeholders (first pass)** — `{type}`/`{method}` left literally in the output. Resolved from the case's own preconditions when possible, else flagged.
6. **`_NEGATED_ERROR_RE` proximity bugs** — an unrelated "no error message" clause elsewhere in the same sentence wrongly suppressing a real error. Fixed by bounding the search window, then widening it after a second real example needed more tolerance.
7. **Account-not-found message treated as positive** — "No passcode account matches that username." tagged POSITIVE because it used none of the usual error keywords. Given its own dedicated check.
8. **Analyzer under-extraction** — the most serious bug in the project: a spec that should yield a dozen-plus behaviors came back with exactly one, schema-valid the whole way. Fixed by adding a general content-validation retry mechanism.
9. **Garbled Analyzer behavior ids** — the model's full behavior statement leaking into the `behavior_id` column. Fixed by removing `id` from the schema entirely and stamping it in code.
10. **Pseudo-XML tag debris** — the model serializing an answer as `<tag='value'>` tokens instead of prose. Flagged, not auto-reformatted.
11. **Gap-fill runaway** — two review rounds turned 16 traceable cases into 41, most of the extra 25 duplicates. Capped new cases per round and added duplicate detection.

## 2026-09-23 — Taxonomy, deterministic coverage, and the placeholder/debris tail

12. **`case_type`/`broad_category` taxonomy misclassification** — non-functional cases mistagged FUNCTIONAL; session-gating rules mistagged PARAMETER; a field name and test condition conflated into one string. Fixed with keyword reconciliation, a dedicated reclassifier, and a tightened Generator prompt.
13. **Deterministic PARAMETER-coverage enumeration** (major addition) — comparing my own manual Excel tracker against a real run exposed a structural gap: the LLM only generated whichever field states it happened to think of. Fixed by adding a Field Extractor agent plus a deterministic, code-driven state enumerator matching my own one-factor-at-a-time methodology.
14. **`SELECTION` field with zero options** — Username/Credential misclassified as a dropdown with no options instead of free text. Reclassified in code.
15. **FUNCTIONAL cross-mode mismatch coverage** — a stated business rule that no generated case tested directly. Added a deterministic pass reusing the same field data as #13.
16. **`{method}` placeholder leak in the new cross-mode cases** — fixed by letting a caller pass in a value it already knows deterministically, rather than relying only on the case's own text.
17. **Two more placeholder shapes** — `{type}` and `{sessionStore}`, each needing its own resolution path.
18. **Tracker header rows** — kept the row shape/labels from my real tracker, left live-tracking values blank for me to fill in by hand.
19. **Brace-wrapped string debris** — a field came back as a Python/JSON-ish string literal instead of plain prose. Unwrapped and kept.
20. **A second bracket-wrapped debris shape** — a run-on multi-attribute span the existing XML-tag check couldn't catch. Given its own detector, flagged rather than unwrapped.

## 2026-09-24 — The placeholder/debris tail, an Analyzer output cache, and a repetition loop that wouldn't fully die

21. **A multi-word placeholder name, `{country value}`** — the placeholder regex excluded spaces, so a two-word placeholder name wasn't even detected. Widened to allow space-separated words.
22. **A third debris shape: a full JSON object dump** — a multi-key JSON object (in one real case, even internally malformed) came back instead of prose. Given its own detector, flagged rather than parsed.
23. **A dangling `is 'expected'.` annotation** — an orphaned label tacked onto an otherwise-correct `expected_result`. Stripped when anchored to the very end of the field.
24. **A new `steps` leak shape: a bareword-keyed wrapper around a real step** — a real step serialized inside a `{expect: ...}` wrapper, with an unresolved placeholder still inside it. Fixed by unwrapping the wrapper and extending the same placeholder-substitution pass to `steps`, which previously only ran on the prose fields.
25. **Expanded `_TEXT_FIELD_STATES`** with more type-confusion states (float, non-English/Unicode characters, symbols-only) applied to every TEXT field, plus an opt-in `format_hint` (email/phone/postal code/country code) so format-specific invalid states only apply to a field the spec actually constrains to that format.

**Also built this day:** an Analyzer-output cache — `run_pipeline()` can now take an already-extracted `behaviors` list and skip the Analyzer stage entirely, with a new `--behaviors-file` CLI flag so debugging just the Generator/Reviewer stage doesn't mean re-running the least deterministic part of the pipeline every time.

**Also found this day:** re-running the Gatherer stage several more times on identical input to gauge how often the repetition-loop failure mode still happened, even after the `frequency_penalty=0.4` mitigation, showed it wasn't fully solved — one run collapsed into a single self-contradictory template repeated across every behavior (a silent content failure, worse than a crash since nothing flagged it), another hit the loud JSON-parse failure again. Rather than raise the base penalty for every run (which would make already-successful first attempts more stilted for no benefit), the retry loop now *escalates* the penalty specifically on retries — attempt 1 unchanged, attempt 2 at 0.6, attempt 3 at 0.8. A pure code/config change, no prompt edits, so it carried none of the "prompt bulk causes quality collapse" risk earlier fixes had taught the hard way. Confirmed as a mitigation, not a cure — this is an inherent property of running a small local model.

26. **Tackling the remaining category-skew gaps found the same day** — one more code-level category signal added (`form is/was submitted`, for domains like signup forms that never mention login/session/redirect wording at all); two more content bugs (a garbled/tautological consent-checkbox cascade rule, and an Optional-checkbox wrongly described as blocking Submit) deliberately left unfixed, since fixing them meant teaching the model better facts, not re-tagging its output — and this session had just proven how fragile the local model was to further prompt growth.

## 2026-09-25 — The move to OpenAI, and a cleaner failure mode

**The model switch.** Every fix up to this point was chasing failure modes produced by a small local model (`llama3.2:latest`, 4.1GB, CPU-only). Switched the same, unmodified pipeline to `gpt-4o-mini` via OpenAI's API — purely a `.env` change. Result across all three example specs: **4/4 succeeded on attempt 1 of 3, zero retries, zero leaked/templated/duplicate content** — a categorically different result from every local-model run this project had seen, and strong evidence the local model itself, not the pipeline's prompts or retry logic, was the real source of the repetition-loop family of bugs. One content-quality note surfaced instead: several genuine success statements were still left tagged NEGATIVE, a gap in the category-reconciliation signal detection, not a new kind of bug — tracked and picked up two days later (see below).

**The clean-failure fix.** Separately: if user-story creation ever exhausts its retries and fails outright, does the CLI say so, or does it just crash with a raw traceback and no indication that everything downstream was skipped as a result? Fixed with a small wrapper that catches that specific failure, prints a clear one-line message, and exits cleanly — `--debug` still shows the full traceback for anyone actually diagnosing it.

## 2026-09-28 — Closing the category-skew gap for real, an end-to-end verification pass, and a Reviewer that won't converge

27. **Closed the category-skew gap from 2026-09-25**, in two passes. First pass added two new success phrasings ("can proceed to submit", "proceeds without errors") and a new access-denial signal (a redirect gated on an explicit invalidity condition, not tied to the literal word "login"). Live re-verification on a fresh run immediately surfaced one more gap the first pass didn't cover: the plain word "successfully" — describing a *prior* login in a precondition clause, not the actual denial outcome being tested — was firing as a success signal on its own, ahead of the redirect/denial logic. Second pass made the access-denial check run first, overriding a misleading "successfully" the same way an existing check already overrode a misleading bare "redirect".
28. **Ran the full end-to-end pipeline on OpenAI for the two remaining example specs** (resort-signup, todo-list — login-portal was already done on 2026-09-25), closing out that backlog item. Both ran clean, no crashes or retries. More notably, both showed the same pattern: the Reviewer kept finding new, non-duplicate gaps at the round cap, never converging to "looks complete" — not a spec-specific quirk, a general property of the default 2-round cap.
29. **Tested with a higher round cap (4, then 6)** to see whether the Reviewer would eventually converge on its own. It didn't — a round late in the run was sometimes the *most* productive one, not the least, ruling out a naive "stop once a round looks unproductive" rule.
30. **Added a diagnostic instead of guessing at a stopping rule** — a per-round duplicate-rate metric (what fraction of a round's reported gaps turned out to be duplicates of existing cases) exposed in the log line, so the real trend is visible before committing to any auto-stop logic.
31. **Built the actual auto-stop heuristic on top of that diagnostic** — the pipeline now stops the review loop early once two *consecutive* rounds both come back mostly duplicates, specifically because a single high-duplicate round on its own turned out not to be a safe signal (a real run's most productive round immediately followed its most duplicate-heavy one). Verified on two independent live runs since: it correctly did not fire on data shaped like the misleading case it was built to avoid.

**Also confirmed this day:** the two content bugs left deliberately unfixed back in fix #26 turned out to be resolved on their own by the OpenAI switch — no further code change needed, just verified by inspection on a fresh run.

## 2026-10-02 — Three per-stage review files, and a web UI

Since the 2026-09-28 snapshot below, the pipeline's final leg — test case to
automation — grew from a deterministic Playwright-skeleton generator into a
real LLM step-mapping pass over a site map captured from the live app, paired
with a deterministic reconciliation layer that renders anything it can't
safely resolve as a commented-out `test.skip()` with the specific reason,
rather than a script that might silently do the wrong thing.

32. **Three per-stage review files** — `user_stories_review.md` (Reviewer
    coverage gaps per round, including rounds that found nothing),
    `test_cases_review.md` (surfaces any `[NOT PROVIDED BY MODEL]`/
    `[MALFORMED OUTPUT]` field markers into their own file instead of
    leaving them buried in the full test case list), and
    `automation_review.md` (surfaces every `NEEDS_REVIEW` automation case's
    specific reason). 32 new unit tests, all passing; live-verified across
    three separate real runs.
33. **A web UI, Phase 0** — a FastAPI backend + vanilla-JS frontend wrapping
    the exact same pipeline functions the CLI already calls: zero new
    backend capability, deliberately. Entry stage (raw material / existing
    user story / existing test cases) and target output, gated by the same
    valid-combination matrix as the CLI's own flags, enforced server-side
    and mirrored client-side. Caught and fixed five real usability bugs
    from live testing in rapid iteration — a card-alignment CSS bug, no way
    to append/remove uploaded files across multiple folder picks, a forced-
    JSON test-case upload when the project already reads CSV, and a hard-
    required "original source material" upload relaxed to optional with an
    honestly-labeled fallback — plus one UX finding worth its own line:
34. **Replacing the default debug-log view with a plain-language progress
    list** — the UI's first version showed the raw `logger.info`/
    `logger.warning` feed as its only run output. Accurate, and genuinely
    useful for debugging, but meaningless to someone who just wants to know
    whether their test cases are done. Fixed by adding a second, curated
    progress channel alongside the existing one — plain-language lines
    ("Creating user stories...", "User stories created.", ...) shown by
    default, with the full debug log collapsed behind a "Show details"
    toggle that auto-opens on an error. Nothing stopped being logged; the
    raw channel just stopped being the first thing shown.

## Where it stands (as of 2026-09-28)

- **245 unit tests, all passing**, no live model calls required to run the suite.
- The pipeline has now been run end-to-end (Gatherer/Analyzer → Generator → Reviewer → test cases) on OpenAI for all three example specs, plus higher-round-count variants of two of them.
- Deliberately parked: mode-dependent field splitting (`Credential` → `Password`/`Passcode`), and cross-type username mismatch coverage (rule 3) — both documented in `FINDINGS_AND_FIXES.md` rather than chased further for now.
- Open: whether the Reviewer's auto-stop heuristic (fix #31) will actually fire correctly on a run that genuinely plateaus — confirmed so far only on runs that don't, which is itself useful (no false positives) but not the full picture yet.

## Where it stands (as of 2026-10-02)

- The pipeline now runs end to end from raw source material through to a
  reviewed, runnable automation suite — not just test cases — with its own
  web UI instead of requiring the CLI.
- Open: live multi-page DOM capture from a URL (today's automation stage
  still needs a pre-captured site map) and Figma-image intake for the
  raw-material upload — both scoped, neither built yet.
- Also open: moving this from a local-only tool to something a real user
  (not just me) can reach — public hosting, plus the secrets/auth/storage
  work that has to come with making it public.
