---
title: "Spec-to-Test-Case Agent - Findings & Fixes Log"
author: "Kenneth Chua"
date: "2026-09-22"
render_with_liquid: false
---

# Spec-to-Test-Case Agent — Findings & Fixes Log

A running record of the failure modes surfaced while exercising the pipeline
against a small local model (`llama3.2:latest` via Ollama), and the
deterministic, code-level fixes built for each one. This complements
`README.md` (setup/usage) — this document is the "why does the code look
like this" story.

## The core pattern

Every fix in this log follows the same shape, established early and reused
throughout: **anything mechanically detectable moves out of the LLM's hands
into deterministic Python post-processing, paired with a unit test built
directly from the real failure case that surfaced it.**

Pydantic's `output_type` schema validation only checks *type/structure*
("is this a string, is this an int"), never *content plausibility* ("is this
string actually the right kind of content"). Every bug below is a variation
on that gap: the model returned something schema-valid but still wrong, and
the schema had no way to catch it.

A second recurring principle, once enough fixes had accumulated to need one:
**flag, don't silently strip, unless the content is genuinely redundant
noise.** If a field's broken content still carries the model's real answer
(just in the wrong format, or missing a substitution), deleting it throws
away information a human should go fix. If the content is a duplicate or
self-contradicting fragment that adds nothing beyond what's already said,
dropping it loses nothing. Which bucket a fix falls into is called out below.

## Failure modes found, and their fixes

### 1. Leaked schema field names in `steps`
**Symptom:** the last "step" in a case was literal JSON debris like
`]expected_result": ` — the model's own schema field names leaking into the
field they're supposed to describe, most likely losing its place near the
end of a long prompt (spec grounding + existing-case list).
**Fix:** `_clean_leaked_field_names` / `_FIELD_NAME_LEAK_RE` — strips any
step that's just a leaked field name, nothing else. **Strip** (pure noise).

### 2. Duplicate candidate answers in `expected_result`
**Symptom:** two full candidate sentences concatenated with a bare comma and
no space (`...does not redirect.,An inline error reading...`) — two
alternative phrasings the model considered, joined instead of one being
picked.
**Fix:** `_clean_expected_result` / `_DUPLICATE_ANSWER_SPLIT_RE` — splits on
the `.,<Capital>` pattern (real prose never writes it that way) and keeps
only the first candidate. **Strip** (redundant).

### 3. Self-contradicting hedge clause
**Symptom:** `"No error message is shown, however, an error message is
presented."` — the model second-guessing its own answer and leaving both
sides in instead of picking one.
**Fix:** `_clean_expected_result` / `_HEDGE_CONTRADICTION_RE` — drops the
trailing `however, an error message ... presents` clause. **Strip**
(contradicts the preceding, already-complete sentence).

### 4. Non-text quoted "message"
**Symptom:** `an inline error reading ',', and not redirect...` — the quoted
message text collapsed down to a bare comma instead of an actual string.
**Fix:** `_clean_expected_result` / `_NONTEXT_QUOTED_MESSAGE_RE` — flags any
quoted string after "reading" that contains no letters at all. A negative
lookahead (`(?![A-Za-z])`) guards a real look-alike case: a doubled leading
quote before genuine text (`reading ''Enter a username...`), which must NOT
be flagged. **Flag** (there's nothing to recover, but silently deleting
would leave "reading " dangling with no marker that something's missing).

### 5. Unresolved `{name}` template placeholders
**Symptom:** `title`/`expected_result` came back with a literal, unresolved
`{type}` or `{method}` instead of the concrete value — even though the same
case states that value plainly elsewhere (e.g. preconditions: `'Customer'
selected as the user type`). In `steps`, a messier variant appeared as
pseudocode-looking debris around the placeholder (`| )->{method} ->
'password'`).
**Fix:** `_clean_unresolved_placeholders`:
- In prose fields, substitute the real value when it can be read off the
  *same case's own preconditions* (never guessed, never pulled from
  elsewhere) via `_resolve_placeholder_value`; otherwise leave a clearly
  flagged marker (`[UNRESOLVED PLACEHOLDER: type]`).
- In `steps`, a step combining an arrow (`->`) with an unresolved
  placeholder is debris, not a real action, and is stripped.
**Substitute when resolvable, else flag** — never strip a whole field.

### 6. `_NEGATED_ERROR_RE` proximity bugs (two separate fixes)
**Symptom A:** an unrelated "no error message" clause elsewhere in the same
`expected_result` wrongly suppressed a real, currently-displayed error
(`"...'Incorrect Passcode...' with no error message."` — the "Incorrect
Passcode" error is real; the "no error message" is about something else).
**Fix A:** search a bounded window immediately before the specific error
occurrence, not the whole text.
**Symptom B:** the window was too short — `"without displaying any error"`
(16 characters between negator and "error") went uncaught at a `{0,15}`
gap tolerance.
**Fix B:** widened to `{0,20}`, verified against both the real case (16
chars) and the original counter-example (`"does not redirect and an inline
error is shown"`, 24 chars — correctly still excluded) with an actual
character-count script, not manual counting.
This feeds `_reconcile_case`, which corrects a case's `category`/`priority`
when its own `expected_result` text contradicts the tag the model gave it
(e.g. an error is shown but the case is tagged POSITIVE).

### 7. Account-not-found message treated as a positive signal
**Symptom:** `"No passcode account matches that username."` came back
tagged POSITIVE — it names no "error"/"incorrect"/etc. keyword at all, so
the error-detection heuristic above had nothing to key off. A later run
showed the `<type>` qualifier is sometimes dropped entirely (`"No account
matches that username."`).
**Fix:** `_ACCOUNT_NOT_FOUND_RE` — checked independently, before the
keyword-based logic, since the phrase itself always means failure regardless
of wording around it.

### 8. Analyzer under-extraction (1 behavior instead of 14–24)
**Symptom:** the most serious issue found this project — the Analyzer
returned schema-valid JSON containing a single `TestableBehavior` for a
spec that normally yields well over a dozen. Not a parse failure (existing
malformed-JSON retry never caught it), just implausibly thin content that
every downstream step then silently built out from. Initially co-occurred
with CPU contention (a second model, `gpt-oss:latest`, loaded alongside
`llama3.2:latest`, both at 100% CPU) but was confirmed to recur even without
that contention — a real, standalone instability.
**Fix:** general-purpose content-validation retry, extending the existing
malformed-JSON retry loop rather than duplicating it:
- `_run_structured` gained an optional `validate` callback that can reject
  schema-valid-but-content-bad output by raising `_ContentValidationError`,
  with a retry prompt worded around "this content is a problem" (not "this
  JSON didn't parse", which would be actively misleading here).
- `_validate_analyzer_output` / `MIN_ANALYZED_BEHAVIORS = 3` is the concrete
  check, wired into `analyze_spec`.
This is the first fix in the log that's infrastructure rather than a single
regex — it's the general mechanism any future "schema-valid but obviously
wrong" failure can hook into.

### 9. Garbled Analyzer behavior ids
**Symptom:** `behavior_id` in the output CSV showed the model's full
behavior statement or a garbled paraphrase instead of a short id like `B3`
— e.g. `"Why the new task input field is empty when it should contain
text"` in the `behavior_id` column. Confirmed across multiple specs, not
one-off.
**Fix:** exactly the same fix already applied to `TestCase.id` — remove the
`id` field from the schema entirely rather than trying to correct it.
`DraftBehavior` (the Analyzer's actual output type) has no `id` field; the
model has nothing to get wrong. `analyze_spec` stamps `B1`, `B2`, ... in
code, in order, immediately after the Analyzer returns and before anything
downstream (Generator, Reviewer, dev-questions) ever sees a behavior.
`TestableBehavior` (with the code-assigned id) is the type used everywhere
else in the pipeline.

### 10. Pseudo-XML tag debris (most recent fix)
**Symptom:** instead of writing plain prose, the model serialized an answer
as a chain of XML-ish tags:
```
preconditions: <page='index.html'> <user_role='Customer'>
                <select_user_type='Customer'> <no_login_method_selected>
                </select_user_type> </select_user_type> </user_role> </page>
expected_result: <dropdown_with_options_revealed>
```
**Fix:** `_flag_xml_tag_debris` — flags a field only when its *entire*
content (ignoring whitespace between tags) is made of tag-shaped tokens
(`</?name(='value')?>`), never a field that merely contains a stray `<`/`>`
as part of real prose (e.g. a boundary case reading `"age < 18"`). This is
the same **flag, don't strip** call as #4/#5: the tag-wrapped text is the
model's real answer, just in the wrong format, so it's kept (prefixed
`[MALFORMED OUTPUT - needs manual review]:`) rather than discarded.
Deliberately *not* auto-converted into prose (stripping `<>`/underscores) —
that's a step toward guessing/reformatting model output rather than
detecting-and-flagging, the same kind of shortcut that caused the doubled-
quote false positive in fix #4 before it was caught.

### 11. Gap-fill runaway
**Symptom:** two review rounds turned 16 traceable cases into 41, with most
of the extra 25 having no `behavior_id` and several being outright
duplicates (same scenario, reworded).
**Fix:** two-part —
- `_MAX_NEW_CASES_PER_ROUND_FACTOR = 1.0` in `fill_gaps` caps how many new
  cases a single review round may add (a suite shouldn't usually need more
  new cases per round than it has behaviors in the first place).
- `fill_gap` skips any gap-filled case that's a near-exact repeat of an
  existing one (same title or `expected_result`, normalized), since the
  Generator doesn't reliably honor "don't repeat existing cases" on a small
  model.

### 12. `case_type`/`broad_category` taxonomy misclassification (three related fixes)
**Symptom A:** a case's `broad_category` named a concrete non-functional
concern (`"Load testing"`, `"Session persistence"`) or the literal
meta-label `"Non-Functional"` itself, but `case_type` stayed `FUNCTIONAL`.
**Fix A:** `_NON_FUNCTIONAL_KEYWORDS` reconciliation in `_reconcile_case`,
checked against `broad_category` (the field that actually names the
concern), with the literal phrase `"non-functional"` checked first since no
concrete keyword is a substring of it.
**Symptom B:** three real cases (`"valid session already exists in
localStorage"`, `"no session"`) were tagged `PARAMETER` with a blank or
vague `broad_category` (`"Our Application"`) — not really about one input
field's value at all, but the spec's own session-gating/redirect business
rule.
**Fix B:** `_looks_like_session_gating_case` reclassifies a `PARAMETER` case
mentioning "session" in its `broad_category`/`permutation` fields to
`FUNCTIONAL` (not `NON_FUNCTIONAL` — that axis is reserved for genuine
load/security/session-*mechanics* testing, checked first so it still wins
when the concern is genuinely non-functional).
**Symptom C:** Kenneth's own review of an Excel-opened CSV caught a case
(`broad_category: "No input length"`) where the field name and the test
condition were conflated into one string.
**Fix C:** tightened the Generator's own prompt instructions (not a
post-hoc fix): `broad_category` must be the field's exact name only, never
a phrase describing the test condition.

### 13. Deterministic PARAMETER-coverage enumeration (major addition)
**Symptom:** reviewing Kenneth's real manual Excel tracker (`QuickSample.xlsx`)
against a real run's output surfaced a structural gap, not a formatting bug:
his tracker walks every field's every state deterministically (a `None
Selected`/`Invalid Selected`/`Valid Selected: <option>` state for a
dropdown; `Blank`/`Too Many Characters`/`Leading Spaces`/`Trailing
Spaces`/`Spaces In Between`/`Incorrect Caps`/`Valid` for free text), while
the LLM-driven Generator only produces whichever states it happens to think
of per behavior — a real run missed an entire `Admin` dropdown option and
all invalid-type (integer/boolean) states on `Username` this way.
**Fix:** a new deterministic pass, `enumerate_parameter_coverage`, added
before the Reviewer loop:
- A new `Field Extractor` agent catalogues each input field's name and kind
  (`TEXT`/`SELECTION`/`NUMERIC_CODE`) — a narrower, more mechanical ask than
  writing cases.
- `_enumerate_field_states` generates the full state list **in code**, not
  left to the model — every `SELECTION` option gets its own state
  (closing the missing-`Admin` gap), every `TEXT`/`NUMERIC_CODE` field gets
  its full blank/length/whitespace/case/type-confusion template (closing
  the missing-invalid-type gap).
- Only one field varies at a time, everything else held at a valid
  baseline (`_baseline_description`) — matches Kenneth's own tracker's
  one-factor-at-a-time methodology exactly (his explicit confirmation:
  "As long as the coverage is there and is done in an orderly manner").
- The Generator still writes the prose per field/state, but
  `case_type`/`broad_category`/`permutation_1` are re-stamped
  deterministically afterward, so they can't drift.
- `_dedupe_against` drops any case that's an exact textual repeat of one
  the existing behavior-based generation already produced.
- CSV output is now sorted by `case_type` then `broad_category`
  (`_csv_sort_key` in `output_writer.py`) instead of raw generation order,
  per Kenneth's own feedback that the unsorted CSV was "untidy... makes
  debugging difficult."

### 14. `SELECTION` field with zero options
**Symptom:** the Field Extractor tagged `Username` and `Credential` -
genuinely free-text fields - as `SELECTION` with an empty `valid_options`
list, so they only got the two generic "none/invalid selection" states
instead of their real TEXT template.
**Fix:** `_fix_selection_with_no_options` - a `SELECTION` field with zero
options is a contradiction (nothing to select), so it's mechanically
detectable and reclassified to `TEXT` in code rather than left alone.

## Observed but not (yet) fixed

- **Truncated gap-fill steps** — seen once, in resort-signup's gap-filled
  cases (`Click the 'Submit' button | ==`, `Uncheck | one`). Did not recur
  in either of the two reruns (task-list-manager, login-portal) done the
  same session. Treated as local-model noise for now rather than built into
  a fix — revisit if it recurs.
- **Semantic self-contradictions that don't match a known keyword pattern**
  — e.g. a case titled "Email Duplicate Check Success" whose
  `expected_result` reads "A 'Duplicate found' message is displayed, but the
  account creation succeeds" (a duplicate found should block submission, not
  succeed). Not caught by `_reconcile_case` since no `_ERROR_KEYWORDS`/
  `_SUCCESS_KEYWORDS` term applies. Would need spec-specific business-rule
  awareness to catch mechanically, which conflicts with keeping fixes
  general-purpose — noted, not built.
- **Occasional stray-character oddities** (a lone leading hyphen, a doubled
  word like "Enter Enter") — seen once each, not established as a pattern.
- **Mode-dependent PARAMETER field splitting is unreliable** (investigate
  further before relying on it) — the login-portal spec's `Credential`
  field is really two different fields depending on the `Login using`
  toggle (a free-text password one way, a 6-digit numeric passcode the
  other). The Field Extractor's worked example (a concrete, spec-grounded
  example in its prompt) and a content-validation retry
  (`_validate_fields_output`, same `_ContentValidationError` mechanism as
  fix #8) were both added to force the split into separate `Password`/
  `Passcode` entries. Across three consecutive runs, the model still didn't
  converge: run 1 left `Credential` as one generic field; after the retry
  validator was added, run 2 instead misclassified the *controlling* field
  (`Login using`, the actual dropdown) as `TEXT`, so `Passcode`'s
  numeric-code states never appeared in any run. Each retry produced a
  *different* wrong answer rather than converging on the right one - a
  sign this specific judgment call (detecting cross-field dependency from
  prose) is past what a small local model reliably does, no matter how the
  prompt is worded. Decision (Kenneth, this session): stop iterating
  reactively for now: `Credential` still gets full generic TEXT coverage
  (just not the passcode-specific numeric states), and the rest of the
  PARAMETER/FUNCTIONAL/NON_FUNCTIONAL coverage this run added is solid.
  Two follow-ups worth real investigation later rather than more blind
  retries: (a) a purely mechanical duplicate-field-name/inconsistent-kind
  check (no model judgment needed, so it can't misfire the way a
  content-judgment retry can), or (b) a deliberate, narrower special case
  for "a field whose behavior depends on a same-purpose toggle" instead of
  leaving detection entirely to the model.

  **Options considered for how to proceed, and why Option 2 was picked:**

  | # | Option | Pros | Cons |
  |---|--------|------|------|
  | 1 | One more validator, then stop chasing it - add the mechanical duplicate-field-name/inconsistent-kind check (follow-up (a) above) as a third attempt | Cheap to build; doesn't rely on the model's content judgment at all, so it can't misfire the same way the worked-example and content-validation-retry attempts did | Still reactive - a third patch on top of two that already failed to converge; no guarantee it produces the *right* split rather than a *third* different wrong one |
  | 2 | **Accept current coverage, move on now (chosen)** | Stops a diminishing-returns cycle (2 attempts, 2 different failure modes, 0 convergence); the rest of this run's PARAMETER/FUNCTIONAL/NON_FUNCTIONAL coverage is already solid and shipping it has real value now; keeps the issue visible as a documented finding rather than silently dropped | `Credential`/`Passcode` numeric-code-specific states (blank/too-few-digits/too-many-digits/letters-in-a-numeric-field) stay uncovered until this is revisited |
  | 3 | Hardcode this one pattern - special-case "Login using" / "Credential" by name so the split is forced deterministically for this spec | Guaranteed correct for this spec (and any other spec that reuses the same field names); removes model judgment from this specific case entirely | Spec-specific special-casing, not a general fix; doesn't help with structurally similar mode-dependent fields that use different field names in a different spec - the underlying detection problem stays unsolved |

  Kenneth's decision: go with Option 2 - stop iterating reactively for now,
  document it as a finding, revisit with (a) or (b) above if it turns out
  to matter for another spec.
- **FUNCTIONAL cross-type/cross-method mismatch coverage** — the login
  spec states two explicit "the *other* one doesn't match" business rules:
  rule 3 (a username that exists under the *other* user type doesn't
  match) and rule 5 (a credential value that's correct for the *other*
  login method doesn't match - e.g. typing a real passcode while still in
  Password mode). Comparing a real run's 6-14 FUNCTIONAL cases against
  these rules found neither was directly tested - the generated cases
  covered a generically wrong password/passcode, not specifically "right
  value, wrong mode." This was a business-rule coverage gap, not a
  field-state one, so the deterministic PARAMETER enumeration (fix #13)
  didn't touch it. **Update: rule 5 is now fixed** - see fix #15 below.
  Rule 3 (cross-type username mismatch) is still open; it needs per-type
  identity values (which username belongs to which user type) that aren't
  captured by the current field extraction, so it's a bigger change than
  what fix #15 reused, and is deliberately scoped out for now.
- **New unresolved-placeholder shapes surfaced by the run that verified
  fix #17** — five more cases, each a different underlying cause, none of
  them a miss in fix #17's own logic (verified: the placeholder system is
  correctly falling back to a flagged marker rather than guessing, in
  every one of these):
  - TC-60: `{method}` stayed flagged because the case itself covers *both*
    Password and Passcode scenarios at once ("'Admin' ... Password, or
    'Customer' ... Passcode") - there genuinely isn't one right value to
    substitute.
  - TC-68: `{type}` stayed flagged because the precondition says "dropdown
    set to invalid value" - no concrete Admin/Customer is stated anywhere
    in the case to resolve it from.
  - TC-64: `{message}` plus a bare `{0}` in the same expected_result - the
    model garbling its own output (looks like a half-written Python
    format-string), not a real template gap.
  - TC-05: `{username}` - a placeholder name not seen in any prior run,
    with nothing in the case to resolve it from.
  - TC-06's `{"User type dropdown is set to Customer"}` - a stray
    curly-brace-wrapped quoted string, structurally different from the
    `{name}` placeholders above - **is now fixed, see fix #19 below.**

### 15. FUNCTIONAL cross-mode mismatch coverage (spec rule 5)
**Symptom:** see the "cross-type/cross-method mismatch" gap above - a real
passcode entered while still in Password mode (or vice versa) was never
specifically tested, only generically-wrong values.
**Fix:** a new deterministic pass, `enumerate_cross_mode_mismatches`,
reuses the same field extraction as fix #13 (`_group_mode_dependent_fields`
groups the `Password`/`Passcode`-style variants by their shared controlling
field) and generates one FUNCTIONAL case per pairing: controlling field
held at mode A, but mode B's own valid value typed in. Deterministically
re-stamped as `case_type=FUNCTIONAL`, `broad_category="<controlling field>
cross-mode mismatch"`, same `_dedupe_against` guard as fix #13 so it can't
double up with anything the behavior-based Generator pass already covered.
Kenneth's explicit go-ahead: "go the functional."

### 16. `{method}` template placeholder leaked into a delivered case
**Symptom:** the very first live run of fix #15 (cross-mode mismatch
coverage) produced TC-48 with a raw, unresolved placeholder in its
`expected_result`: *"An inline error reading '[UNRESOLVED PLACEHOLDER:
method]. Please try again.' is shown..."* - the existing placeholder
handling (fix from an earlier session) correctly caught that the model
left `{method}` unresolved, but had no way to fill it in, so it fell back
to its flagged-marker path instead of a real value.
**Fix:** `_resolve_placeholder_value`/`_clean_unresolved_placeholders` now
take an optional `extra_context` dict of values the *caller* already knows
deterministically, checked before the existing preconditions-text lookup.
`generate_cross_mode_case` knows exactly which login method is active (it
chose that value itself before asking the model to write prose about it),
so it now passes `{"method": active_mode.depends_on_value}` straight
through instead of leaving it to a flagged placeholder.
`generate_parameter_case` gets the same treatment for a mode-dependent
field's own cases (e.g. writing a `Passcode` state), since the active mode
is already pinned there too via `forced`. Two new tests: the substitution
path (built from TC-48's real text) and a guard that `extra_context` only
resolves the name it's given, leaving an unrelated placeholder to fall
through to the existing (unchanged) preconditions lookup.

### 17. Two more unresolved-placeholder shapes (`{type}`, `{sessionStore}`)
**Symptom:** the run that shipped fix #16 surfaced two more leaked
placeholders neither earlier fix caught: TC-44's `{type}` stayed
unresolved because its preconditions phrased the role as "a valid Admin
username" rather than the one phrasing the original resolver recognized
("'Admin' selected as the user type"); TC-48's `{sessionStore}` had
nothing in that case's own preconditions to resolve it from at all - it's
the model's own near-miss spelling of the spec's literal `sessionStorage`
term (rule 8), a single fixed mechanism the spec never varies, not a
per-case value like `{type}`/`{method}`.
**Fix:** `_USER_TYPE_IN_PRECONDITIONS_PATTERNS` now tries a second
phrasing (`"admin"/"customer" next to "username"/"account"`) alongside the
original one. Separately, `_resolve_placeholder_value` now also accepts
`spec_text` and resolves a small, explicit set of spec-constant placeholder
names (`sessionstore`/`sessionstorage`) straight from the spec's own
literal wording when it's present - grounded in the spec document already
used to prompt every Generator call, not guessed. Every case-generating
call site (`generate_cases_for_behavior`, `generate_parameter_case`,
`generate_cross_mode_case`, `fill_gap`) now passes its own `spec_text`
through. 3 new tests, one per shape (including a guard that
`{sessionStore}` still gets flagged, not silently dropped, when no
`spec_text` is available).

### 18. Tracker header rows 1-3 (live pass/fail/QA-name metadata)
**Symptom:** `EXTRA_HEADER_ROWS` had sat empty since the taxonomy-expansion
work - the code path to prepend rows above the CSV's column-header row
existed, but Kenneth's real tracker (`QuickSample.xlsx`) rows 1-3 are live
tracking state (`VERSION:`/`ALL_PT:`, `QA:`/`PASSED:`/`UNTEST:`,
`TOTAL:`/`FAILED:`) that a freshly-generated CSV has no true value for -
this was a genuine open decision, not just an unbuilt TODO.
**Fix:** Kenneth's call (via a quick options check this session): keep the
row shape and labels so the CSV drops into the same layout as the tracker,
but leave every live-tracking value blank rather than inventing a `0` -
he fills those in by hand once the suite is actually being run.
`EXTRA_HEADER_ROWS` now holds the 3 real rows, each padded to
`len(CSV_FIELDS)` so they can't silently misalign against the data rows
that follow. 2 new tests: the padding invariant, and an end-to-end
`write_csv` check that the 3 rows land before the real column-header row.

### 19. Brace-wrapped string debris (`{"..."}` instead of plain prose)
**Symptom:** TC-06's preconditions came back as `{"User type dropdown is
set to Customer"}` - a Python/JSON-ish single-string literal (curly braces
around one quoted string) instead of the sentence itself.
`_PLACEHOLDER_RE` (the `{name}` leak detector, fixes #16/#17) doesn't match
this shape at all - there's no `{word}` token, just `{"...arbitrary
prose..."}` - so it passed straight through uncaught.
**Fix:** a new, narrowly-scoped detector, `_unwrap_brace_wrapped_string`,
same "whole field only" discipline as the pseudo-XML tag check (fix from an
earlier session) - only fires when the ENTIRE field content is nothing but
one quoted string inside braces, never a field that merely mentions braces
as part of real prose (e.g. describing a JSON payload). Unlike XML tag
debris, this ISN'T flagged for manual review: the quoted text inside IS the
complete, correct answer, just wrapped in punctuation that doesn't belong,
so it's unwrapped and kept - the same "strip the noise, keep the real
content" treatment as a leaked field name. Wired into all 4
case-constructing call sites, applied first (before `_clean_expected_result`
and the rest of the chain) so downstream cleanup sees plain prose rather
than braces/quotes. 6 new tests, including guards for mismatched-quote
debris and prose that legitimately contains braces.

### 20. A second bracket-wrapped debris shape, missed by the XML tag check
**Symptom:** task-list-manager's TC-40 (from a consolidated 3-spec
diagnosis this session) came back with preconditions written as one long
`<...>` span holding several comma-separated `key = value` pairs, one of
them JSON-shaped: `<TaskList page with New task input = 'A valid task',
Task list = [[{"task":"A task"},{"task":"Another task"}]], Filter = All>`.
This is the same underlying failure as the pseudo-XML tag debris fix (an
earlier session) - the model serializing its real answer instead of
writing prose - but `_XML_TAG_RE` only matches a tag with zero or one
attribute before its own closing `>`, so this run-on multi-attribute span
passed straight through completely uncaught, not even flagged.
**Fix:** a second, narrower detector, `_looks_like_bracket_wrapped_debris`,
checked alongside the existing one inside `_flag_xml_tag_debris`: fires
only when the ENTIRE field is a single `<...>` span AND that span contains
at least one `key = value`-shaped pair (the second condition is what
separates "the model dumped a structured object" from any other
legitimate reason a field might be bracket-wrapped). Flagged, not
unwrapped - unlike fix #19's simple wrapped-string case, there's no safe,
lossless way to turn nested key=value/array/object content back into one
clean sentence, so it gets the same "flag for manual review, don't guess"
treatment as the original XML tag debris. 2 new tests, including a guard
against over-firing on a bracket-wrapped field with no `=` inside it.

This was the session's "one final enhancement" per Kenneth's call after
diagnosing all 3 spec runs together (login-portal, resort-signup,
task-list-manager) - resort-signup's 163 cases came back with only 2
flagged rows, both the same legitimate `[NOT PROVIDED BY MODEL]` gap, a
good sign the accumulated fixes generalize past the spec they were built
against. **Caveat surfaced during that diagnosis:** a run's output file
mtime reflects when it *finished*, not when it *started* - since the
pipeline loads its code into memory once at start, a fix pushed mid-run has
no effect on that run even if its output file is written afterward. None
of the 3 runs diagnosed this session are confirmed to include fix #19 or
#20 for that reason - a fresh run started after this fix was pushed is
needed to actually verify them live.

**Update, next morning:** all 3 specs were rerun overnight (the laptop force-
restarted mid-session, but every run completed cleanly beforehand) - the
first batch confirmed to fully include fixes #16-#20. login-portal and
task-list-manager came back clean; resort-signup (the largest, most varied
spec) surfaced two more real leaks, fixed as #21 and #22 below.

### 21. A multi-word placeholder name, `{country value}`
**Symptom:** TC-42's expected_result kept a literal `{country value}` -
`_PLACEHOLDER_RE` (`\{(\w+)\}`) excludes spaces, so a placeholder name with
more than one word in it wasn't even detected, let alone flagged.
**Fix:** widened to `\{(\w+(?: \w+)*)\}` - one or more space-separated
words, still no other punctuation, so it can't accidentally swallow a
JSON-ish `{"key": value}` dump (see fix #22). TC-42's own preconditions
state the dropdown is at its empty/default value with no concrete country
named, so the correct outcome is still a flagged marker, not a guess - the
bug was purely in detection, not resolution.

### 22. A third debris shape: a full JSON object dump
**Symptom:** TC-193's expected_result came back as a complete, multi-line
JSON object - `{"Code": 400 "message": "...", "Validation errors":
[{...}]}` - not the simple single-quoted-string case fix #19 unwraps (that
fix's regex only matches ONE quoted string with nothing else inside the
braces, so a multi-key object never matches it). In this real example the
JSON was even internally malformed (a stray `"code": 100 "}"` with
mismatched quoting), reinforcing that there's no safe way to parse and
recover it automatically.
**Fix:** a third detector, `_looks_like_json_object_debris`, checked
alongside the XML-tag and bracket-wrapped checks inside
`_flag_xml_tag_debris`: fires only when the whole field is wrapped in one
pair of braces AND contains at least one `"key": `-shaped substring.
Flagged, not unwrapped - same reasoning as fix #20's bracket dump. 2 new
tests, including a guard against over-firing on a brace-wrapped field with
no key-value pair inside it.

**Update, 2026-09-24 morning:** the fixed pipeline (fixes #16-#22) was
rerun fresh against resort-signup after being pushed. Fixes #21/#22 held -
no `{country value}`-style multi-word placeholders and no raw JSON-object
dumps anywhere in the output - but the run surfaced one more new leak, in
`TC-44`, fixed as #23/#24 below (two problems in the same case, one in
`expected_result`, one - for the first time - in `steps`).

### 23. A dangling `is 'expected'.` annotation left on `expected_result`
**Symptom:** TC-44's `expected_result` ended with an orphaned label -
`"...persists the entered email address. is 'expected'."` - the real
sentence, complete and correct, with a stray trailing fragment tacked on
(most likely what's left of a `{expect: "..."}`-style wrapper once
everything else around it got cleaned up).
**Fix:** a fourth pattern added to `_clean_expected_result`,
`_TRAILING_IS_EXPECTED_ANNOTATION_RE` - strips `is 'expected'.` only when
it's anchored to the very end of the field, the same "whole-field or
end-anchored, never mid-sentence" guard every other cleaner in this log
uses. 2 new tests, including a guard against stripping a sentence that
legitimately ends with the word "expected" some other way.

### 24. A new `steps` leak shape: a bareword-keyed wrapper around a real step
**Symptom:** the SAME case's `steps` list had a step that survived
untouched: `"{expect: An inline error message reading 'No {type} account
matches that email address.' is shown ...}"` - the model's real, complete
step, serialized inside a dict-like `{expect: ...}` wrapper, with an
unresolved `{type}` placeholder still inside it. This slipped through for
two structural reasons at once: `_clean_unresolved_placeholders`'s
placeholder substitution has only ever run on `title`/`preconditions`/
`expected_result` - never on `steps` - and the only step-level cleanup that
exists (`_STEP_LOOKS_LIKE_PLACEHOLDER_DEBRIS_RE`) only drops a step shaped
like arrow-wrapped pseudocode, which this isn't.
**Fix:** two changes together. (1) A new detector,
`_unwrap_key_wrapped_step` (`_KEY_WRAPPED_STEP_RE`), unwraps a step that is
ENTIRELY a `{word: ...}` wrapper around real prose - recoverable with zero
information loss, so unwrapped rather than flagged, same call as fix #19
makes for the quoted-string variant of this mistake. (2) Every surviving
step (after the existing arrow-debris drop, which still runs first and
unchanged, so genuinely unrecoverable pseudocode steps are still stripped
exactly as before) now also gets the same `{name}` placeholder
substitution/flagging pass the three prose fields already get, instead of
being left alone. 4 new tests: the wrapper unwraps and the embedded
placeholder resolves when the case states it, stays honestly flagged when
it doesn't (TC-44's own case - its preconditions never state a user type),
a guard against a step that merely mentions a brace-wrapped value as part
of normal prose, and confirmation the pre-existing arrow-debris-drop
behavior for TC-02's case is unchanged.

