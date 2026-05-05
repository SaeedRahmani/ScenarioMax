# Project Handoff — Data Pipeline Integration

This file is a self-contained brief for an AI agent continuing work on integrating
the three components (data conversion, scenario filtering, training/evaluation) into
a single project.

---

## 1. Repositories

| Role | URL | Branch | Notes |
|---|---|---|---|
| **Data conversion** | https://github.com/SaeedRahmani/ScenarioMax | `main` | Fork of valeoai/ScenarioMax with custom changes |
| **Scenario filtering** | *(provide URL)* | *(provide branch)* | Filters invalid/low-quality scenarios from GPUDrive JSONs |
| **Training & evaluation** | *(provide URL)* | *(provide branch)* | Training pipeline that consumes filtered GPUDrive JSONs |

---

## 2. What SaeedRahmani/ScenarioMax does

ScenarioMax converts raw autonomous driving datasets (nuPlan, Waymo, nuScenes) into
formats usable by simulators and training pipelines.

### Output format used in this project: GPUDrive JSON

Each scenario produces one `.json` file:

```
/output/
  scenario_abc123.json
  scenario_def456.json
  ...
```

### Top-level JSON structure

```jsonc
{
  "name": "scenario_abc123.json",
  "scenario_id": "abc123",
  "objects": [...],      // agent trajectories (including ego)
  "roads":   [...],      // map features (lanes, edges, crosswalks, …)
  "tl_states": {...},    // traffic-light states, keyed by integer lane_connector_id
  "metadata": {...}
}
```

Full schema documented in: `docs/nuplan_to_gpudrive.md`

### Running nuPlan → GPUDrive conversion

```bash
# Prerequisites
export NUPLAN_DATA_ROOT=/path/to/nuplan/dataset   # directory with .db files
export NUPLAN_MAPS_ROOT=/path/to/nuplan/maps

# Install
git clone https://github.com/SaeedRahmani/ScenarioMax.git
cd ScenarioMax
uv venv -p 3.10 && source .venv/bin/activate
make nuplan

# Run
scenariomax-convert \
  --nuplan_src $NUPLAN_DATA_ROOT \
  --dst /output/gpudrive \
  --target_format gpudrive \
  --num_workers 8 \
  --nuplan_scenario_duration 15.0
```

See `docs/nuplan_to_gpudrive.md` for all CLI options and an explanation of the
two-stage pipeline internals.

---

## 3. Changes in SaeedRahmani/ScenarioMax vs valeoai/ScenarioMax (upstream)

These are the 4 commits not yet in `valeoai/ScenarioMax`:

```
5e5738d  docs: add nuplan to gpudrive conversion tutorial
48a5161  changed duration + fix traffic light
7b02f48  Merge branch 'main' of https://github.com/SaeedRahmani/ScenarioMax
8137a77  updated ReadMe for nuPlan
```

### Functional changes (all in commit `48a5161`)

**`scenariomax/raw_to_unified/datasets/nuplan/extractor.py`**
- Bug fix: `str(tl.lane_connector_id)` → `tl.lane_connector_id`
  Traffic-light `lane_id` values are now stored as **integers** throughout the
  pipeline instead of strings.

**`scenariomax/raw_to_unified/datasets/nuplan/load.py`**
- `get_nuplan_scenarios()` gains two new parameters:
  - `scenario_duration: float = 15.0` — configurable scenario length in seconds
  - `direct_from_logs: bool = False` — skip the Hydra scenario builder and parse
    `.db` files directly (faster, no Hydra dependency at runtime)
- When `scenario_duration != 15.0`, direct-log parsing is automatically selected.
- `get_nuplan_scenarios_by_scene()` also gains the `scenario_duration` parameter
  so it honours the requested clip length instead of a hardcoded `19.0 s` minimum.

**`scenariomax/convert_dataset.py`**
- New CLI flag: `--nuplan_scenario_duration` (default `15.0`)

**`scenariomax/core/pipeline.py`**
- `scenario_duration` and `direct_from_logs` are threaded through from CLI args to
  the dataset loader.

---

## 4. Recommended project structure

```
my-project/
├── pyproject.toml
├── scenariomax/        ← git submodule: SaeedRahmani/ScenarioMax
├── scenario_filter/    ← filtering code (invalid/low-quality scenario removal)
├── training/           ← training & evaluation pipeline
└── README.md
```

### Setting up

```bash
# Clone your project
git clone <my-project-url>
cd my-project

# Add ScenarioMax as a submodule
git submodule add https://github.com/SaeedRahmani/ScenarioMax.git scenariomax
git submodule update --init

# Install ScenarioMax
cd scenariomax
uv venv -p 3.10 && source .venv/bin/activate
make nuplan
cd ..
```

### `pyproject.toml` dependency entry (if using uv)

```toml
[tool.uv.sources]
scenariomax = { path = "./scenariomax", editable = true }
```

### Updating ScenarioMax later

```bash
cd scenariomax
git pull origin main
cd ..
git add scenariomax
git commit -m "bump scenariomax submodule"
```

---

## 5. Data flow across the three components

```
nuPlan .db files
       │
       ▼
┌─────────────────────────────────────┐
│  SaeedRahmani/ScenarioMax           │
│  scenariomax-convert --target_format gpudrive │
└─────────────────────────────────────┘
       │  /output/gpudrive/*.json
       ▼
┌─────────────────────────────────────┐
│  scenario_filter/                   │
│  Removes invalid/low-quality JSONs  │
└─────────────────────────────────────┘
       │  /filtered/*.json
       ▼
┌─────────────────────────────────────┐
│  training/                          │
│  Training & evaluation pipeline     │
└─────────────────────────────────────┘
```

The filtering step reads the GPUDrive JSON files and drops scenarios that do not
meet quality criteria (e.g., too few valid agents, degenerate trajectories, missing
map data). The training pipeline then consumes the filtered JSON directory.

The agent continuing this work should:
1. Confirm the exact interface expected by the filtering code (input directory of
   `.json` files, output directory of filtered `.json` files, any config flags).
2. Confirm the exact interface expected by the training pipeline (input directory,
   any metadata files it requires alongside the JSONs).
3. Wire the three components together via a top-level script or Makefile in
   `my-project/`.
