# SynthID-text removal benchmark

bench_synthid_text.py measures how well the Layer B rewrite
(rewrite_text.py) removes SynthID-text-class watermarks, and at what
cost. It generates a controlled corpus with the MarkLLM SynthID scheme, runs
removal variants, and emits a shareable report.

## What it measures

| Metric | Meaning |
| --- | --- |
| Clear rate | % of watermarked samples that flip to not-watermarked after removal (MarkLLM same-config detection) |
| Score suppression | mean/median drop in detector score (before - after) |
| Quality | lexical divergence (bigram Jaccard distance), length drift, number/URL survival |
| Cost | estimated tokens in/out, wall time per document, optional USD at your prices |
| Efficiency | clears per million output tokens - removal rate per unit of rewrite cost |
| Attempts | mean rewrite attempts per document (the Layer B loop stops early on pass) |
| Controls | no-removal baseline, Layer A only (expect ~0% - Unicode scrub must not clear a statistical mark), sanity-gate exclusions, optional re-stamp check |

## How to run

Prerequisites (all external, matching the repo's optional-harness model):

1. A MarkLLM checkout: run service/scripts/setup_markllm.sh (clones
   THU-BPM/MarkLLM at a pinned commit and creates ~/MarkLLM/.venv).
2. A rewrite backend: Ollama (default, loopback) or any
   OpenAI-compatible endpoint. `--rewrite-model` is required and names the
   LLM that performs the rewrite - it is a different model from
   `--markllm-model` (default `facebook/opt-1.3b`), which only generates and
   detects the watermark and never rewrites. Prefer a **non-origin** model:
   rewriting with the same watermarked model that produced the text can
   re-stamp it (`--restamp-control` measures this).

    # minimal: 3 docs, 1 seed, paraphrase with up to 3 attempts (default, Ollama)
    python3 service/scripts/bench_synthid_text.py \
      --markllm-dir ~/MarkLLM \
      --rewrite-backend ollama --rewrite-model llama3.2 \
      --out-dir out/bench-2026-06-01

    # recommended full run: more docs/seeds, backtranslate variant, re-stamp control
    python3 service/scripts/bench_synthid_text.py \
      --markllm-dir ~/MarkLLM \
      --docs 10 --seeds 3 \
      --variants "paraphrase:3,backtranslate:3" \
      --restamp-control \
      --rewrite-backend openai-compatible \
      --rewrite-model deepseek-v4-flash \
      --rewrite-base-url https://api.deepseek.com \
      --rewrite-allow-remote \
      --out-dir out/bench-deepseek \
      --tag deepseek-v4-flash

`--markllm-dir` may also come from `MARKLLM_DIR`. The seed corpus defaults to
this repo's `benchmarks/corpus`; point `--corpus` at another directory of
`.txt` files (or a single file) to use your own. Non-loopback rewrite
endpoints require `--rewrite-allow-remote`.

The rewrite API key is never placed on the child process's argv: it is passed
through the environment as `WATERMARKS_REWRITE_API_KEY`. Export that variable
(it is inherited by the benchmark and the rewrite child), or pass
`--rewrite-api-key` if you accept that the key is then visible in this
process's own argv.

Scheme: the benchmark defaults to MarkLLM's `synthid`. `--scheme` accepts any
scheme key of detect_text_watermark.py (e.g. `kgw`) and `--config` overrides
the algorithm config JSON (default `<MarkLLM checkout>/config/<ALG>.json`) -
the same config drives both generation and detection, so a run stays
same-config by construction.

No vendor tier: Google retired SynthID text watermarking on its API in
Aug 2026 (DETECT_TEXT_WATERMARK is rejected on current models), so detection
here is MarkLLM same-config only. A vendor tier can be re-added if Google
exposes detection again (e.g. via Vertex AI).

**How variants map to rewrites:** each <strength>:<candidates> variant runs
the Layer B rewrite with candidates as the **variants per evaluation round**;
`--rewrite-loops` (default 1, mirrors `--max-loops` /
`WATERMARKS_REWRITE_LOOPS`) sets how many rounds run before the best-effort
variant is returned. Strengths come from rewrite_text.py: `paraphrase`,
`backtranslate`, `structural`, `humanize`, `code`. The rewrite is iterative:
it generates a variant, runs MarkLLM detection (same-config) on it, and stops
as soon as an attempt is not watermarked - so a variant usually costs fewer
rewrites than its candidate count, and paraphrase:3 means "try up to 3
variants, stop on the first pass" (raise `--rewrite-loops` to keep retrying
new variants until one passes). The report's att column (and mean_attempts in
results.json / attempts in results.csv) records the actual attempts per
document.

