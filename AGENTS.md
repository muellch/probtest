# AGENTS.md — probtest

Orientation for agents and new developers. This is top-down: it explains the
*shape* of probtest and the non-obvious knowledge you only get after working in
it. It is deliberately **deep on the FOF (feedback-file) comparison path** —
that is where the code is most surprising — and **shallow elsewhere**. Sections
that need a domain expert are marked:

> ⚠️ **Shallow — needs domain knowledge.** Expand this.

For build/install/run/test mechanics, see `README.md` (poetry, Python 3.10,
`poetry run pytest`, `poetry run probtest <command> --help`). This file is about
*understanding*, not setup.

---

## Level 0 — what probtest is

Probtest tests weather/climate models (built around ICON, but the source has no
ICON-specifics). The core problem: model output is **not bit-reproducible** —
change the machine, compiler, or core count and the numbers move. So correctness
can't be "is the output identical?"; it must be "is the output **within
acceptable noise**?"

The idea is **probabilistic**: perturb the model's initial conditions slightly,
run an ensemble, let the ensemble spread define what "acceptable variation"
looks like (a set of **tolerances**), and later **check** a new run against
those tolerances. Within tolerance = PASS, outside = a real regression.

**Lifecycle:** `perturb → run_ensemble → stats → tolerance → check`.

---

## Level 1 — architecture

### Two sides meeting at a common format

```
INPUT SIDE (type-agnostic)        THE "WAIST"        COMPARISON SIDE (type-aware edges)
perturb → run_ensemble      →   pandas DataFrame   →   stats → tolerance → check
(jitter inputs, run model)      (everything funnels    (reduce, derive bounds,
                                 into a DataFrame)       PASS/FAIL)
```

1. **Input side is type-agnostic.** `perturb` jitters NetCDF initial conditions;
   `run_ensemble` runs the model N times. Neither knows or cares what the model
   emits. This is why ensemble *creation* is orthogonal to whether you later
   compare a scalar, a field, or a feedback file — it happens upstream of output
   type entirely.
2. **The waist is the pandas DataFrame.** Every output is funneled into a
   DataFrame so the comparison math can be generic and per-element.
3. **The comparison side re-acquires type-awareness at its edges.** Even though
   everything becomes a DataFrame, the parse/check code forks on file type
   (notably FOF — see below).

### The command map (`engine/`)

`probtest.py` is just a Click group that registers one subcommand per
`engine/*.py`:

| Command | Role |
|---|---|
| `perturb` | jitter NetCDF initial conditions |
| `run_ensemble` | run the model N times |
| `stats` | reduce ensemble output to summary statistics |
| `tolerance` | derive tolerance bounds from ensemble spread |
| `check` | PASS/FAIL a run against tolerances |
| `select_members` | pick representative ensemble members |
| `fof_compare` | compare two feedback (FOF) files directly |
| `performance` / `performance_check` | runtime/timing regression |
| `cdo_table` | NetCDF field comparison via CDO |
| `init` | config scaffolding |

### Where things live

- `engine/` — one CLI subcommand per file.
- `util/dataframe_ops.py` — the generic comparison engine (relative diff,
  tolerance check, the parse + check entry points, the FOF fork).
- `util/fof_utils.py` — FOF-specific logic (split/sort, element comparison,
  detailed logging).
- `util/model_output_parser.py` — parsers that turn raw output into DataFrames.
- `util/utils.py` — `FileInfo`, the `FileType` enum, `expand_fof`, seed helpers.
- `util/log_handler.py` — loggers (incl. `log_dataframe`, which skips empty
  frames).
- `visualize/` — plotting. `tests/` — pytest suite (test data is generated
  dynamically per test).

### Two parser registries — do not confuse them

- `model_output_parser` (in `util/model_output_parser.py`) = `{"netcdf", "csv"}`
  — the field/stats path. **FOF does NOT go through this.**
- `file_name_parser` (in `util/dataframe_ops.py`) = keyed by `FileType`
  (`FOF`, `STATS`) — dispatches by file *type*; used by `tolerance`.

`FileType` lives in `util/utils.py`. `FileInfo` classifies a file as `FOF` when
its name contains `"fof"` or `"ekf"`, else `STATS`.

> ⚠️ **Shallow — needs domain knowledge.** The input side (`perturb`,
> `run_ensemble`) and the `stats`/`tolerance` derivation for the *field/stats*
> path, plus `performance*`, `select_members`, `cdo_table`, and `visualize/`,
> are only understood here at level 0/1. Someone who works with the ICON
> perturbation/ensemble workflow should expand these.

---

## Level 2/3 — FOF (feedback files): the sharp edges

Feedback (FOF) files are data-assimilation outputs: observations plus the
model's equivalent of each observation (`veri_data`). probtest compares two of
them (e.g. a CPU vs GPU run of the model) for PASS/FAIL. This path is the most
surprising part of the codebase. **FOF is "stapled on":** it has its own parser
(`parse_probtest_fof`), a different data shape (a dict of `reports` +
`observation` frames), its own check, and a separate `fof_compare` command. Keep
that in mind — much FOF logic does *not* share code with the stats path.

### The FOF check is two parts

