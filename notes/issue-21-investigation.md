# Issue #21 Investigation: `last-call-deadline`

## Goal
Add a lint that reports a valid `last-call-deadline` only when its date has already passed.

## Existing responsibility

### `preamble-req-last-call-deadline`
- Implemented through `RequiredIfEq`.
- Checks the relationship between:
  - `status`
  - `Last Call`
  - `last-call-deadline`
- Reports when the deadline is missing while `status: Last Call`.
- Also reports when the deadline exists but `status` is missing or is not `Last Call`.

### `preamble-date-last-call-deadline`
- Implemented through generic `Date<S>`.
- Looks up the configured field by name.
- Returns successfully when the field is absent.
- Trims the field value.
- Validates exact `YYYY-MM-DD` component lengths.
- Parses with `chrono::NaiveDate::parse_from_str(value, "%Y-%m-%d")`.
- Reports one source-anchored diagnostic when the date is malformed.
- Returns successfully when the date is valid.

## New responsibility

### Proposed `preamble-future-date`
- Looks up a configured date field, initially `last-call-deadline`.
- Returns successfully when the field is absent.
- Returns successfully when the date cannot parse.
  - Reason: `preamble-date-last-call-deadline` owns malformed-date errors.
- Computes the accepted civil-date threshold.
- Reports an error only when a valid parsed date is older than that threshold.
- Must not inspect `status`.
  - Reason: `RequiredIfEq` already owns the `Last Call` policy.

## Architecture observations
- Preamble lints implement `Lint`.
- The primary entry point is:
  `fn lint<'a>(&self, slug: &'a str, ctx: &Context<'a, '_>) -> Result<(), Error>`
- Fields are read with:
  `ctx.preamble().by_name(self.0.as_ref())`
- Diagnostics are emitted through:
  `ctx.report(...)`
- Diagnostics are anchored to the original YAML source using `Snippet`.
- Generic lint configuration uses a wrapper such as:
  `pub struct Date<S>(pub S);`

## Open questions
- Where are preamble lints exported and registered as default lints?
- What is the project’s preferred way to obtain the current date?
- Should the comparison use a UTC-12 civil-date threshold?
- How should time-dependent behavior be tested without flaky tests?
- Which existing fixtures use now-expired Last Call deadlines and need updates?

## Next investigation step
Read lint registration, configuration, and date-lint tests.

## Investigation update — 2026-10-05
- Confirmed `Date<S>` returns early for a missing field and owns malformed-date diagnostics.
- Confirmed `RequiredIfEq<S>` owns both requiredness and “only allowed when” policy.
- New future-date lint should be generic and should not inspect `status`.

## Investigation update — registration and configuration

### Export path
- Preamble lint modules are declared and re-exported from:
  `eipw-lint/src/lints/preamble.rs`
- Existing Date pattern:
  `pub mod date;` and `pub use self::date::Date;`
- Proposed FutureDate pattern:
  `pub mod future_date;` and `pub use self::future_date::FutureDate;`

### Default lint representation
- Built-in lint enum location:
  `eipw-lint/src/lints/known_lints.rs`
- Existing Date enum variant:
  `PreambleDate { name: preamble::Date<S> }`
- FutureDate will need:
  - `PreambleFutureDate { name: preamble::FutureDate<S> }`
  - an `as_inner` match arm returning `name`
  - a `map_to_str` match arm reconstructing `preamble::FutureDate(name.0.as_ref())`

### Default enablement
- Default lint registration location:
  `eipw-lint/src/config.rs`, function `default_lints()`
- Existing deadline date registration:
  slug `preamble-date-last-call-deadline` using
  `PreambleDate { name: preamble::Date("last-call-deadline") }`
- Proposed registration:
  slug `preamble-future-date` using
  `PreambleFutureDate { name: preamble::FutureDate("last-call-deadline") }`
- Default registration should sit immediately after the existing deadline-date lint.

### Execution model
- `eipw-lint/src/lib.rs` builds the Linter from `Options::default()`.
- Every default lint receives the parsed preamble through `Context`.
- `config.rs`, not `lib.rs`, determines which default lints run.

### Environment note
- `rg` is unavailable in this Codespace.
- Use `grep -R` instead of `rg` until ripgrep is installed.

### New risk to investigate
- Adding a default future-date lint may cause legacy test fixtures with dates such as
  `2020-01-01` or `2021-01-01` to emit additional diagnostics.

## Investigation update — test architecture and dependency

### Test architecture
- Integration fixture runner:
  `eipw-lint/tests/eipv.rs`
- It iterates over directories under:
  `eipw-lint/tests/eipv/`
- Each fixture provides:
  - `input.md` as linter input
  - `valid.txt` when no diagnostic is expected, or
  - `expected.txt` when diagnostics are expected
- The fixture runner uses the default linter configuration, so adding a default lint
  affects every applicable fixture.

### Existing deadline fixture impact
- Multiple integration fixtures contain:
  `last-call-deadline: 2020-01-01`
- The valid Last Call fixture contains:
  `last-call-deadline: 2021-01-01`