Cost warning: with MarkLLM as the evaluator, each attempt also costs one
MarkLLM detection — up to (candidates x loops) detections per input. The
persistent serve worker (default) keeps the model loaded so detections are
cheap; the --no-worker one-shot path re-loads the model per detection.

Cost modeling: --cost-per-mtok-in 0.30 --cost-per-mtok-out 1.20 (example
prices) attaches an estimated USD figure per row; token counts are
chars / --chars-per-token estimates (default 4.0).

## Sanity gate

A sample only counts once it survives the gate, so a "clear" is always a real
flip and not an artefact of a weak sample:

- the watermarked sample must be at least 50 characters after stripping
  surrounding whitespace,
- and it must be detected as watermarked before removal.

Failures are recorded as `excluded` rows with a reason and are reported in the
Controls section. An unwatermarked control that detects positive does not
exclude the sample; it is flagged as a weak control in the row's notes.

## Outputs (in --out-dir)

- report.md - self-contained Markdown you can paste anywhere: methodology,
  config, results table, controls, caveats, exact reproduction command.
- results.json - full per-sample/per-row data + aggregates.
- results.csv - one row per (doc, seed, variant) for plotting.
- work/ - generated watermarked/unwatermarked samples (kept for inspection).

## Running from Docker (compose)

The `wr-markllm` service in compose.yaml can run the benchmark end-to-end
(image: pinned MarkLLM checkout at /opt/markllm + all scripts). The image
installs CPU torch by design, so use it for portability/CI, not for GPU
throughput on this machine — for GPU runs use the host `setup_markllm.sh`
venv instead (see README).

The service's entrypoint is `detect_text_watermark.py`, which only takes its
own `detect` / `watermark` / `serve` subcommands, so the benchmark needs an
entrypoint override:

```bash
docker compose --profile harness build wr-markllm
docker compose run --rm --entrypoint python3 wr-markllm \
  /app/bench_synthid_text.py --markllm-dir /opt/markllm \
  --corpus /bench-corpus --out-dir /data --tag docker-run \
  --docs 10 --seeds 3 --variants "paraphrase:3,backtranslate:3" \
  --restamp-control
```

`--corpus` is required in the container: the bundled corpus lives outside the
image and is mounted read-only at /bench-corpus. Env (rewrite backend) is
wired from your .env via compose interpolation; results land in the
`bench-out` volume (/data). The benchmark starts its persistent MarkLLM serve
worker inside the container too, so the ~2-4h one-shot runs are not a
constraint there either.

## What it can and cannot claim

- Can claim: under the MarkLLM SynthID scheme config the benchmark
  controls, at these seeds/docs, with this rewrite backend, this clear rate and
  cost were observed. Same-config-only detection is deterministic and
  reproducible (fixed seeds, pinned MarkLLM commit, recorded commands).
- Cannot claim: that Google's production SynthID-Text detector will fail.
  MarkLLM's SynthID is a research reimplementation with a different keying,
  and Google retired text watermark detection on its API (Aug 2026), so no
  vendor tier exists to verify against. Rewriting with a watermarked model
  can also re-stamp the text - run --restamp-control to check.

## Sharing a run

Share the --out-dir directory. report.md embeds the reproduction command,
the MarkLLM commit, the watermarks-remover commit, and the caveats, so a reader
can (a) trust what was measured and (b) rerun it. Keep work/ out of archives
unless you want the raw samples.

## Notes on statistical power

- A single document tells you nothing - the watermark is probabilistic. Use
  several documents (--docs 10+) and several seeds per document
  (--seeds 3+) so clear-rate differences are distinguishable.
- Longer text carries more watermark signal: default --max-new-tokens 300.
  Very short samples are excluded by the sanity gate automatically.
- Compare variants (strength x candidates) within one run, not across runs
  with different backends - the rewrite model dominates the outcome.