### 25. Expanded `_TEXT_FIELD_STATES` with more invalid/type-confusion and format-specific states
**Symptom:** "Next up" item 3 from an earlier session - `_TEXT_FIELD_STATES`
(fix #13) only covered blank/length/whitespace/capitalization plus
integer/boolean type-confusion. Kenneth's own list of states worth adding
mixed two different kinds of gap: some are generic type-confusion states
that belong on every TEXT field the same way integer/boolean already do
(float, non-English/Unicode characters, symbols-only); others only make
sense for a field the spec actually constrains to a specific format (email,
mobile number, postal/zip code, country code) - adding those globally would
put a meaningless "invalid email format" state on every plain TEXT field,
including ones like Username or free-text Notes that were never an email to
begin with.
**Fix:** two additions kept structurally distinct in `models.py`/
`pipeline.py`. (1) `float instead of text`, `non-English/unicode
characters`, and `symbols only (special characters, no letters or digits)`
were added directly to `_TEXT_FIELD_STATES`, so - like `integer`/`boolean
instead of text` before them - they apply to every TEXT field automatically.
(2) A new opt-in `FieldFormat` enum (`NONE`/`EMAIL`/`PHONE`/`POSTAL_CODE`/
`COUNTRY_CODE`) and `DraftFieldSpec.format_hint` field let the Field
Extractor agent report a field's spec-stated format constraint, defaulting
to `NONE`; `_enumerate_field_states` looks it up in a new `_FORMAT_HINT_STATES`
dict and appends the matching state(s) ON TOP OF the generic TEXT template
only when set - an `EMAIL` field gets both `invalid email format` and
`non-email format` states, `PHONE`/`POSTAL_CODE`/`COUNTRY_CODE` each get
their one matching state, and a field with no format_hint (the default for
every existing field, including every field in the three example specs)
gets none of them, unchanged from before this fix. The extractor's prompt
(`agents_def.py`) was updated to only set `format_hint` when the spec
itself states the format, not merely because a field's name sounds like one
(e.g. a `Country` dropdown of full country names is a SELECTION field, not
a `COUNTRY_CODE`-format TEXT field). 6 new tests in
`test_pipeline_field_enumeration.py`: the three generic states appear
regardless of format_hint, a field with no format_hint gets none of the
format-specific states, and each of the four `FieldFormat` values adds only
its own matching state(s) - none of the others. Full suite reverified: 206
passed (200 + 6 new).

**Not yet reverified against a live model** - `format_hint` is a new field
the live Field Extractor agent has never been asked to populate before, so
the first live run against any of the three example specs (resort-signup's
Email field is the obvious candidate) is worth checking specifically for
whether the model actually sets it when the spec supports it, and doesn't
over-set it when it doesn't.

## Cache the Analyzer's output (`behaviors`) to skip re-analyzing the spec

Kenneth's own description of the pipeline's stages: gather info -> build
user stories (Analyzer) -> read test cases -> create test cases (Generator/
Reviewer). Debugging or rerunning just the last step used to mean redoing
the whole thing, since `run_pipeline()` was one call and `behaviors` only
ever existed as an in-memory object for its duration - `generate_tests.py`
never wrote it to disk.

**Built 2026-09-24:** `run_pipeline()` now takes an optional `behaviors`
argument - when given, it skips `analyze_spec` entirely and generates test
cases straight from those already-extracted behaviors (default `None`
keeps the original always-analyze-fresh behavior unchanged). The CLI gets a
new `--behaviors-file PATH` flag: if the file already exists, it's loaded
and the Analyzer stage is skipped; if it doesn't exist yet, the Analyzer
runs normally and its output is written there as JSON for next time.
Deleting the file (or pointing at a new path) forces a fresh analysis. 2
new tests (`tests/test_pipeline_behaviors_cache.py`) confirm the wiring:
cached behaviors bypass `analyze_spec`, and omitting them still calls it as
before.

**Extended same day:** `--behaviors-file`'s resume behavior still always
runs the full pipeline through to test-case generation - it just decides
whether the Analyzer's input comes from a fresh call or the cache. That's
not the same as running the Analyzer stage **by itself** and stopping,
which Kenneth asked for next, specifically for debugging - inspecting what
the Analyzer extracted before committing to a full run. New `--analyze-only`
flag: runs only `analyze_spec`, writes the result to `--behaviors-file`
(defaulting to `<out-dir>/user_stories.json` if not given), prints the
behavior count, and exits - no test cases, no `test_cases.md`/`.csv`. Unlike
the resume flow, it always re-analyzes and overwrites the file even if one
already exists, since regenerating it fresh is the entire point of the
flag. 3 new tests (`tests/test_generate_tests_cli.py`, the first tests
covering `generate_tests.py` itself rather than only `spec_to_tests/`)
confirm: it writes behaviors and skips `run_pipeline` and case output
entirely, it honors a custom `--behaviors-file` path, and it overwrites an
existing cache rather than silently reusing it.

## A Gatherer stage: user stories directly from raw project material

Kenneth's fuller description of the intended architecture, stated
explicitly (2026-09-24): three separate callable functions - "call user
stories creation" alone, "call test case creation" alone (from existing
user stories), and "call user stories creation, then test case creation" as
one combined run - where "user stories creation" itself should be able to
read whatever raw material is actually available (business dialogue,
meeting notes, a data-flow diagram, a wireframe), not only an
already-written spec.

