# MetricConsensus

**A GenLayer Intelligent Contract that verifies and aggregates a number from several web sources: an LLM only *extracts* one value per source; everything after that (parsing, grounding, unit conversion, freshness, duplicate detection, outliers, median/mean, final status) is deterministic integer code that every validator recomputes.**

> **Status.** Deployed and partly exercised on GenLayer Studio Next (GenVM v0.3) on 12 Nov 2026: metric registration and one full verification round reached consensus with three validator models. See [Tested on Studio Next](#tested-on-studio-next) for exactly what was and was not run live. The 102 local tests run against a *simulation* of the SDK with scripted pages and answers and do not prove real validator consensus or real model behaviour.

## What it does

An owner registers a **metric** (what is measured, expected unit, data type, extraction instructions, minimum number of sources, maximum source age, maximum deviation, aggregation method, outlier policy) and 2 to 5 source URLs. `verify_metric` runs a verification round:

1. Leader and every validator fetch all sources.
2. The model extracts, per source: the number as written, its unit, whether it is the `CURRENT` value (or `HISTORICAL` / `AMBIGUOUS` / `NOT_FOUND`), the measurement timestamp *only if the page states it*, and a verbatim excerpt.
3. The contract does the rest, deterministically: **grounding** (the number must literally appear in the page), unit normalisation, freshness, duplicate detection, median-based outlier detection, aggregation, status.
4. Validators re-extract independently, recompute everything, and must agree within explicit tolerances. Only then is an observation stored.

| Status | Meaning |
|---|---|
| `CONSENSUS_REACHED` | enough independent, fresh, valid values agree within the tolerance; the aggregate is stored |
| `DISAGREEMENT` | valid values exist but there is no clear core majority |
| `OUTLIERS_DETECTED` | a core majority agrees but some values deviate (policy `FLAG`: no aggregate is produced) |
| `INSUFFICIENT_SOURCES` | too few valid independent observations |
| `SOURCE_UNAVAILABLE` | fewer than the minimum number of sources could be read |
| `STALE_DATA` | too few observations are fresh enough |
| `INCONCLUSIVE` | ambiguous values, or timestamps that are missing / unparseable / in the future while recency is required |

The contract never invents a value, never converts currencies, never lets the model aggregate, and never treats a missing source as disagreement.

## Why this design

- **Numbers are fixed-point integers (x 10^6).** No floating point anywhere (a test parses the source and rejects any float literal, `float()` call or true division), so leader and validators cannot drift by rounding. Rounding is half away from zero and defined once.
- **The model cannot make up a number.** An extracted value is rejected unless it appears verbatim in the page (`VALUE_NOT_IN_SOURCE`), so "current vs previous price", review counts and ids are not silently confused with hallucinated values. A number that *is* on the page but wrong (or injected into it) is caught by the cross-source deviation check.
- **Explicit reasons.** Every excluded observation carries one: `NOT_ACCESSIBLE`, `HISTORICAL_VALUE`, `AMBIGUOUS_VALUE`, `MISSING_VALUE`, `VALUE_INVALID`, `VALUE_NOT_IN_SOURCE`, `UNIT_INVALID`, `UNIT_INCOMPATIBLE`, `STALE`, `TIMESTAMP_UNKNOWN`, `DUPLICATE_OF_n`, `OUTLIER`.
- **Copies are one observation.** Pages with near-identical text are detected in code and collapse into a single independent observation.

## Units

Reliable, exact conversions only: temperature (C, F, K), length (mm, cm, m, km, in, ft, mi), mass (mg, g, kg, lb, oz), time (ms, s, min, h), data size (B, KB, MB, GB, TB, decimal), percent, count. Currencies (USD, EUR, GBP, JPY, CHF, CAD, AUD, CNY, UAH, PLN, BTC, ETH, USDT, USDC) are **never** converted into each other; a mismatch is `UNIT_INCOMPATIBLE`. The bare symbols `$` and the yen sign are rejected as ambiguous. See [docs/AGGREGATION.md](docs/AGGREGATION.md).

## Consensus (Equivalence Principle)

`verify_metric` uses the custom leader/validator pattern through `gl.vm.run_nondet_default` (falls back to `run_nondet_unsafe` on GenVM v0.2).

- **Leader:** fetches, extracts with the LLM, then runs the deterministic pipeline.
- **Validators:** do not trust the leader. They **recompute** every normalised value, validity, freshness, exclusion, outlier flag, the aggregate and the status from the leader's own raw extractions; then **re-fetch and re-extract** every source themselves and compare: status, aggregate (within 0.5 %), and per source retrieval status, validity, freshness, included/outlier flags, exclusion reason and normalised value (within 0.5 %), plus the deterministic duplicate pairs. Wording, excerpts, relationship labels (other than duplicates) and confidence are not compared.
- State is written only after consensus; a failed or undetermined round changes nothing. If a source changes between the leader's and a validator's fetch beyond the tolerance, validators disagree and the round fails safely.

Details: [docs/CONSENSUS.md](docs/CONSENSUS.md).

## Public API

| Method | Kind | Notes |
|---|---|---|
| `register_metric(metric_name, description, category, data_type, expected_unit, extraction_instructions, source_urls, min_sources, max_sources, max_age_hours, max_deviation_bps, aggregation_method, outlier_policy, external_ref)` | write | `source_urls`: separated by `\|\|`, spaces or newlines (may be empty). `max_age_hours = 0`: no recency requirement. `max_deviation_bps`: 100 = 1 %. `aggregation_method`: `MEDIAN` (default), `MEAN`, `MIN`, `MAX`. `outlier_policy`: `FLAG` (default) or `EXCLUDE`. |
| `add_metric_source(metric_id, source_url, label)` | write | Owner only; append-only; duplicates refused. |
| `verify_metric(metric_id)` | write | Owner only; every call is a new round. Use a generous fee. |
| `get_metric`, `get_metric_sources`, `get_latest_observation`, `get_observation(id, n)`, `get_observation_history`, `get_metric_count`, `get_observation_count` | view | History is newest first. |
| `get_integration_status(metric_id)` | view | Stable read interface for other contracts and apps ([docs/INTEGRATION.md](docs/INTEGRATION.md)). |

The metric definition is immutable (no update method). Observations are append-only (at most 50 per metric) and each stores the evidence set it used. A round that ends in an agreed unsuccessful status is recorded (`accepted: false`) but never becomes the "last accepted observation"; a round without consensus leaves no trace on chain.

## Deploy

1. Open [Studio Next](https://studio-next.genlayer.com/), paste `contracts/metric_consensus.py` (keep the two header lines), deploy with no arguments.
2. Register a metric, verify, read the observation. Examples: [docs/USAGE.md](docs/USAGE.md).

The `Depends` runner hash is tied to the GenVM bundle (this file uses the one reported for Studio Next bundles v0.6.0-rc9/rc10). If schema loading fails with `runner absent` / `runner malformed`, copy the first two lines of a built-in example contract in your Studio build.

## Tests

`python -m unittest discover -s tests -v` (102 tests, no dependencies): numerics (parsing, rounding, conversions, median even/odd, tolerance), per-source rejection reasons, aggregation and outlier rules, freshness, duplicates, registration and access control, scripted end-to-end rounds, simulated consensus (agreement within tolerance, disagreement, source changes between retrievals, validator errors), prompt-injection scenarios, history, and static checks of the source. **Simulation only**; mutation checks (blind validator, early state write, outliers silently dropped, silent currency conversion, grounding disabled, stale data accepted) make the suite fail.

## Tested on Studio Next

Live, 12 Nov 2026, GenVM v0.3.0-rc10, 3 agreeing validator models per run, using three public gists as sources (one page listing a current price next to a previous price, a review count and a shipping cost; one with a single price; one with a price offered by several vendors). Metric: USD, `MEDIAN`, minimum 2 sources, maximum age 72 hours, deviation limit 5 %.

| Scenario | Result | Transaction |
|---|---|---|
| `register_metric` with three sources | stored (state read back) | `0xe0a9ff36419c5dbfc6a1a98ecbe7b9ad6557a66f1427f83287a2f519458446bd` |
| `verify_metric` | `CONSENSUS_REACHED`, aggregate `100` (median of 100, 101, 99.5); the current price was extracted rather than the previous price, review count or shipping cost; every value found verbatim in its page; excerpts verified; freshness computed from the page date (18 h); 3 independent observations | `0xf158b89952e67ea2078c2deea0de4fcfd71e8269a4f24711a3ebfdf1b6c0bfa4` |

**Not run on a live network** (simulation tests only): rejection of invalid registrations and of non-owners, `OUTLIERS_DETECTED` / `DISAGREEMENT` / `EXCLUDE` policy, stale or undated sources, unit and currency handling, `SOURCE_UNAVAILABLE`, duplicate pages, a page with an injected instruction, repeated rounds and history preservation, the 50-observation cap, and how often validators fail to agree on live pages that change between fetches.

Observed on the live run: the model labelled all three sources `POSSIBLY_RELATED` (they share a host); such labels are informational and are not compared by validators.

## Known limitations

- **Only part of the scenarios were run live** (see the table above). Real validators may disagree on live pages more often than the simulation suggests; then nothing is stored and the round can be retried.
- Plain HTTP GET; each page is cleaned and truncated to **6,000 characters**. Prices rendered by JavaScript, behind logins or past the truncation point are not seen (the source then yields `MISSING_VALUE` or looks empty).
- **Several websites showing a number do not make it true.** The contract reports agreement of sources, not objective truth, and cannot establish where the data came from. Independence is only reported conservatively (duplicates in code, same registrable domain, the model's labels).
- A number that appears on a page but is injected or wrong there is only caught if other sources disagree; it passes if all sources show it.
- The extraction depends on the model reading "current vs previous" correctly; grounding only proves the number is present on the page.
- Only the metric owner can verify (prevents history flooding). Pages that change between the leader's and validators' fetches beyond 0.5 % fail the round.
- Currencies are never converted; a metric in USD cannot use a source in EUR.
- Cross-contract calls to the other contracts were not implemented or verified; see INTEGRATION.md.

MIT, see [LICENSE](LICENSE).