- These dates will become expired under the new default lint.
- Likely approach:
  replace valid/unrelated-fixture deadlines with a stable far-future date such as
  `3000-12-12`, so unrelated fixtures continue testing only their intended rule.
- Keep malformed deadline fixture values unchanged because Date<S> owns malformed-date errors.

### Chrono dependency
- Current dependency:
  `chrono = { version = "0.4.40", default-features = false }`
- Existing Date<S> uses `NaiveDate` parsing and does not require the current clock.
- The future-date lint needs the current date, so investigate enabling Chrono's
  `clock` feature before implementation.

## Investigation update — exact test conventions

### Fixture behavior
- Fixture harness:
  `eipw-lint/tests/eipv.rs`
- It runs:
  `Linter::<Text<String>>::default()`
- It allows only `preamble-file-name` before running.
- `expected.txt` means exact diagnostic output is expected.
- `valid.txt` means an empty diagnostic output is expected.
- This confirms new default-lint behavior must be covered by an integration fixture.

### Direct unit-test pattern
- Existing example:
  `eipw-lint/tests/lint_preamble_date.rs`
- Pattern:
  - create an in-memory Markdown preamble string
  - start with `Linter::<Text<String>>::default()`
  - call `.clear_lints()`
  - enable one lint with `.deny(slug, lint)`
  - call `.check_slice(None, src)`
  - call `.run().await`
  - assert exact rendered diagnostic text
- FutureDate should have direct tests for:
  - missing field returns no diagnostic
  - malformed date returns no diagnostic
  - clearly expired valid date returns one diagnostic
  - far-future valid date returns no diagnostic

### Integration-test pattern
- Add a default-linter fixture for an expired, valid Last Call deadline.
- The fixture should include:
  - `input.md` with a valid preamble and a clearly past deadline
  - `expected.txt` containing the expected `preamble-future-date` diagnostic
- Update valid and unrelated existing fixtures with expired but syntactically valid
  deadlines to a stable far-future date, likely `3000-12-12`.
- Do not alter the malformed deadline fixture because it must continue testing
  the existing date-format lint independently.

### Diagnostic stability
- Do not include the dynamic current date in the diagnostic title or label.
- Prefer a stable title such as:
  `preamble header \`header\` must be today or a future date`
- Prefer a stable span label such as:
  `date is in the past`

## Investigation update — workspace and lockfile

### Workspace
- Root workspace members:
  `eipw-preamble`, `eipw-lint`, `eipw-lint-js`, and `eipw-snippets`
- Workspace edition:
  Rust 2021
- Minimum Rust version:
  1.81
- Project license:
  MPL-2.0

### Dependency and lockfile
- `Cargo.lock` is tracked at the repository root.
- There is no crate-local `eipw-lint/Cargo.lock`.
- Current resolved Chrono package:
  version `0.4.40`
- Current Chrono lockfile dependency list contains only:
  `num-traits`
- `eipw-lint/Cargo.toml` disables Chrono default features.
- The future-date lint needs a current date; likely change:
  enable Chrono feature `clock`.
- Let Cargo regenerate `Cargo.lock`; never hand-edit the lockfile.

### Implementation readiness
- Source integration points identified.
- Default registration point identified.
- Unit and integration test patterns identified.
- Legacy fixture impact identified.
- Ready to implement after agreeing on the date threshold and tests.

## Design decision — calendar boundary and minimal scope

### Chosen boundary
- Use a UTC-12 civil-date threshold.
- Compute:
  `let threshold = (Utc::now() - Duration::hours(12)).date_naive();`
- Report only when:
  `deadline < threshold`
- This keeps a deadline valid while its stated calendar date is still in
  progress anywhere on Earth.

### Why not a plain UTC boundary?
- Plain UTC would reject a deadline at midnight UTC.
- That can reject a date while it remains the same calendar day for authors
  west of UTC.
- The prior PR review specifically raised this concern.

### Scope gate
- Reuse existing `Date<S>`, `RequiredIfEq<S>`, `Lint`, `Context`, and
  diagnostic patterns.
- Use Chrono already present in the project; enable only its `clock` feature.
- Do not add a clock abstraction, configuration setting, external dependency,
  GitHub/PR-history integration, or unrelated refactor.
- Make the smallest focused change that closes issue #21.

## Process decision — gate-based minimal implementation

This contribution follows a gate-based approach:

1. Need gate passed:
   issue #21 is open and the future-date lint is absent.
2. Ownership gate passed:
   existing RequiredIfEq owns Last Call policy and Date owns date syntax.
3. Scope gate passed:
   one generic FutureDate lint can own only the expiration comparison.
4. Dependency gate:
   reuse Chrono already in the project; enable only the clock feature if needed.
5. Verification gate:
   require focused unit tests and a default-linter integration fixture.
6. Diff gate:
   no clock framework, configuration setting, GitHub/PR-history logic,
   external crate, or unrelated refactor.

Implementation principle:
- Add the smallest composable lint that closes issue #21.
- Return early for absent and malformed dates.
- Report only a valid date that has elapsed.

## Design decision — EIP-1 and prior review evidence

### EIP-1 wording
- `last-call-deadline` is defined as:
  “The date last call period ends on.”