`docs/mock-project-inputs/` (fabricated business-dialogue.md, meeting-
notes.md, a data-flow-diagram.md, and a login-wireframe.svg, all grounded in
the real login-portal spec) has existed since earlier in the project
specifically as material for this - see this log's very first "Next up"
list. It was never wired into anything until now.

**Built 2026-09-24:**
- **`gatherer_agent`** (`agents_def.py`) - a second agent alongside
  `analyzer_agent`, same `AnalyzerOutput` schema (so it produces the exact
  same `list[TestableBehavior]` shape), but instructed for messier input:
  several labeled source documents instead of one clean spec, the same fact
  restated across sources (corroboration, not a duplicate behavior),
  sources that conflict or supersede each other, and - most importantly -
  material EXPLICITLY marked unresolved (an "Open question, not answered",
  an open action item) must be left out rather than guessed at, the same
  "never invent, never guess" standard the Analyzer already holds.
- **`pipeline.gather_behaviors(source_text)`** - the Gatherer's counterpart
  to `analyze_spec`, sharing the same id-stamping logic (factored out into
  `_stamp_behavior_ids`) and the same `_validate_analyzer_output` floor
  check, so a degenerate too-few-behaviors reply gets retried the same way
  regardless of which agent produced it.
- **`source_gathering.py`** (new module, no LLM calls, pure text prep) -
  `combine_sources()` reads and concatenates any number of raw source
  files, each labeled `--- Source: <filename> ---` so a behavior's
  `source_hint` can be traced back to which document it came from. A `.svg`
  wireframe is handled by extracting its `<text>` element content
  (`extract_svg_text`) rather than adding an image/vision-model dependency
  - a low-fi wireframe's meaningful content is its text labels and
  annotations, not its shapes/positions, and `login-wireframe.svg`'s real
  labels ("User type", "Customer1", the numbered annotations) confirmed
  this is enough.
- **CLI**: `--spec` and a new `--gather-from PATH [PATH ...]` are now a
  mutually-exclusive pair - exactly one names where user stories come from.
  Kenneth's three functions map onto the existing flags directly, with
  either source:
  - user stories alone: `--analyze-only` (with `--spec` or `--gather-from`)
  - test cases alone, from existing user stories: `--behaviors-file
    <existing file>` (works with either source too, since it's just the
    grounding text for later stages either way)
  - both, back to back: neither flag - the plain default run
  `run_pipeline`'s own internal `analyze_spec` fallback only applies to
  direct library callers now - the CLI always resolves behaviors itself
  first (via `analyze_spec` or `gather_behaviors`, whichever source was
  given) and passes the result into `run_pipeline` explicitly, so the
  Gatherer path is actually used end-to-end rather than silently falling
  back to the Analyzer.
- 14 new tests total across `tests/test_source_gathering.py` (SVG
  extraction, source labeling/ordering - no LLM), `tests/
  test_pipeline_gather_behaviors.py` (id-stamping, correct agent called,
  raw material reaches the prompt - `_run_structured` mocked), and
  additions to `tests/test_generate_tests_cli.py` (`--gather-from` +
  `--analyze-only` calls the Gatherer not the Analyzer; a missing source
  file is reported and stops before any LLM call; the combined full run
  routes through `gather_behaviors` before `run_pipeline`, never
  `run_pipeline`'s own default; `--behaviors-file` resume skips user-story
  creation regardless of which source was named).

**Not yet exercised against a live model** - the wiring is verified with
`_run_structured` mocked throughout (no Ollama instance in the test suite,
matching every other test in this project), but `gather_behaviors` itself
hasn't been run against the real `docs/mock-project-inputs/` material yet.
Worth a real run to see what `gatherer_agent` actually extracts from the
business-dialogue/meeting-notes/data-flow-diagram/wireframe material before
trusting it the way `examples/login-portal-spec.md` is trusted today.

**Renamed same day:** `--analyze-only`'s default output filename is
`user_stories.json`, not `behaviors.json` - Kenneth's own term for what the
first stage produces, and the name now used consistently in this log too.
The `--behaviors-file` flag name itself is unchanged (it already works for
either --spec's Analyzer or --gather-from's Gatherer).

