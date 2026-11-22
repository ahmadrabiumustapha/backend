# Fuzzing Guide — Zenith Backend

This directory contains [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz)
(libFuzzer) targets for the attack surface described in
[issue #117](https://github.com/owner/repo/issues/117).

---

## Prerequisites

cargo-fuzz requires **Rust nightly** and **cargo-fuzz** to be installed:

```bash
rustup install nightly
cargo install cargo-fuzz --locked
```

You must run all `cargo fuzz` commands from the **workspace root**
(`/path/to/zenith-backend/`), not from inside `fuzz/`.

---

## Fuzz targets

| Target | What it fuzzes | Key properties asserted |
|---|---|---|
| `strkey_decode` | `decode_stellar_public_key` and `encode_stellar_public_key` | Never panics; encode→decode round-trip is identity; result always starts with `G` |
| `auth_nonce_message` | Wallet-address validation path in the nonce endpoint | Never panics on any UTF-8 input; valid addresses round-trip |
| `auth_verify_request` | Signature verification: base64 decode, ed25519 key construction, `verify_strict` | Never panics on any byte sequence |
| `black_scholes` | `black_scholes()` pricing over the full f32-range input space | Finite output; premium ≥ 0; time_value ≥ 0; delta in [0,1]/[-1,0]; gamma/vega ≥ 0; put-call parity within 1e-4 × spot |
| `implied_vol` | `implied_vol()` Newton-Raphson solver | Never panics; `Some(iv)` is finite and positive; round-trip via BS within 1e-3 |
| `payoff_series` | `combined_pnl`, `combined_payoff_series`, `net_premium` | Series length = steps+1; every point matches direct `combined_pnl`; long+short twin nets to ~0 |
| `json_open_position` | JSON deserialization of `OpenPositionRequest` and `RollPositionRequest` | Never panics; all f64 fields are finite when deserialization succeeds |
| `json_payoff_request` | JSON deserialization of `PayoffRequest` (variable-length `legs` array) | Never panics; runs math on valid inputs |
| `json_alert_request` | JSON deserialization of `CreateAlertRequest`, `AddWatchlistRequest`, `NonceRequest`, `VerifyRequest` | Never panics |
| `cursor_query` | URL-encoded `limit`/`offset` query-string decoding for list endpoints | Never panics; clamped limit always ∈ [1, 200]; offset always ≥ 0 |

---

## Running a single target locally

```bash
# Run with the committed seed corpus (fast smoke-test):
cargo fuzz run strkey_decode fuzz/corpus/strkey_decode -- -timeout=30 -max_total_time=60

# Run with no time limit until Ctrl-C or a crash:
cargo fuzz run black_scholes fuzz/corpus/black_scholes -- -timeout=30

# Run with a larger max input size (default is 4096 bytes):
cargo fuzz run json_payoff_request -- -timeout=30 -max_len=65536
```

## Running all targets

```bash
for target in strkey_decode auth_nonce_message auth_verify_request \
              black_scholes implied_vol payoff_series \
              json_open_position json_payoff_request json_alert_request \
              cursor_query; do
  echo "=== $target ==="
  cargo fuzz run "$target" "fuzz/corpus/$target" \
    -- -timeout=30 -max_total_time=300 -max_len=4096
done
```

---

## Investigating a crash

When libFuzzer finds a crash it writes the minimised input to
`fuzz/artifacts/<target>/crash-<sha1>`.

```bash
# Reproduce the crash:
cargo fuzz run strkey_decode fuzz/artifacts/strkey_decode/crash-<sha1>

# Minimise the input further:
cargo fuzz tmin strkey_decode fuzz/artifacts/strkey_decode/crash-<sha1>

# Get a human-readable hex dump of the crashing input:
xxd fuzz/artifacts/strkey_decode/crash-<sha1>
```

Add a regression test in `src/<module>.rs` or `tests/<module>_test.rs`
that directly exercises the crashing input, so the crash stays fixed.

---

## CI integration

The workflow at `.github/workflows/fuzz.yml` runs two strategies:

| Trigger | Budget | Description |
|---|---|---|
| `pull_request` / `push` | 5 min per target | Smoke-test: catch obvious regressions introduced by a PR |
| `schedule` (nightly, 02:00 UTC) / `workflow_dispatch` | 10 min per target | Deeper batch run: grows the corpus over time |

Each run:
1. Restores the corpus from the previous nightly run (if available).
2. Fuzzes with `-timeout=30` (hang protection) and `-max_total_time=<budget>`.
3. Uploads the corpus snapshot as a GitHub Actions artefact.
4. On failure, uploads the `fuzz/artifacts/<target>/` directory.
5. On nightly failures, minimises each crash before uploading.

---

## Adding a new target

Follow these four steps. Use `cursor_query` as a reference — its files are
`fuzz/fuzz_targets/cursor_query.rs`, the `[[bin]]` block in `fuzz/Cargo.toml`, and
the seeds under `fuzz/corpus/cursor_query/`.

### 1. Create the target file

Create `fuzz/fuzz_targets/<name>.rs`. The file must include the `#![no_main]`
attribute and invoke the `fuzz_target!` macro from `libfuzzer_sys`:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // Drive the code under test with `data`.
    // Panic on any invariant violation — libFuzzer treats a panic as a finding.
});
```

If your target needs structured input, derive `Arbitrary` on a local type and use
`fuzz_target!(|input: MyInput|` instead.

### 2. Register the target in `fuzz/Cargo.toml`

Add a `[[bin]]` entry with `test = false` and `doc = false` (required by cargo-fuzz):

```toml
[[bin]]
name = "<name>"
path = "fuzz_targets/<name>.rs"
test = false
doc = false
```

Add any new dependencies the target needs to `[dependencies]` in the same file.

### 3. Seed the corpus

Create `fuzz/corpus/<name>/` and add at least a few representative seed inputs as
plain files (no extension). Each file is one raw input byte sequence. Meaningful seeds
give the fuzzer a head-start and serve as a smoke-test corpus:

```
fuzz/corpus/<name>/empty          # zero bytes or a minimal valid input
fuzz/corpus/<name>/typical        # a normal, expected input
fuzz/corpus/<name>/edge_case      # a boundary or error-triggering input
```

Seed files are plain text when the target accepts strings (e.g. URL-encoded data) and
raw bytes otherwise. See `fuzz/corpus/cursor_query/` for an example of text seeds.

### 4. Register in CI

Add `<name>` to **both** `matrix.target` lists in `.github/workflows/fuzz.yml` — one
list under the `fuzzing-pr` job (5-minute budget) and one under `fuzzing-nightly`
(10-minute budget). Both lists must stay in sync; a target absent from either list
will not be run in that workflow.

---

## Findings during development

| # | Target | Finding | Status |
|---|---|---|---|
| — | all | No panics, property violations, or hangs found in the seed corpus runs | N/A |

*(Update this table as crashes are discovered and fixed.)*
