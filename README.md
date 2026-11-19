# clf-lab-bf-1-backfill-methods

Automated, repeated classifier-improvement iterations for the **bf-1-backfill-methods** family of the termux-monorepo Classifiers suite (#5605), run as a design of experiments (DoE) with alternatives ∧ data layers.

**Task.** chat intent v2 (request ∨ directive ∨ aside ∨ speculation ∨ question ∨ do)

**Labels.** gold: 11 rows of scripts/classifiers/backfill-v1-v2.json (v2 label). The labeled chat corpus is empty upstream, so a SILVER seed is generated each run: chat_intent v2 rules (pinned commit) label sentences from nc-1 chat units ∧ PR titles of the newest live-2 snapshot. Metrics on silver measure rule distillation; `gold_acc` scores the silver-trained model on the 11 gold rows.

## Layout
| path | role |
|---|---|
| `doe.yaml` | current generation: factors {model, features, layer} ∧ levels, cells to run, eval ∧ improve settings, pinned `data_sha` |
| `data/manifest.json` | data sources: public `timerloggedout-spec/termux-monorepo` paths, pinned per generation to one commit |
| `data/layers.py` | adapters ∧ data layers: `raw`, `dedup` (one row per normalized text), `augmented` (dedup + token-dropout/swap copies on train folds only) |
| `run_doe.py` | runs one batch of unrun cells ⇒ appends to `results/ledger.jsonl` (cell id, macro-F1 ± std, acc, runtime, data sha, commit, node) |
| `improve.py` | keeps `champion.json`; when a generation completes, writes the next one: neighbors of the top-k cells (±1 level, new edge levels, layer ∧ feature swaps) + untried combinations; archives the old design in `doe/history/` |
| `.github/workflows/iterate.yml` | every 6h ∧ on dispatch: run_doe ⇒ improve ⇒ commit ⇒ append one `clf-doe` request to the mesh store |

Factors (generation 0, 1/3 fractional factorial ⇒ 12 cells): model {logreg, linear-svm, nb, small-gbm} × features {tfidf-word, tfidf-char, hashing} × data layer {raw, dedup, augmented}; each model ∧ feature carries its own numeric levels. Metric: repeated stratified k-fold macro-F1 (classes with <3 rows drop per run and are listed in the ledger row).

## How Hark ∧ Muse consume it
1. Each iterate run appends `{"kind":"clf-doe","target":"any","args":{"repo":"timerloggedout-spec/clf-lab-bf-1-backfill-methods",…}}` to `timerloggedout-spec/hark-vm-workspace-backup:repo-exports/REQUESTS.jsonl`; this repo is listed under `clf_labs` in `repo-exports/REGISTRY.json`.
2. A node claims cells by cloning this repo and running `pip install -r requirements.txt && python run_doe.py --node <name> --max-cells 4 --budget-s 900` (CPU-only, 2 vCPU / ~1 GiB, ≤900 s). Cells already in the ledger for the same generation ∧ data sha are skipped.
3. The node appends its new ledger lines with the contents API (`tools/append_jsonl.py OWNER/REPO results/ledger.jsonl "<msg>" < line.json`); `results/ledger.jsonl` merges as union, so concurrent appends keep every line.
4. The next iterate run folds those rows into ranking ∧ the champion. `champion.json` is the current best cell; read it for promotion into the monorepo method registry (DOE-FAMILY rule 8, method@version).

All commits end `[skip render]`. Append-only: results are added, never rewritten.