**Two more mock raw-material sets added same day**, each a different
combination of source types than the login-portal one, so the Gatherer gets
exercised against more than one shape of input:
- `docs/mock-project-inputs-todo-list/` - business-dialogue.md ONLY
  (grounded in `examples/task-list-manager-spec.md`), deliberately the only
  file in its folder, to check the Gatherer works from a single messy
  conversation alone rather than several sources corroborating each other.
  Includes one thread of genuinely open, unresolved back-and-forth (whether
  there's a cap on total task count) that the real spec never answers -
  same "leave it out, don't invent an answer" test the login-portal
  dialogue's unresolved redirect-message question already covers.
- `docs/mock-project-inputs-resort-signup/` - meeting-notes.md +
  signup-wireframe.svg (grounded in `examples/resort-signup-spec.md`), a
  third combination (notes + wireframe, no dialogue thread). The wireframe
  covers every field in that spec, including the two Check-Duplicate
  buttons and the four-checkbox consent cascade; the notes carry the
  validation rules the wireframe's labels alone can't (re-check-on-edit,
  exact-match password comparison, the cascade's both-directions behavior)
  plus one open action item (Check Duplicate's behavior on a network
  failure) that isn't in the spec and shouldn't be invented.

## First live Gatherer run: quality problems, and the fix

The Gatherer's first real run against `docs/mock-project-inputs/` (all 4
login-portal source files, `--analyze-only`) completed successfully -
"Extracted 33 behavior(s)" - but the output wasn't usable as-is. Kenneth's
own bar for it: "the one generated needs to be close to what possibly comes
from the BAs" (a Business Analyst's hand-written user stories) - real
scattered raw material is one of two realistic ways user stories get
created (the other being a BA who already wrote them, which is what
`--spec` is for), so the Gatherer's output has to read like the second one
regardless of how messy the first one's input was.

**Symptoms found in the real 33-behavior output:**
1. **Statements were raw diagram-node fragments, not sentences** - e.g.
   `"Account found\nunder that type?"` and `"Login using =\nPassword or "
   "Passcode?"`, copied close to verbatim from `data-flow-diagram.md`'s
   mermaid source, literal `\n` line breaks included. Nothing like a BA's
   "If no account is found matching the selected user type, an error is
   shown."
2. **4 of 33 behaviors were exact duplicates** (same statement, category,
   AND source_hint) - the model walked the same diagram content more than
   once.
3. **`source_hint` was wrong almost everywhere** - every behavior was
   attributed to only `business-dialogue.md` or `meeting-notes.md`, never
   to `data-flow-diagram.md` or `login-wireframe.svg`, even for behaviors
   clearly lifted from the diagram (the tell-tale literal `\n`s).
4. **Category was badly skewed** - 30 of 33 behaviors were tagged
   `NON_FUNCTIONAL`, when almost none of the extracted content actually was
   (core login/account-lookup/redirect logic, which should mostly be
   POSITIVE/NEGATIVE) - the model defaulting to it as a catch-all under the
   messier, multi-document input.

**Fix, `gatherer_agent`'s instructions (`agents_def.py`):**
- A new "Writing `statement`" section, with a worked before/after example,
  explicitly telling the model a diagram/wireframe fragment is a SOURCE to
  read, never text to copy - rewrite every behavior as its own complete,
  grammatical sentence, and never include a literal line-break in
  `statement`.
- A new "Setting `source_hint`" section: track which source is actually
  being read from, per its `--- Source: <filename> ---` label, instead of
  defaulting everything to whichever source came first.
- A new "Setting `category`" section: judge each behavior the same way the
  Analyzer does, and explicitly call out that scattered input is not an
  excuse to default to NON_FUNCTIONAL.
- The existing "don't walk a diagram's decision points twice" guidance was
  already present but evidently not followed - reinforced with an explicit
  "do NOT walk the same diagram/flow twice" line.

**Fix, a code-level dedupe pass (`pipeline.py`)**, as a second line of
defense that doesn't depend on the model getting the prompt right every
time: `_dedupe_behaviors`, wired into `_stamp_behavior_ids` (so both
`analyze_spec` and `gather_behaviors` get it), drops an exact-duplicate
behavior statement (whitespace/case folded, same normalization `_normalize`
already uses elsewhere) before ids are assigned - so ids stay a clean,
gap-free `B1`, `B2`, ... sequence rather than skipping numbers where
duplicates would have been. Two genuinely different behaviors that merely
share most of their wording are never merged - only an exact (post-fold)
match is dropped.

5 new tests across `tests/test_pipeline_analyzer_retry.py` (dedupe wired
into `_stamp_behavior_ids`/`analyze_spec`, ids stay gap-free, whitespace/
case folding doesn't over-fire on genuinely different behaviors) and
`tests/test_pipeline_gather_behaviors.py` (the dedupe end-to-end through
`gather_behaviors`, using the real login-portal run's own duplicate
statements).

**Reverified against a live model same day - partial success, two new
regressions found (see next section).**

## Second live Gatherer run: sentence quality fixed, but under-extraction and a new `source_hint` failure

Kenneth reran the exact same command
(`--gather-from docs/mock-project-inputs/*.{md,svg} --analyze-only`) against
the fixed `gatherer_agent`. Result: **4 behaviors**, down from 33.

**What the first round's fix actually got right:** every one of the 4
statements is now a real, grammatical, BA-style sentence with no fragment-
copying, no literal line breaks, and no duplicates - e.g. *"If no account is
found matching the selected user type and entered username, an error is
shown stating no account matches."* Exactly the bar Kenneth set ("close to
what possibly comes from the BAs").

**Two new problems, both traced to the same first-round fix:**

1. **Severe under-extraction (4 vs. 33)** - the worked example added to
   the "Writing `statement`" section, meant to show the *shape* of a good
   sentence, appears to have been over-followed by this small local model:
   the model's first extracted behavior closely echoed the example's own
   phrasing, and extraction stopped far earlier than the material supports.
   `MIN_ANALYZED_BEHAVIORS = 3` (the existing under-extraction floor, shared
   with `analyze_spec`) is flat and content-blind to *how much* raw material
   went in - 4 behaviors cleared that floor without issue even though 4
   source documents worth of login-portal material should yield well over a
   dozen, the same as a single clean spec would.
2. **`source_hint` became a literal broken template string** -
   `"{{ .Source }}"`, verbatim, on all 4 behaviors. The prior instructions
   asked the model to attribute each behavior "to the diagram's filename"
   without pinning down a concrete format, and the model appears to have
   invented a Go-template-style placeholder rather than copying a real
   filename - arguably worse than round one's wrong-but-real filenames,
   since this is not even a valid filename.
3. Category was still imperfect on this small sample (2 of 4 read as more
   NEGATIVE-shaped than their assigned category), though the sample is too
   small to call this a confirmed regression the way #1 and #2 are.

**Fix, `gatherer_agent`'s instructions (`agents_def.py`), second round:**
- Rewrote "Setting `source_hint`" to require an EXACT format -
  `` `<filename>: <short quote or paraphrase>` `` - with a concrete worked
  example (`` `meeting-notes.md: 'session survives a refresh, does NOT
  survive closing the tab'` ``) and an explicit list of forbidden patterns:
  no `{{ ... }}`, `${...}`, or `<...>` template syntax, and no generic
  placeholder words like "source" or "document" - always a real filename
  copied verbatim from the `--- Source: <filename> ---` labels.
- Added a new "Read and use EVERY source document given" paragraph right
  after the intro, explicitly stating that a handful of source documents
  describing a real feature this size should still typically yield well
  over a dozen distinct behaviors total - being spread across several
  messier documents is not a reason to extract fewer than a single clean
  spec would.

**Fix, a code-level scaled validator (`pipeline.py`)**, as a second line of
defense against under-extraction that doesn't depend on the model following
the reinforced prompt every time: `_validate_gathered_behaviors_output`
replaces the shared, flat `_validate_analyzer_output` for `gather_behaviors`
only. It counts how many `--- Source: ... ---` documents were combined into
the input and requires at least `max(MIN_ANALYZED_BEHAVIORS, num_sources *
3)` behaviors back - 4 documents now demands at least 12, which would have
caught this run's 4-behavior result and triggered `_run_structured`'s
existing retry loop instead of silently accepting it.
`analyze_spec` is untouched - it still uses the original flat
`_validate_analyzer_output`, since a single already-written spec doesn't
have a source count to scale against.

4 new tests in `tests/test_pipeline_gather_behaviors.py`: the validator
rejects the real 4-behaviors/4-sources shape, accepts a plausible scaled
count, falls back to the flat floor for a single source document (so the
todo-list sample's lone `business-dialogue.md` isn't held to an inflated
bar), and confirms `gather_behaviors` wires this scaled validator (not the
shared flat one) into `_run_structured`.

**Reverified against a live model same day - under-extraction fixed, but a
third `source_hint` failure and a new `category` skew surfaced (see next
section).**

## Third live Gatherer run: under-extraction fixed, source_hint breaks a third way, category overcorrects

Kenneth reran the same command again. Result: **14 behaviors**, comfortably
clear of the scaled floor (12) - the under-extraction fix held. No exact
duplicates. Statement quality held too - all 14 are real, grammatical,
BA-style sentences.

**Two new problems, both traced to the round-two fix's own instructions:**

1. **`source_hint` broke a third, different way.** Round two's fix stopped
   the `"{{ .Source }}"` hallucination, but the same run's `source_hint`
   regressed to raw mermaid diagram node/arrow syntax copied verbatim -
   e.g. `"H{Account found\nunder that type?} --> I{Account found\nunder "
   "that type?}"` - no filename attribution anywhere, literal `\n`s
   included. Three live runs, three different broken shapes for the same
   field, despite an increasingly explicit prompt each time: (1) wrong-but-
   real filename, (2) a template placeholder, (3) raw diagram source with
   no filename at all.
2. **`category` overcorrected away from POSITIVE.** Round two's fix told
   the model not to default everything to NON_FUNCTIONAL; this run swung
   the other way - 0 of 14 behaviors were tagged POSITIVE, even ones that
   are plainly happy-path (a successful login redirecting to the right
   page, a valid stored session skipping the login form entirely).

A third, smaller issue: one behavior (a "user is both an admin and a
customer" statement) looks like a misread of the diagram's actual admin/
customer branching, which is already correctly captured as two separate
behaviors elsewhere in the same output - a content-accuracy issue, not a
formatting one, and left alone for now.

**Fix, `gatherer_agent`'s instructions (`agents_def.py`) - `category`
only:** rewrote "Setting `category`" with concrete POSITIVE examples (a
matching account found, a successful redirect) and an explicit warning that
zero POSITIVE behaviors is exactly as wrong as mostly-NON_FUNCTIONAL was in
round one - the fix for one skew must not create the opposite skew.

**Fix, a code-level safety net (`pipeline.py`) - `source_hint`, not another
prompt rewrite:** three different broken shapes across three live runs of
the same prompt section is a signal this field needs a deterministic
backstop the way `steps`/`expected_result` already have one, not a fourth
attempt at wording it correctly. `_clean_gathered_source_hint` checks
whether a `source_hint` actually starts with a real `<filename>: ` prefix
and contains no diagram debris (`-->`, `{`/`}`, a literal or real line
break) or template syntax (`{{`, `${`); if not, it flags the value with a
`[UNVERIFIED SOURCE - needs manual review]` prefix rather than fabricating
a filename or silently discarding the one clue a human has toward the real
source - the same flag-don't-strip principle used throughout this log.
Wired in via a new `_clean_gathered_behaviors`, called only from
`gather_behaviors` (not `analyze_spec`, whose own `source_hint` is a plain
quote/paraphrase of a single spec with no filename to check for - applying
this there would flag every legitimate result).

7 new tests in `tests/test_pipeline_source_hint_cleaning.py`: the valid
`<filename>: <quote>` shape passes through unchanged, all three real broken
shapes get flagged, a legitimately bracket-containing UI-label quote is
NOT mistaken for mermaid syntax (brackets vs. the curly braces mermaid
actually uses), `_clean_gathered_behaviors` touches only `source_hint`, and
`gather_behaviors` flags broken hints end-to-end.

**Reverified against a live model same day - `source_hint` flagging works
as intended, but `category` is still unpredictable (see next section).**

## Fourth live Gatherer run: source_hint flagging holds, category still swings wildly - moved to a deterministic fix

Kenneth reran the same command again. Result: **16 behaviors**, still clear
of the scaled floor, no duplicates, sentence quality still holding.

**`source_hint` flagging worked as designed:** several behaviors got real
`<filename>: <quote>` attribution (some with overly long quotes lifted
straight from meeting-notes dialogue rather than a short paraphrase, but
correctly attributed and not diagram debris), and every hint that was still
raw diagram/mermaid content got the `[UNVERIFIED SOURCE - needs manual
review]` flag exactly as intended - the safety net did its job.

**`category` swung to a third, different skew:** 14 of the 16 behaviors
were tagged NEGATIVE, including several that are themselves plain success
statements - e.g. *"a success message is shown stating the user is logged
in"* and *"the user is redirected to the admin page"*, both tagged NEGATIVE
despite describing a working happy path. Three live runs of this same
prompt section have now produced three unrelated skews (mostly
NON_FUNCTIONAL, then zero POSITIVE, then mostly NEGATIVE) - the model's
category judgment doesn't correlate with what the prompt says, the same
signal that `source_hint` gave after its own three broken shapes.

**Fix: moved off prompt tuning entirely, onto the same deterministic
pattern already proven for `TestCase.category`.** This codebase already
has `_reconcile_case` (`pipeline.py`), built earlier for the exact same
class of bug at the test-case level: a case's own `expected_result` text
says plainly whether it's describing a success or a failure, and
`_has_unnegated_error_signal`/`_has_success_signal`/`_looks_non_functional`
catch the unambiguous, textually-detectable contradictions and correct
`category` in code rather than trusting the model's tag. `_reconcile_behavior_category`
reuses those exact same three helpers against a `DraftBehavior`'s
`statement` instead of a `TestCase`'s `expected_result` - same logic, same
conservative "only fire on an unambiguous signal" rule, one level up the
pipeline. Wired into the shared `_stamp_behavior_ids`, so both
`analyze_spec` and `gather_behaviors` benefit (a self-contradicting
category is equally wrong regardless of which source produced it, the same
reasoning already applied to dedup).

8 new tests in `tests/test_pipeline_behavior_category_reconcile.py`: both
real failure shapes from this run (a success statement tagged NEGATIVE, a
successful redirect tagged NEGATIVE) get flipped to POSITIVE; a genuine
error statement stays NEGATIVE; an error statement wrongly tagged POSITIVE
flips to NEGATIVE; an ambiguous statement with no clear signal is left
alone; BOUNDARY and a correctly-flagged NON_FUNCTIONAL are never touched;
`gather_behaviors` reconciles end-to-end; and `analyze_spec` gets the same
benefit via the shared wiring.

**Reverified against a live model same day - a real, marked improvement.**

## Fifth live Gatherer run: category reconciliation works, two small gaps found and closed

Kenneth reran the same command. Result: **13 behaviors**, still clear of
the scaled floor. Category distribution came back as 4 POSITIVE / 5
NEGATIVE / 4 BOUNDARY / 0 NON_FUNCTIONAL - genuinely balanced for the first
time across five live runs, and every obvious happy-path redirect (a valid
session, a matching account type) landed correctly as POSITIVE. The
deterministic reconciliation fix worked as intended.

Two small gaps remained, both edges of what that fix covered rather than
new failures of the same kind:
1. One behavior - *"the application logs the user in and saves their name
   to the session storage"* - is a plain success statement but stayed
   NEGATIVE, because `_has_success_signal`'s "logged in" keyword is a
   literal substring match and "logs the user in" doesn't contain it.
2. Two behaviors that are plain error statements ("shows a generic error
   message indicating the specific failure") landed in BOUNDARY instead of
   NEGATIVE - `_reconcile_behavior_category` only corrected POSITIVE<->NEGATIVE
   and NON_FUNCTIONAL by design, so BOUNDARY was untouched even when the
   text was unambiguous.

**Fix, both in `pipeline.py`, no further prompt changes:**
- `_LOG_IN_SUCCESS_RE` (`\blog(?:s|ged)?\b(?:\s+\S+){0,4}\s+in\b`) added to
  `_has_success_signal` alongside the existing keyword list - catches
  "logs the user in"/"log them in" the same way "logged in" already did,
  while the required word boundary after "log"/"logs"/"logged" means it
  does NOT fire on "login" used as a noun ("the login page is shown").
- `_looks_like_boundary` (new, same shape as `_looks_non_functional`) plus
  a matching `elif` branch in `_reconcile_behavior_category`: a BOUNDARY
  tag is only reclassified (to NEGATIVE/POSITIVE off the same error/success
  signals) when the statement itself doesn't actually describe a boundary/
  edge condition (no "maximum", "blank", "whitespace", "exceeds", etc.) -
  so a genuine edge case (a blank field, a max-length passcode) is never
  touched even when it also mentions an error, the same "only correct an
  unjustified label" principle `_looks_non_functional` already uses for
  NON_FUNCTIONAL.

5 new tests in `tests/test_pipeline_behavior_category_reconcile.py`: both
real BOUNDARY-catch-all cases flip to NEGATIVE, a genuine blank-field
BOUNDARY case stays BOUNDARY even though it also mentions an error, a
success statement wrongly tagged BOUNDARY flips to POSITIVE, the "logs the
user in" phrasing gets caught, and "the login page" (noun usage) is
confirmed not to false-positive the new regex.

**Reverified against a live model same day - one new category false
positive found, plus a new content-accuracy failure in `source_hint` the
format check couldn't catch (see next section).**

## Sixth live Gatherer run: a redirect-to-login false positive, and source_hints recycled across unrelated behaviors

Kenneth reran the same command. Result: **18 behaviors**. Category balance
was mostly good again, but two new problems surfaced - both from the SAME
root cause (a signal that's a good heuristic in general but too blunt for
one specific shape of statement) or, for the second, a failure mode
`_clean_gathered_source_hint`'s format check structurally cannot catch.

**1. A `_reconcile_behavior_category` false positive:** *"If the user is
not authenticated, the system redirects them back to the login page..."*
was flipped from NEGATIVE to POSITIVE, because the bare `"redirect"`
keyword in `_SUCCESS_KEYWORDS` doesn't distinguish a redirect TO a
protected page (success) from a redirect BACK to the login page (a denial/
gating outcome) - and "not authenticated" isn't in `_ERROR_KEYWORDS`, so
nothing blocked the flip. This is the reconciliation fix's own first
false positive.

**Fix:** `_REDIRECT_TO_LOGIN_RE` (`redirect(?:s|ed|ing)?...to...login`)
carves the "redirect" keyword out of `_has_success_signal` specifically
when the redirect target is the login page itself - a redirect to *any
other* page (admin, shop, dashboard) still counts as success exactly as
before. None of the other `_SUCCESS_KEYWORDS` needed this carve-out, since
there's no failure-flavored reading of "logged in successfully."

**2. A new source_hint failure, invisible to the format check:** 10 of 18
behaviors - covering completely different things (session guards, account
lookup, login checks) - all got the exact same `source_hint`, verbatim: a
meeting-notes.md quote about the passcode field's 6-digit cap, unrelated to
any of them. Each individual string is well-formed (`_clean_gathered_source_hint`'s
format check has nothing to object to - it's a real filename and a
real-looking quote), so this passed silently: a content-accuracy failure a
format check can't catch by construction.

**Fix, a new list-level check (`pipeline.py`):** `_flag_repeated_source_hints`
counts how many behaviors in the same batch share an identical (already
format-clean) source_hint; 3 or more identical repeats gets every one of
them flagged `[REPEATED SOURCE HINT - needs manual review]` - a quote
that's genuinely this common across unrelated behaviors is a stronger
signal of "the model defaulted to one convenient citation" than of
corroboration. Two behaviors sharing one hint is left alone (plausible
corroboration of a single well-stated fact, per the Gatherer's own
instructions); already `[UNVERIFIED SOURCE ...]`-flagged hints are
excluded from the count so their own shared placeholder text doesn't
trip this check for an unrelated reason. Wired into `_clean_gathered_behaviors`
as a second pass after the existing format check.

6 new tests: 3 in `tests/test_pipeline_behavior_category_reconcile.py`
(the real redirect-to-login case now stays NEGATIVE, a redirect to a
protected page still flips to POSITIVE) and 4 in the new
`tests/test_pipeline_repeated_source_hint.py` (3+ identical repeats
flagged, 2 shared hints left alone, already-flagged hints not double-
counted, both checks compose correctly in `_clean_gathered_behaviors`).

A smaller, left-alone issue from the same run: one behavior's statement
("If a user with the incorrect username type and no username is entered
...") reads as garbled/self-contradictory - a content-quality problem with
no clean mechanical signal to catch it, unlike the two fixed above.

**Reverified against a live model same day - and a separate, unrelated
root-cause failure surfaced in between (see next section) before this
fix's own rerun.**

## An unrelated hard failure: Ollama's context window, not a code bug

Between rounds, a live rerun crashed outright with `RuntimeError: Gatherer
failed to produce valid structured output after 3 attempts`. The retry
warnings told the real story - a monotonic collapse across the three
attempts: **10 behaviors -> 2 behaviors -> 0 behaviors**, for the exact
same 4-file input each time.

That progression is the signature of Ollama silently truncating the
prompt, not a prompt-wording or model-quality problem: Ollama's default
context window is sized to available VRAM (commonly 4096 tokens under
24GB), truncation happens from the FRONT of the prompt with no error
returned, and this project's combined Gatherer prompt (the `gatherer_agent`
instructions plus all 4 source files) runs to roughly 27,000+ characters -
comfortably over a 4096-token window. Each retry appends the validator's
rejection message to the prompt, making it marginally longer, which pushes
even more of the front (the earliest source material, or the agent's own
instructions) out of the truncated window - explaining the 10 -> 2 -> 0
decline exactly.

**This has no fix inside this codebase.** Confirmed via Ollama's own docs:
the OpenAI-compatible `/v1/chat/completions` endpoint this project talks to
(see `llm_config.py`) has no per-request way to set context size - that's
an OpenAI API limitation Ollama doesn't work around for that endpoint. The
fix is a one-time environment change on Kenneth's machine:

```
setx OLLAMA_CONTEXT_LENGTH 16384
```

...followed by fully quitting and restarting Ollama. Verified working:
`ollama ps` afterward showed `CONTEXT 16384` for the loaded model, and the
very next Gatherer run succeeded cleanly (14 behaviors, no retries needed).

## Seventh live Gatherer run (post context-window fix): a malformed question-statement and near-duplicate content

With the context window fixed, the same command produced 14 behaviors with
solid `source_hint` attribution and no repeated-citation problem. Two
smaller content-quality issues remained:

1. One statement was phrased as an unfinished question rather than a
   declarative sentence: *"If the single shared input behaves differently
   depending on the 'Login using' toggle - password rules one way, 6-digit
   numeric the other?"*
2. A couple of near-duplicate behaviors (the same underlying fact, stated
   with different wording) slipped past the exact-match dedupe, since it
   only catches identical statement text.

Given the diminishing, sometimes-regressive returns from further prompt
tuning seen over the last several rounds, the recommendation (and what
shipped) was scoped deliberately narrow: fix the one new *mechanically
clean* signal (a statement ending in "?"), nudge the prompt with a worked
example for the existing corroboration instruction, and leave prose-level
near-duplicates/redundant phrasing as expected manual-review territory
rather than build a fuzzier, riskier dedupe.

**Fix, a new shared validator (`pipeline.py`):**
`_validate_behavior_statements_are_declarative` rejects any behavior whose
`statement` ends in `?`, with a message asking the model to rewrite it as
an assertion of what happens. Wired into both `_validate_analyzer_output`
(analyze_spec) and `_validate_gathered_behaviors_output` (gather_behaviors)
- the same "equally wrong regardless of source" reasoning already applied
to dedupe and category reconciliation - so a malformed question-statement
now triggers the existing retry loop instead of shipping silently.

**Fix, a prompt nudge (`agents_def.py`):** added a concrete worked example
to the Gatherer's existing "corroboration, not two behaviors" instruction -
naming the actual login-portal case (the same fact stated in both
business-dialogue.md and data-flow-diagram.md, written as ONE behavior) -
plus an explicit "check whether an earlier one already covers this" line.

7 new tests in `tests/test_pipeline_declarative_statement.py`: the real
question-statement is rejected, well-formed statements pass, trailing
whitespace after `?` doesn't dodge the check, both `analyze_spec`'s and
`gather_behaviors`' validators reject it, the retry loop recovers on a
second attempt, and an end-to-end `analyze_spec` run propagates the
rejection correctly.

**Reverified against a live model - the question-mark validator and
corroboration example both held up (11 behaviors, all declarative, no
crash), but a new false positive surfaced in the `source_hint` flag
itself (see next section).**

## Eighth live Gatherer run: source_hint flagging over-fired on genuine multi-line quotes

The run (checked remotely while Kenneth was away from his laptop)
completed cleanly - 11 behaviors, no crash, no question-mark statements.
But 9 of the 11 got flagged `[UNVERIFIED SOURCE - needs manual review]`,
and two of those (quoting meeting-notes.md: *"each of those pages checks
for a valid session matching its required role on load..."*) were
genuinely well-formed, correctly attributed quotes - not diagram debris.

**Root cause:** `_SOURCE_HINT_DIAGRAM_DEBRIS_RE` included a bare `\n`
(an actual newline character) as one of its debris signals, alongside
arrows and curly braces. That was meant to catch a diagram's own
line-broken label, but a real, longer quote is exactly the kind of content
that wraps across a line break in the model's own JSON string - the two
flagged quotes here were genuine, just long enough to wrap. Every real
mermaid-debris case observed across all the earlier live runs already had
an arrow or a brace alongside any line break; the bare newline check was
never actually load-bearing on its own; it just happened not to have
misfired yet.

**Fix (`pipeline.py`):** dropped the bare `\n` alternative from
`_SOURCE_HINT_DIAGRAM_DEBRIS_RE`, keeping the arrow, curly-brace, and
literal-backslash-`\n` checks (which is a different, still-needed signal -
a diagram's OWN `\n` sequence copied verbatim, as literal backslash-n
characters, not a real line break). 2 new tests: a genuine multi-line
quote is no longer flagged, and real diagram debris that also happens to
contain a line break is still flagged correctly (arrows/braces alone are
sufficient).

**Not yet reverified against a live model.**

## Ninth live Gatherer run: NON_FUNCTIONAL category skew on neutral-worded business logic

The run (after the eighth run's source_hint fix landed) completed cleanly,
but the category distribution was off in a new way: 10 of 13 behaviors
came back NON_FUNCTIONAL, including core business logic with no
error/success wording for `_reconcile_behavior_category` to key off -
e.g. *"If the input field contains data, the corresponding account is
looked up..."* and *"If the login using is Password, the Credential is
compared to the account.password (exact, case-sensitive)."*

**Root cause:** unlike the earlier category skews (fixes in the third and
fourth live runs), this one isn't mechanically detectable - these
statements have no unnegated error signal and no success keyword either
way, so the existing deterministic reconciliation correctly leaves them
alone (by design, it only fires on unambiguous signals). The skew traces
back to the source material itself: much of the login-portal behavior
comes from a data-flow diagram whose nodes reference `session` /
`sessionStorage`, and the model appears to be pattern-matching "mentions
session" to NON_FUNCTIONAL, rather than distinguishing a diagram's
ordinary guard/lookup/comparison steps (still core FUNCTIONAL logic) from
the actual mechanics of session storage/security (which genuinely are
NON_FUNCTIONAL).

**Fix (`agents_def.py`, prompt-only):** added a paragraph to the
Gatherer's "Setting `category`" instructions, reusing the
session-gating-vs-storage-mechanics distinction already established
elsewhere in the codebase (`pipeline.py`'s `_looks_like_session_gating_case`,
used for `TestCase.case_type`): a diagram's guard/lookup/comparison steps
are usually still FUNCTIONAL (POSITIVE or NEGATIVE) even when the diagram
labels mention "session" or "sessionStorage" as part of what's being
checked; NON_FUNCTIONAL is reserved for the mechanics of storage/security
themselves (encryption at rest, expiry policy), not for ordinary business
rules that merely read or write a session as one step. Two worked
examples are included (the credential-comparison case and a role-guard
redirect case) to anchor the distinction concretely, mirroring the same
login-portal material.

No code/logic change - the shared `_reconcile_behavior_category`
deterministic backstop is left as-is, since this class of skew has no
mechanical signal to catch it on. Full test suite reverified (188 passed,
unchanged - a prompt-string-only edit needs no new tests).

**Not yet reverified against a live model.**

## Tenth live Gatherer run (post category-skew fix): a truncated source_hint, and diagram branches read backwards

The category-skew fix from the ninth run worked cleanly - this run's 12
behaviors split POSITIVE/NEGATIVE with zero NON_FUNCTIONAL and zero
BOUNDARY. Two new, unrelated problems turned up instead:

**1. A truncated `source_hint` the existing format check couldn't catch.**
One behavior's `source_hint` cut off mid-quote inside a nested
parenthetical: `"business-dialogue.md: 'username not found gets its own "
"message ("` - an unclosed `(` and no closing quote at all. It has a real
filename prefix and no mermaid debris, so it passed the existing checks
cleanly despite being broken.

**Fix (`pipeline.py`):** added `_looks_truncated()`, a new signal checked
alongside the existing filename/debris/placeholder checks in
`_clean_gathered_source_hint`: a real, complete quote should have balanced
parentheses, and should close with the same quote character it opened
with. 3 new tests: the exact truncated-mid-paren case is flagged, a
genuine quote with a *balanced* parenthetical is not (must not
over-fire), and a quote missing its closing mark is flagged.

**2. Diagram/prose branches read backwards - a content-accuracy problem,
not a mechanical one.** Three behaviors got the THEN-clause of a
conditional backwards: one said a role mismatch "redirects to the
role-specific page" when meeting-notes.md's own cited quote says the
opposite ("redirect back to index.html"); another attached "for a valid
session" as the precondition to an account lookup that actually happens
during a login attempt, sourced from the data-flow diagram; a third was
self-contradictory ("the login form is displayed... and the user can
skip"), also diagram-sourced (already incidentally caught by the debris
flag). Two of the three trace to the diagram specifically; one is a
misread of meeting-notes.md prose that itself uses terse,
diagram-like "condition - outcome" shorthand.

**Fix (`agents_def.py`, prompt-only):** two additions to the Gatherer's
instructions. First, a new bullet treating a diagram's own text as the
LEAST reliable source when a prose source (business-dialogue.md,
meeting-notes.md) describes the same rule - base the behavior on the
prose's exact wording, use the diagram only to confirm it, and fall back
to the diagram alone only when no prose source covers that rule. Second,
a general instruction (not diagram-specific, since one of the three
failures was a plain prose misread) to re-read the exact source text for
which outcome follows which condition before writing any conditional
statement, rather than paraphrasing from a general memory of the rule's
shape - with a worked example built directly from the redirect-direction
failure above.

This is judgment-territory the deterministic reconciliation layer can't
reach (no error/success keyword signals a reversed conditional), so
unlike fix #1 above it's a prompt nudge, not a code check - the same
category of fix as the NON_FUNCTIONAL skew fix, with the same caveat that
prompt-only fixes for content-accuracy issues have had mixed durability
this session. Full suite reverified: 191 passed (188 + the 3 new
source_hint tests).

**Not yet reverified against a live model.**

## Eleventh live Gatherer run: broad quality regression, reverted the diagram/direction prompt additions

The run immediately after the tenth run's fixes landed came back
**worse than any prior run**, not narrowly broken:

- **Zero POSITIVE behaviors** out of 13 (5 NON_FUNCTIONAL, 3 BOUNDARY, 5
  NEGATIVE, 0 POSITIVE) - the exact failure shape the prompt's own
  category section explicitly calls out as "exactly as wrong as...mostly
  NON_FUNCTIONAL."
- Several statements were long, confusing run-on/compound sentences,
  violating the very first instruction to split compound statements into
  clean, single sentences.
- A new, more serious `source_hint` failure: several behaviors cited a
  real, well-formed quote that was simply the **wrong one** for that
  statement's content (e.g. a statement describing a successful
  role-matched redirect citing a quote about redirecting to the login
  page instead) - not a formatting problem the existing checks can catch,
  since the quote itself is genuine.
- The near-duplicate corroboration problem (two behaviors from the same
  protected-page role-check rule) was still present.

**Root cause (working theory, not confirmed):** the Gatherer's
instructions had grown, across ten rounds of fixes, to roughly 2,500
tokens of rules, bullets, and worked examples - on top of the source
documents themselves. The tenth run's two additions (diagram-vs-prose
priority, conditional-direction verification) added a further ~1,300
characters. Unlike the earlier Ollama context-window crash (a hard
failure with a monotonic count collapse across retries), this looked like
a small local model (llama3.2) losing track of and blending together too
many simultaneous constraints, rather than truncating input - the output
was long-winded and self-contradictory rather than short or absent.

**Fix:** reverted both of the tenth run's `agents_def.py` prompt
additions (the diagram-reliability bullet and the conditional-direction
check), returning `gatherer_agent`'s instructions to their ninth-run
length (~8,700 characters, down from ~10,000). The truncated-`source_hint`
code fix from the tenth run (`_looks_truncated`, `pipeline.py`) was kept -
it's a deterministic code check, unrelated to the prompt-length theory.
Diagram-vs-prose direction accuracy (finding #2 from the tenth run) is
back to being manual-review territory rather than a chased prompt fix,
consistent with the standing guidance that content-accuracy issues with
no mechanical signal are handled that way when a prompt nudge doesn't
hold. Full suite reverified: 191 passed, unchanged (a prompt-only revert
needs no new tests).

**Not yet reverified against a live model.**

## Twelfth live Gatherer run: reasoning leaked into structured output - root cause found: no temperature was ever set

The run right after the eleventh run's revert (same prompt, unchanged)
came back with a THIRD, unrelated quality profile: 30 behaviors (vs. 12-13
in recent runs), several with the model's own reasoning leaked directly
into `source_hint` - e.g. one read *"(This one is not an exact match - I
found a similar phrase in meeting-notes.md... However, I did find a
related statement in login-wireframe.svg)"* - and heavy over-extraction
from purely organizational meeting content (attendee questions repurposed
as behaviors) that the prompt explicitly says to ignore. The repeated-hint
and unverified-source flags correctly caught most of the resulting
mess (12+ of 30 behaviors flagged), so the safety net held, but the
underlying generation quality had clearly degraded further.

**The key signal:** this run used the *exact same* `agents_def.py`
instructions as the clean, well-categorized 13-behavior run two rounds
earlier - nothing in the prompt changed between them. Same prompt, three
very different quality profiles across three runs (clean → category-
skewed with zero POSITIVE → reasoning-leak/over-extraction). That
pattern - not a specific broken shape recurring, but the *shape itself*
changing every run on unchanged input - pointed away from the prompt and
toward the model's own sampling.

**Root cause:** grepping the whole pipeline (`llm_config.py`,
`agents_def.py`) turned up no `temperature` setting anywhere. Every one of
the 6 agents (Analyzer, Gatherer, Generator, Field Extractor, Reviewer,
Dev Questions) was running on Ollama's default sampling temperature
(~0.8) - tuned for varied/creative output, not a deterministic
extraction-and-classification task. That default plausibly explains a
meaningful share of this session's category-skew and content-accuracy
regressions being mis-diagnosed as prompt bugs: the same prompt can
legitimately produce different quality levels from run to run under a
high-temperature default.

**Fix (`agents_def.py`):** added a shared `_MODEL_SETTINGS =
ModelSettings(temperature=0.2)` and passed `model_settings=_MODEL_SETTINGS`
to all 6 `Agent(...)` definitions (the Agents SDK supports this directly
via `ModelSettings`, no extra plumbing needed). This is deliberately
applied pipeline-wide, not just to the Gatherer - every agent in this
pipeline does structured extraction/classification, not creative writing,
so the same low-variance argument applies to all of them equally. Full
suite reverified: 191 passed, unchanged (a model-settings change needs no
new tests - nothing about the code paths under test changed).

**Not yet reverified against a live model.**

## Thirteenth live Gatherer run (post temperature fix): clean overall, one deterministic reconciliation gap found and fixed

The first run after setting `temperature=0.2` came back markedly more
stable: 12 behaviors, a healthy POSITIVE/NEGATIVE split (6/5), no
repeated-source-hint flags, no leaked reasoning, no over-extraction -
only the usual 2 `[UNVERIFIED SOURCE]` flags on a legitimate quote
containing a JSON-object-literal `{ type, username }` (curly braces
correctly read as possible diagram debris, a defensible conservative
flag). This is consistent with the temperature-variance theory: same
prompt as two runs prior, much closer in shape to the clean baseline this
time.

One real bug did surface, and unlike the last few rounds it's fully
mechanical: a behavior stating *"...the system shows an error message if
the session role does not match the required role"* - an unambiguous
error statement - stayed tagged NON_FUNCTIONAL. Two near-duplicate
behaviors describing the same underlying rule were tagged POSITIVE
instead (the corroboration problem, still present, is a separate,
harder-to-fix issue).

**Root cause:** `_reconcile_behavior_category`'s NON_FUNCTIONAL branch
uses `_looks_non_functional(draft.statement)` to decide whether to even
attempt reconciliation, and that helper's keyword list includes
`"session"` and `"storage"` - tuned for a short field like
`broad_category`/`expected_result`, where naming "session" at all usually
IS the actual non-functional concern. A DraftBehavior's `statement` is a
full sentence, and legitimately mentions "session" constantly for
ordinary gating logic (a role check reading a session) without storage/
security mechanics being the point. So the deterministic reconciliation
this session already built to catch exactly this shape of bug was itself
being blocked by an overly broad keyword match - the same
session-gating-vs-storage-mechanics distinction already applied to
`TestCase.case_type` via `_looks_like_session_gating_case`, just not yet
applied here.

**Fix (`pipeline.py`):** added `_NON_FUNCTIONAL_KEYWORDS_STRONG` (the
existing keyword list minus `"session"`/`"storage"`) and
`_looks_strongly_non_functional()`. `_reconcile_behavior_category`'s
NON_FUNCTIONAL branch now only blocks reconciliation when a strong
keyword is present, OR when "session"/"storage" is present with no
error/success signal at all (still correctly NON_FUNCTIONAL, e.g. "the
session token is stored in sessionStorage, not localStorage") - a clear
error/success signal alongside "session"/"storage" alone no longer blocks
it. This is a code-level fix, not a prompt change, so it doesn't carry
the prompt-length/variance risk of the ninth and tenth rounds' attempts
at the same underlying problem. 4 new tests: the exact B10-shaped error
case now reconciles to NEGATIVE, the success-case mirror reconciles to
POSITIVE, a genuine storage-mechanics statement with no signal stays
NON_FUNCTIONAL, and a strongly non-functional statement (encryption)
stays NON_FUNCTIONAL even with error wording present. Full suite
reverified: 195 passed (191 + 4 new).

**Not yet reverified against a live model.**

## Fourteenth live Gatherer run: a runaway 2.5-hour generation with no cap, capped at the model-settings level

The very next login-portal run after the reconciliation fix above landed
ran for roughly 2.5 hours and ultimately failed with the familiar
under-extraction error: `RuntimeError: Gatherer failed to produce valid
structured output after 3 attempts. Last error: Only 10 behavior(s) were
extracted from 4 source document(s), which is implausibly few - expected
at least 12.`

**What was actually happening, confirmed live:** `ollama ps` showed the
model pinned at 100% CPU the whole time (never idle, so not a genuine
hang), and Ollama's own `server.log`
(`%LOCALAPPDATA%\Ollama\server.log` on Windows) showed `n_gen` climbing
steadily past 5,369 tokens at a fairly constant ~5 tokens/sec, for a task
whose entire output is a JSON list of a dozen-odd short behavior objects
- normally a few thousand tokens at most. So this wasn't a hang and
wasn't (by itself) the earlier Ollama context-truncation crash either -
it was one single generation running far longer than the task needed,
across all 3 retry attempts, each one both very slow AND still
under-extracting - consistent with the model spending most of its output
budget rambling or repeating instead of producing behaviors.

**Root cause:** nothing anywhere in the pipeline capped generation
length. `ModelSettings` (added for the temperature fix two rounds ago)
only set `temperature` - no `max_tokens`. Without a cap, a generation
that starts rambling or looping has no way to be cut off; it just runs
until it eventually produces something, hits Ollama's connection timeout
(30 minutes per attempt, so up to ~90 minutes worst case across 3
retries), or - as happened here - keeps going well past even that,
apparently exceeding the configured timeout ceiling as retries stacked.

**Fix (`agents_def.py`):** added `max_tokens=4096` to the shared
`_MODEL_SETTINGS`, applied pipeline-wide (all 6 agents) alongside the
existing `temperature=0.2`. 4096 is generous for this task's real output
shape - well beyond what even a 30-behavior `AnalyzerOutput` needs - while
still bounding worst-case generation time so a bad run fails fast and
retries, instead of grinding for hours. This is expected to help both
symptoms at once: faster failure/retry on a bad generation, and plausibly
better extraction quality too, since a capped generation can't spend its
budget on rambling instead of behaviors. Full suite reverified: 195
passed, unchanged (a model-settings-only change needs no new tests).

**Not yet reverified against a live model.**

## Fifteenth live Gatherer run (post max_tokens fix): confirms the cap works, one more reconciliation gap found and fixed

The next login-portal run after adding `max_tokens=4096` completed in
about 13 minutes - down from the previous run's 2.5 hours - and produced
15 behaviors. This confirms the cap is doing its job: no runaway
generation, no near-timeout retries.

Quality-wise, two things stood out:

1. **A heavy NEGATIVE skew (B6):** *"If the comparison succeeds, the
   system saves the User type and username to sessionStorage."* - a plain
   success-precondition statement - stayed tagged NEGATIVE.
   **Root cause:** `_SUCCESS_KEYWORDS` covered `"successfully"` and
   `"success message"` but never the bare verb `"succeed(s)"`, so nothing
   in `_has_success_signal` recognized this statement as a success signal
   at all, leaving the reconciliation with nothing to flip it. **Fix
   (`pipeline.py`):** added `_SUCCEED_RE` / `_NEGATED_SUCCEED_RE` /
   `_has_unnegated_succeed_signal()`, following the exact negation-window
   pattern `_has_unnegated_error_signal` already uses for `"error"` (check
   a short window immediately before the match for `"not"`/`"n't"`/
   `"never"`/`"fails to"`, rather than negation-checking the whole
   statement) - because "if the comparison does not succeed" is the
   opposite signal, not a success one. Wired into `_has_success_signal` as
   a new branch. 2 new tests: the exact B6 statement now reconciles
   NEGATIVE -> POSITIVE, and a negated-succeed counter-example ("does not
   succeed") correctly stays NEGATIVE, confirming the fix doesn't
   over-fire. Full suite reverified: 197 passed (195 + 2 new).

2. **Diagram-branch-direction inversions recurred (B9/B10):** two
   behaviors again had a conditional's THEN-clause backwards (e.g. "If
   session role is NOT Admin, renders admin page"), the same failure
   pattern seen in the tenth run. This happened on the prompt in its
   current, already-reverted state - no diagram-priority or
   conditional-direction wording present at all - which confirms this is
   model capability variance on this content, not something chaseable
   with more prompt instructions. Reaffirming the standing decision:
   left as manual-review territory rather than triggering another prompt
   round.

**Not yet reverified against a further live model run.**

## Sixteenth live Gatherer run: `max_tokens=4096` was too tight, raised to 6144

The next login-portal run after the "succeed(s)" fix ran for 10m35s and
completed all 3 retries, but every attempt failed the same way:
`RuntimeError: ... Invalid JSON when parsing model output`. `ollama ps`
during the run showed the same working pattern as before (100% CPU,
`UNTIL` resetting) - not a hang - and once it finished, the model
unloaded normally.

**Root cause, confirmed live via `server.log`:** the run's `print_timing`
line read `eval time = 634983.10 ms / 4096 tokens` - `n_gen` hit exactly
4096 and generation stopped there, cutting the reply off mid-JSON-string.
This particular extraction (15 behaviors, longer statements and
source_hints than the norm) genuinely needed more than 4096 tokens to
finish its JSON. Unlike the fourteenth run's rambling/under-extracting
failure, this wasn't the model wasting its budget - it was doing
legitimate work that didn't fit. And because `_run_structured`'s retry
resends essentially the same prompt, the identical cap truncated the
identical way on all 3 attempts, rather than varying retry to retry.

**Fix (`agents_def.py`):** raised `max_tokens` from 4096 to 6144 - real
headroom above a normal-but-larger extraction, while still bounding a
genuinely runaway generation to roughly 15 minutes per attempt at this
model's ~6-7 tokens/sec, instead of the uncapped 2.5-hour failure that
motivated the cap in the first place. Model-settings-only change, no new
tests needed. Full suite reverified: 197 passed, unchanged.

**Not yet reverified against a live model.**

## Seventeenth live Gatherer run: `max_tokens=6144` didn't fix "Invalid JSON" - real root cause was a repetition loop, fixed with `frequency_penalty`

The very next login-portal run after raising `max_tokens` to 6144 failed
again with the same `RuntimeError: ... Invalid JSON when parsing model
output`, on 2 of 3 attempts in one run and all 3 in a second run
immediately after. This was surprising: if the fix were purely about
giving enough room for legitimate content, the failure rate should have
dropped. It didn't.

**Root cause, confirmed live:** the Agents SDK redacts the actual model
output and the underlying Pydantic validation error in this exception by
default (`OPENAI_AGENTS_DONT_LOG_MODEL_DATA`, on by default to avoid
logging sensitive data for hosted-API calls - not a concern here, since
this is entirely local). Re-running with
`OPENAI_AGENTS_DONT_LOG_MODEL_DATA=0` set surfaced the real output for the
first time, and it showed the model stuck in a genuine repetition loop:
the SAME two behavior objects (identical statement, identical
source_hint) repeated verbatim roughly 80 times in a row, alternating,
never advancing to new content, until it hit the token cap and got cut
off mid-object - which is what actually broke the JSON. This is the same
underlying pathology as the fourteenth run's uncapped 2.5-hour rambling
failure; `max_tokens` only bounds how long a stuck generation is allowed
to run, it does nothing to stop the model from getting stuck in the first
place, so raising it just delayed where the cutoff happened without
addressing why the model was looping.

**Fix (`agents_def.py`):** added `frequency_penalty=0.4` to
`_MODEL_SETTINGS`, alongside the existing `temperature=0.2,
max_tokens=6144` - the standard sampling-level lever for exactly this
failure mode, since it directly discourages the model from re-emitting
tokens/phrases it has already used. Also reinforced `_run_structured`'s
generic JSON-invalid retry prompt (`pipeline.py`) to explicitly say "do
not repeat the same item more than once," as a fallback in case a single
penalty value doesn't fully eliminate the behavior. Full suite reverified:
197 passed, unchanged (model-settings/prompt-only change).

**Verified against a live model the same session:** the next run
completed cleanly on the first attempt - 16 distinct behaviors, no
repeated statements, valid JSON. Confirms the fix.

That same run surfaced one more real, narrow gap: 2 behaviors ("...the
system saves the user's type and username to sessionStorage.") were
plain successes with no other recognized signal anywhere in the
statement, so they stayed wrongly tagged NEGATIVE - the same shape of gap
as the earlier "succeed(s)" fix, just for "saves ... to sessionStorage"
instead of the bare verb. **Fix (`pipeline.py`):** added
`_SAVES_TO_STORAGE_RE`/`_has_unnegated_save_to_storage_signal`, following
the identical negation-window pattern as `_has_unnegated_succeed_signal`,
wired into `_has_success_signal`. This is narrowly scoped to that one
function and doesn't touch `_looks_non_functional`'s separate
"session"/"storage" NON_FUNCTIONAL guard - a genuine storage-mechanics
statement with no error/success signal is still left alone, confirmed by
a new test. 3 new tests: the exact B8/B9-shaped case flips
NEGATIVE->POSITIVE, a negated-save counter-example stays NEGATIVE, and a
genuine NON_FUNCTIONAL storage-mechanics statement is unaffected. Full
suite reverified: 200 passed (197 + 3 new).

The same run's remaining NEGATIVE skew (13 of 16 behaviors) was mostly
either correct (redirect-to-login/error cases) or the already-documented
diagram-direction-inversion issue (a handful of statements read
backwards, e.g. "does not have a valid session... renders the shop
page") - left as manual-review territory per the standing decision, not
chased further.

**Not yet reverified against a further live model run.**

## Eighteenth/nineteenth live Gatherer runs: worked-example leakage confirmed on two different specs, fixed by de-risking the prompt's own examples

**Symptom:** two fresh `--gather-from` runs, one against the task-list-manager
dialogue-only material and one against the resort-signup notes+wireframe
material (Kenneth's own request, to establish whether a single run's
oddity was a fluke or reproducible), both produced the exact same two
opening behaviors:
> "If no account is found matching the selected user type and entered
> username, an error is shown stating no account matches." / "A matching
> account is found and login proceeds."

Neither the to-do list nor the resort sign-up source material has
anything resembling a username/account-lookup flow - this is the
login-portal's own business rule. **Root cause, confirmed by inspection:**
these two sentences are lifted almost verbatim from `gatherer_agent`'s own
worked examples in its system prompt (the corroboration-merging example
and the `statement`-writing example both used real text from the
login-portal spec, because that's the material fix #13 onward had always
been developed and tested against). The local model wasn't hallucinating
from nothing - it was echoing the prompt's own illustrative text as if it
were content drawn from the actual `--- Source: ... ---` material handed
to it that call. Getting the SAME two sentences from two structurally
different runs (one single-document, one two-document) rules out
coincidence.

Two further, related problems surfaced in the same two runs:
- **A false answer to an explicitly open question.** The task-list run
  asserted "There is no cap on the total number of tasks someone can
  have" as a fact, when the dialogue explicitly leaves this unanswered
  ("let me ask dev and get back to you, don't want to guess on that
  one"). The Gatherer's existing "don't invent an answer to an unresolved
  item" instruction didn't stop this because the question was phrased
  conversationally rather than tagged "Open" the way the meeting-notes
  format's action items are.
- **Pattern over-generalization to an unstated field.** The resort-signup
  run invented a "Check Duplicate" button on the **Password** field,
  copying a rule the meeting notes state applies only to Email and Phone
  ("Same logic applies to the Phone field's own Check Duplicate button -
  identical rule, just the other field" - explicitly two fields, not
  three). Password's real, correctly-extracted rule (exact match against
  Confirm Password) is a completely different mechanism.
- **Category skew, a third shape.** The task-list run put everything but
  the two leaked behaviors in BOUNDARY; the resort-signup run put
  everything but the leaked pair in NEGATIVE, with the consent-checkbox
  cascade rules wrongly in NON_FUNCTIONAL (that's core UI-state business
  logic, not a load/performance/security/session concern - the same
  mistake category the ninth live run's fix was meant to prevent, just
  recurring in a new shape). Zero POSITIVE behaviors in either run.

Worth noting: the pipeline's own existing safety net (`_run_structured`'s
source_hint/repeated-hint flagging) already caught most of the resort-
signup instances (`[UNVERIFIED SOURCE - needs manual review]` /
`[REPEATED SOURCE HINT - needs manual review]` on 7 of the 12 behaviors,
including both leaked ones) - so this wasn't silently passing through
undetected, but a human still has to catch and remove it by hand, and the
underlying cause (the prompt teaching the model wrong content by example)
was still worth fixing at the source.

**Fix (`agents_def.py`, `gatherer_agent` only):**
1. Replaced both worked examples that used real login-portal text (the
   corroboration-merging example and the `statement`-writing example)
   with an entirely fictional "ticket refund" scenario that can't be
   confused with any of this project's three real example specs, and
   added an explicit up-front warning describing this exact failure mode
   and instructing the model to check its own output against the
   examples before finishing.
2. Also de-risked the `category`-setting section's login-specific worked
   examples (the exact phrase that leaked as the second bad behavior) the
   same way, and added an explicit new example clarifying that a UI-state
   cascade rule (e.g. a "select all" checkbox) is FUNCTIONAL, not
   NON_FUNCTIONAL.
3. Strengthened the "don't invent an answer to an open question" rule to
   also cover a conversationally-phrased unresolved question (not just
   ones formally tagged "Open"), and added a new rule against extending a
   rule stated for named things A/B onto an unstated, merely
   similar-looking C.
4. Explicitly called out, in the category section, that BOTH an
   all-BOUNDARY skew and an all-NEGATIVE/NON_FUNCTIONAL skew have been
   observed on real runs, asking the model to actively check its own
   category spread before finishing.

The Analyzer and Generator agents' own worked examples (which also use
concrete login-portal-shaped text, e.g. `generator_agent`'s `[B7]`
example) were NOT touched here - no leakage has been observed from
either of those agents yet, and they're a separate, lower-priority follow-
up if a similar pattern turns up in their output later, not assumed
guilty preemptively.

No test can exercise prompt *text* against a live model without one
running, so this fix has no new unit test (same as every other prompt-
wording-only change in this document) - full suite reverified anyway to
confirm nothing else broke: 213 passed, unchanged.

**First live verification found a NEW regression, traced to this fix's own
bulk, and corrected same-session.** Two fresh resort-signup runs against
the fix above: run 1 succeeded (no login-portal leakage - that part of the
fix held) but every one of its 9 behaviors collapsed into the same
self-contradictory, content-free template - "If the user submits a valid
X, the system will validate it and display an error message if it is
invalid" - applied uniformly across every field, including nonsensically
to non-submittable things like "Back to Login". Run 2 reproduced the same
templating and then hit a textbook repetition loop on attempt 1 (the exact
pathology `frequency_penalty` was added for in the Seventeenth live
Gatherer run) - the same ~8-statement cycle repeating verbatim roughly 8
times before the token cap cut it off mid-string. Neither pre-fix run
(against the same source material) had shown this templating, so it's new,
and traceable to this fix.

**Root cause:** the fix above added a substantial amount of new prose to
an already-long prompt for a small local model (`llama3.2:latest`, 4.1GB)
- a long up-front anti-leakage warning, an extra category-skew paragraph,
an extra open-question-phrasing rule, an extra pattern-over-generalization
rule, an extra NON_FUNCTIONAL/checkbox-cascade example. More instructions
apparently gave the model more surface to latch onto a "safe," repeatable,
near-content-free template instead of engaging with each field's actual
rule - consistent with this project's repeated lesson that this model
degrades with added prompt complexity (the temperature/context-window/
repetition-loop fixes above are the same lesson in different shapes).

**Correction, same session:** trimmed the prompt back down, keeping ONLY
the part that actually stops the leakage (the two worked examples that
used real login-portal text, replaced with a fictional "ticket refund"
scenario, plus the login-specific phrasing in the category section's
examples made generic) and cutting everything added on top of that
(the long preamble warning, the extra category-skew/open-question/pattern-
generalization paragraphs, the extra NON_FUNCTIONAL example). Net prompt
length ended up close to the original pre-fix size - this was a pure
swap-in-place of the two leaking examples, not a net addition. Full suite
reverified: 213 passed, unchanged (still prompt-text-only).

**Verified against a live model the same session:** a third resort-signup
run against the trimmed prompt produced 11 distinct, specific behaviors -
consent-checkbox requirement, Check Duplicate invalidating on edit,
email/phone/password/address validation, Back to Login discarding the
draft - with no login-portal leakage and no templating/repetition-loop
recurrence. Confirms the theory: the earlier fix's own bulk, not the
example-text content, was driving the new regression; trimming it back
down while keeping the actual leak-fixing swap resolved it.

Two smaller, ordinary-grade gaps remain in this run (not the same class of
problem as the leakage/templating bugs above, left as-is for now): one
behavior (B9) is tautological and misses the real bidirectional "Agree to
all" cascade rule (checking all four individually should auto-check
"Agree to all" too, not just the reverse); another (B10) implies the
Optional marketing-consent checkbox gets validated/blocks Submit, when the
meeting notes explicitly say Optional checkboxes never block Submit. The
category skew also persists unrelated to this fix - all 11 behaviors are
NEGATIVE, zero POSITIVE, same open issue as before.

### 26. Tackling run3's remaining gaps: one more code-level category signal added, two content bugs deliberately left to the model

Picking up the two "smaller gaps" noted in run3 above. Given how painfully
this same session already demonstrated that this local model degrades
(templating, repetition loops) as the Gatherer prompt grows, the approach
here was deliberately conservative: extend the deterministic, code-level
reconciliation where the fix is unambiguous and mechanical, and leave
alone anything that would require rewriting the model's own prose (i.e.
another prompt edit), even though that means two known issues ship
unfixed this round.

**Fixed (code-level, no prompt change):** run3's B2 - "If the user checks
the 'Agree to all' checkbox, all four individual checkboxes are checked
and the form is submitted." - is a plain success statement (the shortcut
checkbox correctly cascades and submission proceeds) but was tagged
NEGATIVE. `_has_success_signal` had no signal for form-submission wording
at all - every existing success keyword (`redirect`, `logged in`,
`successfully`, `succeeds`, `saves ... to sessionStorage`) is login/
session-flavored, and resort-signup's own material never mentions any of
those. Added `_has_unnegated_form_submitted_signal` (a `form is/was
[not/n't] submitted` regex, with the negation captured inline rather than
via a lookbehind window, since run3's own B1 - "the form is NOT submitted
and an error message is displayed" - puts the negator between "is" and
"submitted", not before "form"). Verified B1 stays NEGATIVE and B2 flips
to POSITIVE; both are now regression tests in
`tests/test_pipeline_behavior_category_reconcile.py`.

**Observed but deliberately left unfixed (content bugs, not category
bugs - would need a prompt change):**
- B9: "If the user submits a form without checking the 'Agree to all'
  checkbox, the 'Agree to all' checkbox is unchecked." is tautological/
  garbled and misses the real rule (checking all four individual boxes
  should also auto-check "Agree to all" - a bidirectional cascade, not
  just the reverse direction B2 already covers). No error/success keyword
  appears in this statement at all, so no reconciliation logic could touch
  it either way even if we wanted it to - fixing this means getting the
  model to write a different, more complete sentence, which is a prompt
  change.
- B10: "If the user submits a form with an invalid consent to receiving
  commercial advertising information, an inline error message is
  displayed..." misrepresents the Optional marketing-consent checkbox as
  something that validates and blocks Submit, when the meeting notes say
  Optional checkboxes never block Submit. This one DOES contain an
  unambiguous error signal ("invalid", "error message"), so it reads as a
  confident, well-formed statement - reconciliation has nothing to
  "correct" here since the category matches the (factually wrong) text.
  Same underlying problem as B9: the fix is teaching the model the right
  fact, not re-tagging its output.
- B11 ("the form is navigated away and any unsaved changes are
  discarded") was also considered for a new success signal, since it's a
  normal Back-to-Login flow, not a failure. Left alone: "navigated away"
  and "discarded" are exactly the kind of bare verbs that read as
  negative in other plausible contexts (a session/page being torn down
  because of an error), the same ambiguity `_SUCCESS_KEYWORDS` already
  had to guard `redirect` against with `_REDIRECT_TO_LOGIN_RE`. Adding a
  narrow-enough guard for this one case wasn't obviously safe, so it's
  left as model output rather than risk a wrong "correction" on a future
  run.

The category skew this run showed (11/11 NEGATIVE) is now explained,
not just described: B1, B4-B8 are genuinely NEGATIVE (validation-error
paths, correctly tagged), B2 is now fixed to POSITIVE by this fix, and
B3/B9/B10/B11 are the remaining ambiguous-or-wrong ones discussed above -
so the "skew" was never really a single bug, it was one real
category-reconciliation gap (now closed) plus several distinct content
issues that only look like a skew when counted together.

Tests: **215 tests, all passing** (2 new: the B2 flip and the B1
counter-example, both using run3's exact wording).

## Twentieth/twenty-first live Gatherer runs: the fixed frequency_penalty reduces but doesn't eliminate the repetition loop, so retries now escalate it

Kenneth re-ran the resort-signup Gatherer stage several more times
(`--analyze-only`, no other change since fix #26) specifically to gauge how
often the repetition-loop failure mode (first found and "fixed" via
`frequency_penalty=0.4` in the Seventeenth live run) still happens on the
exact same trimmed, de-leaked prompt:

- **Run3** (already covered above): clean, 11 distinct behaviors, no
  templating.
- **Run4**: no JSON error, but the model itself collapsed - 6 behaviors,
  ALL sharing one self-contradictory template ("If the user submits a
  valid X, the system will validate it and display an error message if it
  is invalid."), with none of run3's specific per-field detail. A silent
  content failure, worse in a way than a crash because nothing flagged it.
- **Run5**: went back to the loud failure mode - `_run_structured`'s
  attempt 1 (and, per the terminal, at least one further retry) hit the
  same "Invalid JSON: EOF while parsing a string" error, centered every
  time on the "Agree to all" consent-checkbox cascade content specifically
  - the same trouble spot across multiple runs now, not random.

Two clean-ish runs (run3, and eventually run5's retries recovering) out of
several attempts on identical input confirms what fix #17's frequency_penalty
was always a *mitigation* for, not a cure: a fixed 0.4 value lowers how often
the model spirals into repeating itself, but doesn't stop it from happening
on a genuinely unlucky sampling run, and case run4 shows it can even produce
syntactically-valid-but-garbled output instead of tripping the JSON-parse
safety net at all.

**Fix:** rather than raise the base `frequency_penalty` (which would make
ordinary, already-successful first attempts - the common case - more
stilted for no benefit, just to guard against a failure that's actually the
exception), `_run_structured`'s retry loop now escalates the penalty
specifically for retries: attempt 1 always runs the agent exactly as
configured in `agents_def.py` (unchanged behavior for the success path),
attempt 2 clones the agent with `frequency_penalty=0.6`, attempt 3 with
`0.8` (`_RETRY_FREQUENCY_PENALTY_ESCALATION` in `pipeline.py`). Uses
`agent.clone(model_settings=...)` (`openai-agents` SDK) plus
`dataclasses.replace` on the immutable `ModelSettings` so the original
agent object - and its `model_settings` - is never mutated; this matters
because `agents_def.py`'s agents are module-level singletons reused across
every call, so a mutation would leak into unrelated, later, non-retry
calls. Guarded with `getattr(agent, "clone"/"model_settings", None)` so the
lightweight `SimpleNamespace` fake-agent doubles already used across
`tests/test_pipeline_analyzer_retry.py` etc. keep working unchanged rather
than hitting `AttributeError` on a retry.

This is a pure code/config change - no prompt-text edits at all - so it
carries none of the "prompt bulk causes quality collapse" risk the
Eighteenth/nineteenth-run regression taught. It cannot fully eliminate this
failure mode (it's an inherent property of running a small local model,
not a bug with a complete fix), but it directly targets the actual
mechanism rather than just working around a fixed value that's already
been shown insufficient.

Tests: **222 tests, all passing** (7 new, in
`tests/test_pipeline_retry_frequency_penalty_escalation.py` - covers: attempt
1 uses the unmodified agent unchanged; attempt 2/3 escalate above the
configured baseline and above each other; escalation only touches
`frequency_penalty`, not `temperature`/`max_tokens`; the original agent's
`model_settings` is never mutated; a content-validation retry escalates
too, not just a JSON-parse retry; and the `SimpleNamespace` fake-agent
doubles from the existing retry tests still work with no crash).

**Not yet reverified against a live model** - the mechanism is unit-tested
in isolation, but whether it measurably reduces the real-world failure
rate can only be confirmed by running resort-signup (and the other two
specs) again live, several more times, and comparing the ratio of
clean/templated/JSON-error outcomes against this run's ~1-clean-in-3
baseline.

## Twenty-second live run: the escalation fix (0.4→0.6→0.8) traded a loud failure for a quiet one, softened to 0.4→0.5→0.6

Kenneth ran resort-signup two more times (`--analyze-only`) right after the
fix above was synced, specifically to see whether escalating
`frequency_penalty` on retries actually helps:

- **Run5 (post-fix):** attempt 1 succeeded outright - no escalation ever
  triggered. 6 behaviors, no crash, no exact-duplicate collapse, but a
  repetitive sentence scaffold and clearly under-extracted vs. run3's 11.
- **Run6 (post-fix):** attempt 1 failed content-validation ("Only 1
  behavior(s) were extracted from 2 source document(s), which is
  implausibly few"), so attempt 2 ran with the escalated
  `frequency_penalty=0.6`. That retry succeeded and produced a full,
  well-formed 12-behavior result - MORE behaviors than run3 - but B1 was
  "If the ticket has already been scanned at the gate, the refund request
  is blocked." - a near-verbatim echo of the Gatherer prompt's own
  **fictional** "ticket refund" worked example (deliberately written as
  fiction by the Eighteenth/nineteenth-run fix specifically so a leak
  could never be mistaken for real material - it worked exactly as
  designed there, this is a NEW instance of the same underlying leakage
  mechanism, just leaking the harmless fictional text this time instead of
  real login-portal content). Several other behaviors (B2-B5, B7) also
  cited `examples/resort-signup-spec.md` as their source - a file that
  isn't even part of this run's `--gather-from` input - correctly caught
  by the existing source_hint verification safety net as `[UNVERIFIED
  SOURCE - needs manual review]`, but that means roughly half this run's
  citations were fabricated, not just thin.

**Read:** the escalation fix did what it was built to do (stop the
JSON-parse crash, stop the exact-duplicate template collapse), but 0.6 on
the very first retry looks like too big a jump - a plausible mechanism is
that pushing the model harder away from repeating ITS OWN recent phrasing
makes reaching for genuinely different vocabulary (including the prompt's
own nearby example text) more attractive, not less. This is arguably a
worse failure than the one it replaced: a crash is impossible to ship by
accident, but a full, plausible-looking, schema-valid result with
fabricated content inside it is exactly the kind of thing that's easy to
miss on a casual read.

**Fix:** softened `_RETRY_FREQUENCY_PENALTY_ESCALATION` from
`[None, 0.6, 0.8]` to `[None, 0.5, 0.6]` - still escalates (so retries
aren't back to square one), just less aggressively, on the theory that a
smaller nudge is more likely to break a genuine repetition loop without
giving the model enough of a push to go reaching for unrelated content.
No test changes needed - the existing 7 tests in
`tests/test_pipeline_retry_frequency_penalty_escalation.py` only assert
relative ordering (attempt 2 > baseline, attempt 3 > attempt 2), not the
exact values, so they cover the softened numbers unchanged. All 222 tests
still passing.

**Not yet reverified live against this specific failure shape** (the
run6 leakage pattern) - the softened values are a reasoned guess based on
one observed data point, not a confirmed fix. Worth specifically watching
for on the next few runs that need a retry: does a leak like run6's B1
still happen at 0.5/0.6, less often, or not at all? Given how much
back-and-forth tuning this single mechanism has already needed live, it's
probably not worth a third round of hand-tuning the exact numbers without
a lot more data - if 0.5/0.6 still shows the same failure mode, the
bigger structural change already on the backlog (splitting the Gatherer's
work into smaller per-source-file calls, so no single generation has to
carry this much content at once) is a more promising direction than
continuing to chase the right penalty value.

**Run7 (post-softening, 0.5/0.6):** attempt 1 failed the same JSON-parse
way (repetition loop, this time centered on the "Check Duplicate"
content rather than "Agree to all" - confirming the failure isn't tied to
one specific piece of source text, it can land almost anywhere), so
attempt 2 ran with the softened `frequency_penalty=0.5`. That retry
produced 13 behaviors, the fullest yet, with **no leaked fictional
content and no fabricated external citations** - every `source_hint`
correctly pointed at `signup-wireframe.svg`, actually part of this run's
input. First real evidence the softened values avoid run6's specific
leakage failure mode, though still just one data point.

It did surface a different, genuine content problem: B12/B13 claimed the
Check-Duplicate-invalidates-on-edit rule applies to the Password and
Address fields too, when the source material is explicit that only Email
and Phone have a Check Duplicate button at all ("Same logic applies to
the Phone field's own Check Duplicate button - identical rule, just the
other field" - no equivalent sentence exists anywhere for Password or
Address). Confirmed by re-reading `meeting-notes.md` directly. This is a
straightforward fabrication - a real rule over-generalized to fields that
don't have the mechanic - not a phrasing/templating issue. Useful
validation, though: the existing `[REPEATED SOURCE HINT - needs manual
review]` flag correctly caught all four Check-Duplicate behaviors
(B10-B13) as citing the identical source quote, which is what prompted
the manual check that found the fabrication. The flag can't by itself
tell "legitimately reused across genuinely similar fields" apart from
"fabricated by over-generalizing" - that's a deeper semantic check than
citation-matching can do - but it correctly routes exactly this shape of
problem to a human for review rather than letting it ship silently. Left
as-is; the flag already does what a flag can do here.

## First live login-portal run of the session: the fictional "ticket refund" example leaks into a SECOND, unrelated spec - removed from the prompt entirely

Kenneth's first login-portal Gatherer run this session (`--analyze-only`,
softened 0.5/0.6 escalation already in place) hit the exact same
"Invalid JSON: EOF while parsing..." repetition-loop shape seen on
resort-signup - but this time the repeated, truncated content was the
Gatherer prompt's own fictional worked example: "If the ticket has
already been scanned at the gate, the refund request is blocked."
appeared twice in a row, attached to a `source_hint` quoting a real
(unrelated) session-refresh line from `meeting-notes.md` - a mismatched
citation on top of the leak itself.

This is the SECOND confirmed spec this fictional example has leaked into
(resort-signup run6, now login-portal on its very first run) - different
source material, different real content, same exact fictional sentence
both times. That rules out "specific to one spec's content" as an
explanation. The Eighteenth/nineteenth-run fix deliberately made this
example fictional specifically so a leak could never contaminate real
output with fabricated-but-plausible real-sounding content - that
worked, in the sense that what leaks is obviously fictional (tickets/
gates have nothing to do with login-portal OR resort-signup) rather than
misattributed real content. But "the leak is harmless when it happens"
turned out to be the wrong bar - a leak that gets caught by a human
reviewing the output is fine; a leak that ships unnoticed because it
reads as a plausible, well-formed JSON entry is still a real defect, and
two confirmed leaks in two different specs (plus the run6 investigation
already flagging this as a likely destabilization attractor) is enough
occurrences to stop treating "fictional, so it's contained" as sufficient
on its own.

**Fix:** removed the fictional "ticket refund"/"scanned at the gate"
worked-example text from `gatherer_agent`'s instructions entirely
(`agents_def.py`) - all three places it appeared (the corroboration/
dedup example, the fragment-to-sentence rewriting example, and the
POSITIVE-category example) - replaced with abstract descriptions of the
same rule/pattern that give no concrete narrative sentence for a
destabilizing model to grab onto and repeat. This is a net REDUCTION in
prompt length (7,476 -> 6,698 chars), consistent with the "less is more"
lesson from the Eighteenth/nineteenth-run regression - it removes content
rather than adding more of it, so it carries none of that regression's
risk. `source_hint`'s own worked example (`meeting-notes.md: 'session
survives a refresh, does NOT survive closing the tab'`) was left as-is -
it's a short quoted fragment illustrating a FORMAT rule, not a narrative
scenario describing a fictional feature's behavior, so it doesn't have
the same "complete story to copy" shape that made the ticket/refund
example attractive to repeat.

No test changes needed - the existing test suite doesn't assert on the
removed prompt text (it tests `_reconcile_*`/pipeline logic, not prompt
wording). All 222 tests still passing.

**Not yet reverified against a live model** - this was a same-day,
pre-emptive fix made specifically so Kenneth wouldn't spend real paid
API tokens (he's planning to test against OpenAI next) against a prompt
with a known, recurring contamination source still in it. Worth
specifically watching on the next few live runs (local or paid) for
whether the ticket/refund content (or any other narrative leak) still
appears - if the destabilization-attractor theory is right, removing the
concrete narrative should help across ALL specs, not just resort-signup
or login-portal specifically.

### 27. Switched the live model from local Ollama to OpenAI, and stopped a failed extraction from crashing with a raw traceback

Two separate, same-evening (2026-09-25) changes, done together but worth
keeping distinct:

**The model switch.** Every fix in this log up to #26 was chasing failure
modes (repetition loops, worked-example leakage, category miscounts)
produced by running against a small local model (`llama3.2:latest`, 4.1GB,
CPU-only). Tonight Kenneth switched the same, unmodified pipeline to
`gpt-4o-mini` via OpenAI's API - purely a `.env` change (`OLLAMA_BASE_URL`,
`OLLAMA_API_KEY`, `OLLAMA_MODEL`; the var names keep their `OLLAMA_` prefix
since `llm_config.py` just hands whatever's in them to an OpenAI-compatible
client, it doesn't care which provider they came from). Result across all
three example specs (resort-signup, login-portal x2, todo-list), each run
via `--analyze-only --debug`: **4/4 succeeded on attempt 1 of 3, zero
retries triggered, zero leaked/templated/duplicate content.** This is a
categorically different result from every local-model run this project has
seen - strong evidence the small local model, not the pipeline's prompts or
retry logic, was the actual source of the repetition-loop family of bugs.
One content-quality note, not a new bug: resort-signup's run9 had several
success-outcome statements ("...proceeds without errors") left tagged
NEGATIVE rather than POSITIVE - `_reconcile_behavior_category`'s
success-signal detection didn't catch these particular phrasings. Tracked
as a new backlog item below, not fixed tonight.

Two loose ends from getting `.env` working at all, worth noting for anyone
hitting the same thing: (a) the project had no `python-dotenv`/
`load_dotenv()` call, so a `.env` file did nothing on its own - added both
(`generate_tests.py` now calls `load_dotenv()` before the `spec_to_tests`
imports, since `agents_def.py`'s module-level `_MODEL = get_model()` reads
`OLLAMA_*` at *import* time, not lazily); (b) PowerShell's `Tee-Object
-FilePath` writes **UTF-16**, not UTF-8, and also fails outright if the
target directory doesn't exist yet (which it usually doesn't, since the
pipeline itself creates `<out-dir>/<spec>-gathered/` on first write) -
create the directory first (`New-Item -ItemType Directory -Force -Path
...`), and read these logs with `iconv -f UTF-16LE -t UTF-8` (or
PowerShell's own `Get-Content`, which handles the encoding natively) rather
than plain `grep`/`cat`, which silently find nothing.

**The clean-failure fix.** Separately, Kenneth asked: if user-story
creation fails outright (i.e. `_run_structured` exhausts all
`MAX_STRUCTURED_RETRIES` and raises `RuntimeError`), does anything tell the
person running this that no test cases were generated, or does it just
crash? Before tonight: just crash, with a raw Python traceback and no
indication that the behaviors file, `test_cases.md`/`.csv`, and (if
requested) `dev_questions.md` were all skipped as a result. Fixed with a
small `_create_behaviors_or_none()` wrapper in `generate_tests.py` around
both call sites of `create_behaviors()` (the `--analyze-only` path and the
combined-run path) that catches that specific `RuntimeError`, prints a
one-line summary to stderr, and returns exit code 1 instead of propagating
the exception - `--debug` still shows the full traceback for anyone
actually diagnosing the underlying failure, this only replaces the default,
non-debug experience. Not yet exercised against a real exhausted-retries
failure (none of tonight's live runs hit one), so this is unit-tested only
so far (see Testing below), same caveat as several fixes above at the time
they landed.

### 28. Closed item 9's category-skew gap: two new success phrasings, one new destination-agnostic access-denial signal

Fixes both directions of the signal-detection gap fix #27 flagged as item 9,
built directly from the exact behaviors that surfaced each, re-fetched from
Kenneth's device rather than relying on the earlier summary of them:

**resort-signup run9 (NEGATIVE that should be POSITIVE), two new phrasings
added to `_has_success_signal`:**
- B3/B5 ("...they can proceed to submit the form unless they edit the email
  afterward." / "...they can proceed to submit unless they edit the phone
  number afterward.") - new `_CAN_PROCEED_TO_SUBMIT_RE` (`can proceed to
  submit`). Naturally excludes "cannot proceed"/"can't proceed" without a
  separate negation-window guard, since neither matches `\bcan\s+proceed\b`
  at all.
- B8/B10 ("...account creation proceeds without errors.") - new
  `_PROCEEDS_WITHOUT_ERROR_RE` (`proceeds ... without errors`). This is the
  OPPOSITE polarity of a bare "error" mention, which nothing in
  `_has_success_signal` could read as a success signal before this - it
  isn't caught by `_NEGATED_ERROR_RE` either, since that guards
  `_has_unnegated_error_signal`, a different function, and only recognizes
  negators immediately before the literal word "error".

**login-portal run3 (POSITIVE that should be NEGATIVE), one new signal
added to `_has_unnegated_error_signal`:**
- B12/B13 ("When accessing admin.html, if there is no valid session
  matching an Admin role, users are redirected back to index.html without
  any message shown." and the shop.html/Customer equivalent) - new
  `_has_unnegated_access_denial_signal` (`_ACCESS_DENIAL_CONDITION_RE` +
  `_REDIRECT_RE`). The existing `_REDIRECT_TO_LOGIN_RE` guard only
  suppresses the bare "redirect" success keyword when the destination is
  literally "login" - it missed a redirect to some OTHER page (index.html
  here) gated on an explicit invalidity condition ("no valid session",
  "wrong role", "not authenticated", etc.). Checked independently of the
  destination page name, since that's a denial outcome regardless of which
  page the user lands back on.

Deliberately left alone (not a fix target): B13's form-clear-on-back-nav
statement (genuinely ambiguous, not a clear success/error signal) and B9's
missing cascade rule / B11's ambiguous category from item 8 (content bugs
needing better model prose, not another signal-detection rule).

Both new signal functions feed into the same `_has_success_signal`/
`_has_unnegated_error_signal` calls already shared by
`_reconcile_behavior_category` (behavior-level) and `_reconcile_case`
(test-case-level), so this fixes both the Gatherer/Analyzer-stage category
and any test case a mistagged behavior would otherwise have produced.
6 new tests added to `tests/test_pipeline_behavior_category_reconcile.py`,
built from the real B3/B5/B8/B10 (resort-signup run9) and B12/B13
(login-portal run3) statement text, plus one guarded counter-example
(an invalidity condition with no redirect at all must not trigger the new
access-denial signal). Not yet re-verified against a live OpenAI run -
these were built from behaviors already captured in fix #27's runs, not a
fresh run against the fixed code.

### 29. Item 9/13, second pass: access-denial signal now overrides a preceding "successfully" precondition

Live re-verification (item 13) of fix #28 surfaced a real miss on fresh
login-portal Gatherer output (run4, B9/B10): "After successfully logging in
as an Admin, if they navigate to admin.html without a matching role in
sessionStorage, they should be redirected back to index.html." got wrongly
flipped NEGATIVE->POSITIVE. Two gaps, both fixed here:

- `_ACCESS_DENIAL_CONDITION_RE` didn't recognize "without a matching
  role"/"no matching role" as an invalidity condition - only "no
  valid"/"not valid"/"invalid"/"no match"/"does not match"/"wrong
  role"/"not logged in|authenticated" were covered. Added
  `\bwithout\s+(?:a\s+)?matching\b|\bno\s+matching\b` as its own
  alternative (not folded into the existing "no match" pattern, since that
  requires "match" as a bare noun, not "matching" as an adjective).
- More fundamentally: even with that widened, `_has_success_signal` would
  still have flipped this case, because the bare `"successfully"` keyword
  in `_SUCCESS_KEYWORDS` fires unconditionally via its
  `other_success_keywords` check, before the redirect/denial logic is ever
  reached - "successfully" here describes the PRIOR login (a true
  precondition), not the actual outcome the statement is testing (the
  denial-redirect). Fixed by checking
  `_has_unnegated_access_denial_signal` FIRST in `_has_success_signal` and
  returning `False` outright when it fires - the same "denial condition
  overrides an otherwise-generic success keyword" reasoning already applied
  narrowly to the bare "redirect" keyword via `_REDIRECT_TO_LOGIN_RE`,
  generalized here to any access-denial condition and extended to
  "successfully" too.

4 new tests added to `tests/test_pipeline_behavior_category_reconcile.py`:
the real B9/B10 statements (both now correctly stay NEGATIVE), plus two
guarded counter-examples confirming the new check doesn't over-suppress a
genuine "successfully" success statement with no access-denial condition,
and doesn't over-match a redirect gated on a MATCHING (not "without
matching") role, which is a legitimate success case. Closes item 13 and
item 9 for real this time (pending yet another live re-verification, same
standing caveat as fix #28's own "not yet re-verified" note - the pattern
of finding one more phrasing gap on the next live run is now expected
often enough that this caveat should just be assumed going forward, not
re-stated as if it were a surprise).

### 30. Item 14: a duplicate-rate diagnostic per review round, not an auto-stop heuristic

Item 14's resort-signup `--max-review-rounds 4` run showed a non-monotonic
pattern that rules out a naive auto-stop rule: round 3 found 8 gaps but
only added 1 new case (7 duplicates, 87.5%), which looks like near-
convergence - but round 4 immediately after found 6 gaps and added 14 new
cases (0% duplicates), the most productive round of the four. An
early-stop heuristic keyed off a single low-yield round (like round 3)
would have cut the run off right before its most useful round.

Rather than guess at a stopping rule from two data points, added
`_duplicate_rate(num_gaps, num_new_cases)` - a small, pure function that
computes what fraction of a round's reported gaps were duplicates of
already-existing cases (as opposed to genuinely new coverage) - and wired
it into `run_pipeline`'s existing per-round log line: `Review round N: X
gap(s) found, Y case(s) added (Z% duplicates)`. This is diagnostic only,
not an automatic stopping decision - it gives a person deciding whether to
raise `--max-review-rounds` further (or stop) the actual trend to look at,
rather than just the raw gap count, which run10/run4's investigation
(items 13/9) already showed can be misleading on its own (a "gap found"
doesn't distinguish new coverage from a duplicate the Reviewer re-flagged).
Whether to eventually build a real auto-stop rule on top of this signal -
and what that rule should be - is left as its own future decision, not
decided here.

5 new tests in `tests/test_pipeline_duplicate_rate.py`, built from the
real round 1/3/4 gap/case counts from the `--max-review-rounds 4` run, plus
guarded edge cases (zero gaps must not raise `ZeroDivisionError`; more new
cases than gaps, since `fill_gap` can emit more than one case per gap, must
clamp to 0.0 rather than go negative).

### 31. Item 14, part 2: wired the duplicate-rate diagnostic into an actual auto-stop heuristic

Kenneth decided to go with the code-level stopping condition (option (b)
from item 14/fix #30) rather than a Reviewer prompt change. Added
`_has_converged(duplicate_rates)`: True once the most recent
`_HIGH_DUPLICATE_ROUNDS_TO_STOP` (2) review rounds, back to back, each had
a `_duplicate_rate` at or above `_HIGH_DUPLICATE_RATE_THRESHOLD` (0.75).
Requiring TWO CONSECUTIVE high-duplicate rounds, not just one, is the
whole point and is deliberately built from the real run's counter-example:
resort-signup's round 3 (8 gaps, 1 added - 87.5% duplicates) looked like
near-convergence on its own, but round 4 right after was the most
productive round of the four (6 gaps, 14 added - 0% duplicates). A
single-round trigger would have stopped the run right before its best
round; the two-in-a-row requirement specifically does not fire on that
exact sequence (verified as its own test).

Wired into `run_pipeline`'s review loop: after each round's
`_duplicate_rate` is computed and logged, `_has_converged` checks the
running list of rates, and if it fires, the loop logs why and breaks
rather than spending more (real, costed) LLM calls on `--max-review-rounds`
that are unlikely to add much. Threshold (0.75) and patience (2 rounds)
are plain module constants for now, same style as `MAX_STRUCTURED_RETRIES`/
`MAX_REVIEW_ROUNDS`, not new CLI flags - easy to promote to a flag or tune
later if a real spec shows the defaults need adjusting, but not worth the
surface area on one data point.

7 new tests: 5 unit tests for `_has_converged` in
`tests/test_pipeline_duplicate_rate.py` (including the exact real sequence
that must NOT trigger early-stop, and a plain two-consecutive-high-rounds
case that must), plus 2 end-to-end tests in the new
`tests/test_pipeline_reviewer_convergence.py` proving `run_pipeline` itself
actually stops calling `review_cases` early when convergence is detected
(asserting the mocked call count), and that it does NOT stop early on the
real run's exact non-monotonic 4-round shape reproduced with mocks.

Not yet verified against a real live run - built and tested from the
already-captured `--max-review-rounds 4` numbers, not a fresh run against
this new code. Worth watching the per-round log line (now includes the
duplicate percentage) on the next live e2e run to see whether the
heuristic fires sensibly or needs its threshold/patience adjusted.

**First live verification (2026-09-28, same session):** todo-list rerun
with `--max-review-rounds 6` (cached `run1.json` behaviors) - 13 behaviors
-> 67 test cases (up from 59 at the 2-round default), all 6 rounds ran, the
heuristic did NOT fire:

| Round | Gaps found | Cases added | Duplicate % |
|---|---|---|---|
| 1 | 4 | 3 | 25% |
| 2 | 4 | 2 | 50% |
| 3 | 5 | 2 | 60% |
| 4 | 5 | 2 | 60% |
| 5 | 4 | 5 | 0% |
| 6 | 4 | 2 | 50% |

Correct behavior, not a miss: rounds 3-4 came close (60% each, back to
back) but stayed under the 75% threshold, so the loop kept going - and
round 5 right after was the most productive round of the six (0%
duplicates, 5 cases added). If the threshold had been looser (say 50%
instead of 75%), rounds 3-4 would have triggered an early stop right
before that same best-round pattern seen in fix #30's resort-signup data -
a second, independent real-run confirmation that the two-in-a-row-at-75%
design specifically avoids this failure mode, not just on the one dataset
it was built from. Net: the heuristic hasn't yet been observed actually
firing on a real run (todo-list, like resort-signup, still doesn't
converge even at 6 rounds), but it has now been observed correctly NOT
false-positiving on a second, independent spec - worth continuing to watch
for an actual trigger on a future run before concluding the threshold/
patience defaults are well-tuned either way.

## Testing

Every fix above shipped with unit tests built directly from the real
captured failure case (not synthetic examples), plus at least one guarded
counter-example proving the fix doesn't over-fire on legitimate output (see
`tests/test_pipeline_reconcile.py`, `tests/test_pipeline_analyzer_retry.py`,
`tests/test_pipeline_xml_tag_debris.py`, `tests/test_pipeline_brace_wrapped_string.py`,
`tests/test_pipeline_gapfill.py`, `tests/test_pipeline_field_enumeration.py`,
`tests/test_output_writer.py`, `tests/test_pipeline_behaviors_cache.py`,
`tests/test_generate_tests_cli.py`, `tests/test_source_gathering.py`,
`tests/test_pipeline_gather_behaviors.py`,
`tests/test_pipeline_behavior_category_reconcile.py`,
`tests/test_pipeline_duplicate_rate.py`,
`tests/test_pipeline_reviewer_convergence.py`).
Current suite: **245 tests, all passing**, runnable without a live Ollama
instance (no LLM calls in the test suite itself).

## Next up

1. ~~Rerun all 3 example specs fresh (login-portal, resort-signup,
   task-list-manager) to verify fixes #19 and #20 live.~~ Done overnight
   2026-09-24: login-portal and task-list-manager came back clean; the
   fresh resort-signup run surfaced two more real leaks, now fixed as #21
   and #22 above.
2. ~~Cache the Analyzer's output (`behaviors`) to disk after `analyze_spec()`
   runs, and add a `--behaviors-file` CLI flag to resume from there.~~ Done
   2026-09-24 - see "Cache the Analyzer's output" section above.
3. ~~Expand `_TEXT_FIELD_STATES` with more invalid/format-specific states
   (Kenneth, this session): float, non-English/Unicode characters, symbols,
   email format, non-email format, mobile number format, postal code
   format, country code format.~~ Done 2026-09-25 - see fix #25 above.
   float/unicode/symbols went into `_TEXT_FIELD_STATES` globally; the four
   format-specific ones went behind a new opt-in `DraftFieldSpec.format_hint`
   instead, so they only apply to a field the spec actually states takes
   that format. **Not yet reverified against a live model** - see fix #25's
   closing note; worth checking on the next live run whether the Field
   Extractor actually sets `format_hint` correctly (resort-signup's Email
   field is the obvious live test case).
4. Rule 3 (cross-type username mismatch coverage) - the remaining half of
   the "Observed but not (yet) fixed" FUNCTIONAL gap above, since rule 5 is
   now built (fix #15). Needs per-type identity values captured during
   field extraction, which is a bigger change than fix #15 reused.
5. Revisit mode-dependent PARAMETER field splitting (`Credential` ->
   `Password`/`Passcode`) if it turns out to matter for another spec, with
   one of the two follow-up approaches noted above (or the "hardcode this
   pattern" option) rather than another reactive retry-validator patch.
6. ~~`docs/mock-project-inputs/` - built as raw input for the next pipeline
   stage ("gather from resources like chats/screenshots -> build user
   stories"). Not yet fed through anything.~~ Done 2026-09-24 - the Gatherer
   stage (`gather_behaviors`, `--gather-from`) now reads exactly this
   material. See "A Gatherer stage" section above. Still needs a real run
   against it (mocked in tests so far, per that section's closing note).
7. ~~Real per-run logging (Kenneth, raised 2026-09-25, explicitly deferred -
   not urgent, do later): a `--log-file` flag ... and flip
   `OPENAI_AGENTS_DONT_LOG_MODEL_DATA` ...~~ Done 2026-09-25, same evening -
   no `--log-file` flag was needed in the end: `generate_tests.py --debug`
   already gated the Agents SDK's own raw model logging behind
   `OPENAI_AGENTS_DONT_LOG_MODEL_DATA`, so the only missing piece was
   setting that env var (now in `.env`) and piping `--debug`'s output to a
   file (`... 2>&1 | Tee-Object -FilePath run.log` in PowerShell - mind the
   UTF-16 encoding and pre-existing-directory gotchas noted in fix #27).
   The per-agent-call summary line (which agent, attempt number, duration,
   outcome) is still not built - worth doing if the raw DEBUG firehose
   proves too noisy to scan by eye on a real failure.
8. ~~Resort-signup B9/B10/B11 content bugs (fix #26's deliberately-deferred
   half)...~~ Confirmed resolved 2026-09-28, no code change needed - the
   OpenAI switch (fix #27) alone fixed all three, visible in today's run10
   (item 13's resort-signup check): the old garbled/tautological B9 is now
   two clean, distinct statements (checking "Agree to all" cascades down to
   all four boxes; unchecking any individual box cascades back up to
   uncheck "Agree to all" - a genuine bidirectional rule, not a tautology);
   the old B10's Optional-checkbox-blocks-Submit misstatement is now
   correctly scoped to "at least one Required consent checkbox"; and the
   old B11's ambiguous "navigated away/discarded" wording is now clear and
   correctly tagged POSITIVE. Confirms these were genuinely local-model
   content quality issues, not something needing a prompt or code fix.
9. ~~`_reconcile_behavior_category`'s signal detection has gaps in BOTH
   directions (fix #27, found across resort-signup run9 and login-portal
   run3, both on OpenAI)...~~ Done 2026-09-28, in two passes - fix #28 (two
   new success phrasings, one new destination-agnostic access-denial
   signal), then fix #29 (the "successfully"-as-precondition gap fix #28's
   own live re-verification surfaced, item 13). Given the pattern of each
   live re-verification finding one more phrasing gap so far, treat this as
   "done for the cases seen to date," not "provably complete" - the next
   live run may well surface another one.
10. ~~The actual end-to-end run (Gatherer/Analyzer -> Generator -> Reviewer
    -> test_cases.md/csv) has NOT been exercised against OpenAI at all
    yet...~~ Done 2026-09-25, same evening - login-portal run3's cached
    behaviors (`--behaviors-file`, no `--analyze-only`) went straight
    through Generator -> 2 Reviewer rounds -> `test_cases.md`/`.csv`: 15
    behaviors -> 37 test cases, no crashes, no retries. Round 1 found 4
    gaps (cases added); round 2 found 5 more, but the existing dedupe logic
    correctly skipped one as already covered ("Skipped duplicate gap-filled
    case ... already covered by an existing case") rather than adding a
    redundant TC. Spot-checked the actual `test_cases.md` content (not just
    the summary line): concrete preconditions, numbered steps, specific
    expected results, correct category/priority/type/permutation metadata
    on every case reviewed - genuinely usable output, not a rough draft.
    Confirms the OpenAI switch holds up through the full pipeline, not just
    the Gatherer stage. resort-signup and todo-list haven't had the same
    e2e treatment yet - same pattern, whenever there's time
    (`--behaviors-file` pointing at their respective `run9.json`/
    `run1.json`, no `--analyze-only`).
11. Portfolio write-up (Kenneth, 2026-09-25): draft a section on this
    project for the portfolio site (kennethchuaqiyang.github.io) titled
    something like "From business discussion to test case." Decided on a
    hybrid structure rather than either a single polished summary or a raw
    dump of this journal:
    - A short (3-4 paragraph) narrative on the portfolio page itself,
      matching the existing portfolio's write-up style/format (not this
      log's own voice) - the architecture in brief (raw scattered material
      -> Gatherer -> user stories -> Generator/Reviewer -> test cases), plus
      one or two concrete "here's a real bug I found and how I fixed it"
      vignettes pulled from this log (e.g. the fictional worked-example
      leak, or the frequency_penalty repetition-loop chase) to demonstrate
      debugging judgment, not just that a pipeline was built.
    - A clear link from that page to the full journal (this file, or a
      published version of it - see fix #27's `mrg0ea0dqp3` artifact,
      titled "Spec-to-Test-Case Agent - Fix & Enhancement Journal," which
      already exists as a narrative companion to this file) for anyone who
      wants to evaluate the engineering depth directly.
    Rationale: a recruiter skimming gets the 30-second version; someone
    evaluating seriously gets the real story instead of a sanitized one.
    **Drafted 2026-09-28** (in chat, not yet published to the site) - pulled
    the actual portfolio site's verbatim write-ups first to match voice
    precisely (heading as a problem framed, two narrative paragraphs, a
    **Details** paragraph, a **Bug found** callout, links), then revised
    twice on Kenneth's feedback: (a) reframed as a personal side project,
    not a client-facing pitch, since this isn't used on actual work: (b)
    corrected the project's actual scope - it's not just "raw material ->
    test cases", it's the full intended chain business discussion -> draft
    user story -> test case -> automation script, with the last leg
    currently only a deterministic Playwright-skeleton generator
    (`playwright_stubs.py` - steps/expected-results filled in, selectors
    left as TODOs), not yet LLM-driven. Kenneth's call: leave the
    automation-script part of the write-up as-is for now and revisit it
    once that stage is actually built out, rather than describing a future
    state as if it's finished. Still needed before publishing: the actual
    URL for the `mrg0ea0dqp3` journal artifact (not available in this
    session), and pushing this project to an actual GitHub repo (it
    currently only exists in the cloud sandbox and Kenneth's Windows
    machine, synced by hand - see the top of this file).
12. resort-signup and todo-list's first e2e runs on OpenAI (2026-09-28,
    closing out item 10 for both remaining specs - login-portal was already
    done in fix #27):
    - resort-signup: 14 behaviors -> 174 test cases, no crashes/retries.
      Category split skewed toward NEGATIVE (111 NEGATIVE / 32 POSITIVE / 31
      BOUNDARY / 0 NON_FUNCTIONAL) - partly expected for a form with this
      many required fields, but likely inflated some by item 9's known
      signal-detection gap.
    - todo-list: 13 behaviors -> 59 test cases, no crashes/retries. Category
      split much more balanced (34 POSITIVE / 22 NEGATIVE / 3 BOUNDARY / 0
      NON_FUNCTIONAL) - consistent with todo-list having far fewer
      validation rules than resort-signup's form.
    Both clean runs. More notably, **the same pattern showed up on both**:
    the Reviewer was still finding new (non-duplicate) gaps in round 2, the
    last round allowed by the default `--max-review-rounds 2` - neither
    spec's round 2 produced a converging "looks complete" message (resort-
    signup: "critical gaps ... related to duplicate checks and consent
    validation logic ... unlinked validations for various fields"; todo-
    list: "missing negative cases for character limits, performance testing
    for large task lists, verification of UI elements"). So all three
    specs' suites are usable but likely not exhaustive - this isn't a
    resort-signup-specific quirk, it's a general property of the default
    round cap. ~~Follow-up: rerun with a higher `--max-review-rounds` (3 or
    4) on at least one spec to see whether the Reviewer actually converges
    toward "approved" given more rounds, or keeps finding new gaps
    indefinitely...~~ Done 2026-09-28 - see item 14 below: it does NOT
    converge, at least not by round 4.
13. ~~Fix #28 (item 9) verified live on OpenAI (2026-09-28) - mixed result,
    NOT fully closed:~~ Done 2026-09-28, same session - see fix #29 above
    (both the widened regex and the "successfully"-override fix landed
    together). Original finding kept below for the record.
    - resort-signup run10 (fresh Gatherer run, different phrasing than
      run9): clean by inspection - no error-flavored statement mistagged
      POSITIVE, no success-flavored statement mistagged NEGATIVE. B9/B10's
      old content bugs (item 8) also look resolved, as a bonus. The exact
      "proceeds without errors"/"can proceed to submit" phrasings weren't
      reproduced this run (the model worded things differently), so the new
      regexes weren't actually exercised here - inconclusive on those two
      signals specifically, though nothing regressed.
    - login-portal run4 (fresh Gatherer run, different phrasing than run3):
      B8 correctly flipped NEGATIVE->POSITIVE (credentials-match ->
      saved-to-sessionStorage, a genuine success). But B9/B10 - this run's
      version of run3's B12/B13 access-denial case - were ALSO flipped
      NEGATIVE->POSITIVE, which is wrong; the model had them right
      originally. New statement wording: "After successfully logging in as
      an Admin, if they navigate to admin.html without a matching role in
      sessionStorage, they should be redirected back to index.html." Two
      distinct gaps caused this, found by inspection (not yet in
      pipeline.py):
      (a) the new `_ACCESS_DENIAL_CONDITION_RE` doesn't match "without a
          matching role" - only "no valid"/"not valid"/"invalid"/"no
          match"/"does not match"/"wrong role"/"not logged in|
          authenticated" are covered, and this phrasing is none of those.
          Straightforward: widen the regex once there's a moment.
      (b) A separate, likely more important gap: the word "successfully" in
          "After successfully logging in as an Admin" (describing the PRIOR
          login, a true precondition) is enough on its own to satisfy
          `_has_success_signal` via the plain `"successfully"` keyword in
          `_SUCCESS_KEYWORDS`, before the redirect/denial logic is even
          reached - regardless of what the actual tested OUTCOME is. This is
          a pre-existing gap, not something fix #28 introduced, but fix #28
          made it visible on a real case. Fixing (a) alone would NOT fix
          this case, because `_has_success_signal` returns True from
          "successfully" before ever checking the access-denial pattern -
          `_has_success_signal`'s `other_success_keywords` check runs first
          and returns immediately. Needs its own fix, likely either scoping
          `"successfully"` to require it describe the sentence's main/final
          clause rather than firing anywhere in the text, or having a
          detected access-denial condition suppress the plain "successfully"
          success keyword the way `_REDIRECT_TO_LOGIN_RE` already suppresses
          the plain "redirect" keyword.
    Net: fix #28 didn't regress anything and fixed real cases (resort-signup
    by inspection, login-portal's B8), but item 9 stays open rather than
    closed - re-flagged rather than marked done a second time.
14. Item 12's follow-up: resort-signup rerun with `--max-review-rounds 4`
    (2026-09-28, same cached `run9.json` behaviors as the 2-round run for a
    like-for-like comparison) - **the Reviewer does NOT converge toward
    "approved" by round 4**:
    | Round | Gaps found | Cases added |
    |---|---|---|
    | 1 | 9 | 8 |
    | 2 | 6 | 11 |
    | 3 | 8 | 1 |
    | 4 | 6 | 14 |

    Final: 196 test cases (up from 174 at the 2-round default). Round 4 -
    the last round allowed - still found 6 new, non-duplicate gaps and
    added MORE cases than round 3 did (14 vs 1), not fewer - no sign of
    tapering off, let alone converging. The gap descriptions stayed
    substantive throughout (leading/trailing-whitespace validation,
    consent-checkbox edge cases, field-to-behavior traceability gaps in the
    round 4 summary), not degenerating into trivial or repetitive findings
    that would suggest the Reviewer was just padding for the sake of it.
    Conclusion: this answers item 12's open question - simply raising
    `--max-review-rounds` further is unlikely to reach a natural stopping
    point on its own; the round cap alone isn't a real answer for a spec
    this size. ~~If exhaustiveness matters more than cost/time for a given
    run, a higher cap is a reasonable lever, but the real fix (not yet
    designed) is more likely one of: (a) a Reviewer prompt change...(b) a
    different stopping condition entirely...~~ Done 2026-09-28, in two
    steps: fix #30 first added a diagnostic duplicate-rate metric to the
    per-round log line (rather than guessing at an auto-stop rule blind -
    the round 3 vs. round 4 numbers above show a naive single-round trigger
    would have been actively wrong, cutting off before the run's most
    productive round), then Kenneth chose direction (b) and fix #31 built
    the actual auto-stop heuristic on top of it: two consecutive
    high-duplicate-rate rounds required before stopping, specifically so a
    single misleading round like round 3 doesn't end a run early. Not yet
    verified against a real live run - see fix #31's closing note.

(Items 2-3 from an earlier version of this list - expanding the
`case_type`/`broad_category`/permutation taxonomy, and this list's own
former item 4, the header rows - are covered by fixes #12/#13/#18 above;
the taxonomy expansion and the header rows are both done. Rule 5 cross-mode
coverage, formerly this list's item 1, is now covered by fix #15. Two more
placeholder shapes found in the same run as fix #15's first live test are
covered by fix #17.)