`check_file_with_tolerances` forks on `FileType.FOF` into:

1. **Multiple-solutions / exact-match check** (`check_multiple_solutions_from_dict`):
   over *both* `reports` and `observation` frames, every column *without* a rule
   must match **exactly**; columns *with* a `--rules` entry may differ within an
   allowed set. `veri_data` is skipped here.
2. **`veri_data` tolerance check**: the floating-point physics quantity goes
   through the *generic* relative-diff + `check_variable` machinery (the same one
   stats uses). Tolerances are derived by `tolerance` from ensemble spread.

So: the float quantity (`veri_data`) gets a tolerance; everything else
(positions, ids, flags, …) must match exactly (with `--rules` as an escape
hatch).

### ⚠️ Sorting establishes row correspondence — it is load-bearing

FOF observations are **not positionally stable** between runs; the model may emit
them reordered. The comparison is **positional** (row *i* of file A vs row *i* of
file B), so before comparing, `split_feedback_dataset` **sorts** both files by
identity keys to produce a canonical order. **If the sort does not yield a unique
order, every downstream comparison silently compares unrelated observations.**

- Non-radar files sort by `lat, lon, statid, varno, level, time_nomi`.
- **Radar files sort by `dlat, dlon, …` instead of `lat, lon`.** `lat/lon` is the
  radar *station* position — identical for all observations of one radar, so
  useless as a discriminator. `dlat/dlon` is the per-observation position.
  Detection is `"dlat" in ds.data_vars`.
- On real files the sort keys are unique (no ties), which is what makes the
  positional comparison valid.

### ⚠️ Radar files are over-allocated — `n_*` vs `d_*`

Each dimension has two sizes:

- `n_hdr` / `n_body` (dataset **attributes**) = the **real** counts.
- `d_hdr` / `d_body` (array **dimensions**) = the **allocated** size.

For classic files `n == d`. For **radar (emvorado)** files `d > n`: the arrays are
allocated larger than the real data, and the tail `[n:d]` is **padding**. The
allocated outer dims (`d_*`) are constant across files; only the real counts
vary (e.g. an operational 5-of-20-levels run has `n_hdr=5`, `d_hdr=20`).

Rules of thumb:
- **Identifying variables** by shape needs `d_*` (arrays are stored at `d_*`).
- **Real-data operations** need `n_*`. `split_feedback_dataset` strips padding
  with `isel(d_hdr=slice(0, n_hdr), d_body=slice(0, n_body))` up front.
- `fof_compare`'s size gate and tolerance length use `n_body`, **not** `d_body`.
  Mixing the two silently misaligns the tolerance with the data.

### ⚠️ Padding is NaN — and so is real-missing data

In radar `veri_data`, **missing reflectivity is NaN** (often the majority of the
real region), and padding is *also* NaN. So **NaN cannot distinguish real-missing
from padding.** Therefore padding is removed by **count** (`n_body`), never by
NaN-value. This also means the sentinel substitution (`replace_nan_with_sentinel_float64`)
must run *after* padding is stripped, or it would mask the over-allocation.

`split_feedback_dataset` strips by count, which assumes the real data is
**front-packed** (`[0:n]`, padding as a tail). This holds on real files; a guard
validates it via `dlat` (non-NaN for real obs, NaN for padding — unlike
`veri_data`) and raises if a file is not front-packed.

### ⚠️ `veri_data` is 2-D

`veri_data` has dims `(d_veri, d_body)`. The code assumes a single verification
run; `d_veri == 1`. `split_feedback_dataset` raises if `d_veri > 1` (it would
otherwise emit `n_body × d_veri` rows and misalign everything).

### ⚠️ NaN-vs-real is a failure, by design

A `veri_data` cell that is NaN in one file but a real value in the other means an
observation appeared/disappeared. The relative diff is NaN there, which the
tolerance check would otherwise treat as "within tolerance" and pass silently.
The FOF path forces those cells to fail. Both-NaN still passes (both missing =
equal).

---

## Known issues / open questions

- **Exact-match may be too strict for model-derived metadata (Issue B).** The
  exact-match path requires fields like `mdlsfc` or `qual` to match exactly, but
  these are model-derived/coded and can differ between two hardware runs (float
  rounding flips a discrete classification). For HW-divergence validation
  (e.g. CPU vs GPU) this can cause failures that may or may not be "real". The
  open task is a per-field classification: true observation metadata (exact) vs
  model-derived (tolerance or skip). The skip list and `--rules` mechanism are
  where that lands once decided.
- **Different `n_body` between two files** (a genuinely different observation
  *set*) cannot be compared positionally — it needs an identity-key join, not
  padding tricks. `fof_compare` currently fails loudly (the `n_body` size gate)
  rather than silently misaligning. On real CPU/GPU pairs `n_body` was identical
  (invalid obs are kept as NaN, not dropped → `n_body` is grid-geometry
  determined), so this may never arise in practice — confirm before building a
  join.

> ⚠️ **Shallow — needs domain knowledge.** This list is FOF-focused. Open issues
> and conventions for the perturb/ensemble/stats/tolerance and performance
> subsystems should be added by someone who works in them.