- It is optional generally and needed only when status is `Last Call`.
- It must use ISO 8601 `yyyy-mm-dd`.
- EIP-1 says a Last Call review end date is typically 14 days later.
- EIP-1 does not define a time zone or a midnight expiration instant.

### Prior maintainer review on PR #120
- Keep this lint small and single-purpose.
- Check date freshness only if the configured field is present.
- Do not inspect status; existing required/date lints own that work.
- Avoid a plain UTC-date boundary because it can reject a deadline too early.
- Reviewer suggested a UTC-12-style boundary.
- Use stable far-future dates such as `3000-12-12` in tests.
- Use the MPL source header.

### Chosen interpretation
- Use the UTC-12 civil-date threshold:
  `(Utc::now() - Duration::hours(12)).date_naive()`
- An expired deadline satisfies:
  `deadline < threshold`
- This treats `last-call-deadline` as valid through its stated day everywhere,
  consistent with EIP-1’s date-only wording and prior maintainer feedback.

## Gate approval — implementation begins

- Design gate approved.
- Implement a small generic `FutureDate<S>` lint.
- It validates only a present, parseable date field.
- It uses a UTC-12 civil-date threshold.
- It does not inspect `status`.
- First implementation slice:
  source module, module export, Chrono clock feature, and isolated tests.
- Default-lint registration and fixture migration occur only after the local test gate passes.

## Gate approval — implementation begins

- Design gate approved.
- Implement a small generic `FutureDate<S>` lint.
- It validates only a present, parseable date field.
- It uses a UTC-12 civil-date threshold.
- It does not inspect `status`.
- First implementation slice:
  source module, module export, Chrono clock feature, and isolated tests.
- Default-lint registration and fixture migration occur only after the local test gate passes.

## Environment finding — dictionary Git submodule

- Cargo and the Rust toolchain are installed and running in the Codespace.
- Baseline test compilation initially failed before executing tests because
  `eipw-lint/src/lints/markdown/spell.rs` embeds dictionary files using
  `include_str!`.
- Required files were absent:
  `eipw-lint/dictionaries/dictionaries/en/index.aff`
  `eipw-lint/dictionaries/dictionaries/en/index.dic`
- The repository declares the dictionaries as a Git submodule.
- Local setup action:
  `git submodule update --init --recursive`
- This is an environment/setup prerequisite, not part of the future-date-lint
  contribution. Do not add placeholder dictionary files or modify spell logic.

## Gate result — baseline environment verified

- Initialized the required dictionary Git submodule successfully.
- Dictionary revision:
  `8cfea406b505e4d7df52d5a19bce525df98c54ab`
- Verified embedded English dictionary files exist.
- Rust toolchain verified:
  - cargo 1.81.0
  - rustc 1.81.0
- Baseline command passed:
  `cargo test -p eipw-lint --test lint_preamble_date`
- Result:
  5 passed; 0 failed.
- Environment gate is complete. Any future test failure can now be investigated
  as a feature-change issue rather than a missing-environment prerequisite.

## Environment note — Rust PATH in new terminals

- Rust was installed with Rustup under `~/.cargo/bin`.
- New Codespaces terminal sessions may not automatically load Cargo.
- Fix for a current shell:
  `source "$HOME/.cargo/env"`
- Persistent Bash setup:
  add `source "$HOME/.cargo/env"` to `~/.bashrc`.
- This is local environment setup only and is not part of the contribution diff.

## Gate result — local implementation verification

- Initial focused test run:
  3 passed and 1 failed.
- The failure was not logic-related.
- The past-date diagnostic rendered a span covering the full 10-character
  value `2000-01-01`; the expected snapshot had an incorrect caret span.
- Corrected the snapshot to match the actual source-anchored diagnostic.
- `cargo fmt` also reordered module declarations and re-exports in
  `preamble.rs` according to the repository formatter.
- No implementation scope expanded; this was snapshot and formatting cleanup.

## Gate result — local lint implementation passed

- Ran `cargo fmt`; formatting is now clean.
- Ran `cargo fmt --check`; passed.
- Ran:
  `cargo test -p eipw-lint --test lint_preamble_future_date`
- Result:
  4 passed; 0 failed.
- Ran `git diff --check`; passed with no whitespace errors.
- FutureDate behavior verified in isolation:
  - missing configured field: no diagnostic
  - malformed configured field: no diagnostic
  - clearly expired valid date: one diagnostic
  - far-future valid date: no diagnostic
- Local implementation gate passed.

## Gate result — existing integration fixtures preserved

- Ran:
  `cargo test -p eipw-lint --features tokio --test eipv`
- Result:
  1 passed; 0 failed.
- Updated 15 historical valid deadline values in fixture input files to
  `3000-12-12`.
- Existing malformed deadline fixture remained unchanged.
- Existing fixture snapshots did not need modification.
- This confirms legacy fixtures continue testing their original concerns under
  the new default future-date rule.

## Next test requirement
- Add a dedicated `preamble-future-date` eipv fixture using a valid but
  clearly expired deadline.
- Its expected output must contain exactly one `preamble-future-date`
  diagnostic.
