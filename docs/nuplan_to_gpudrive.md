# nuPlan → GPUDrive Conversion Tutorial

This guide explains how to convert the [nuPlan](https://www.nuplan.org/) dataset into [GPUDrive](https://github.com/Emerge-Lab/gpudrive)-compatible JSON files using ScenarioMax.

## Prerequisites

- Python 3.10
- [uv](https://docs.astral.sh/uv/) package manager
- nuPlan dataset downloaded locally (`.db` files + maps)

## Installation

```bash
git clone https://github.com/valeoai/ScenarioMax.git
cd ScenarioMax

uv venv -p 3.10
source .venv/bin/activate

make nuplan   # installs scenariomax + nuplan-devkit dependencies
```

## Environment Setup

```bash
export NUPLAN_DATA_ROOT=/path/to/nuplan/dataset   # directory with .db files
export NUPLAN_MAPS_ROOT=/path/to/nuplan/maps
```

## Running the Conversion

```bash
scenariomax-convert \
  --nuplan_src $NUPLAN_DATA_ROOT \
  --dst /output/gpudrive \
  --target_format gpudrive \
  --num_workers 8
```

Each nuPlan scenario produces one `.json` file in `/output/gpudrive/`.

### Useful options

| Flag | Default | Description |
|---|---|---|
| `--num_workers` | `8` | Parallel workers |
| `--num_files` | all | Cap the number of scenarios processed |
| `--nuplan_direct_from_logs` | off | Skip the Hydra scenario builder; parse `.db` files directly (faster, less filtering) |
| `--nuplan_scenario_duration` | `15.0` | Scenario length in seconds (implicitly enables direct-from-logs mode when != 15.0) |

---

## Pipeline Internals

ScenarioMax uses a **two-stage** architecture:

```
nuPlan .db files
       │
       ▼
 ┌─────────────────────────────────┐
 │  Stage 1 – Raw → Unified        │
 │  raw_to_unified/datasets/nuplan │
 └─────────────────────────────────┘
       │   UnifiedScenario dict
       ▼
 ┌─────────────────────────────────┐
 │  Stage 2 – Unified → GPUDrive  │
 │  unified_to_gpudrive/           │
 └─────────────────────────────────┘
       │   scenario_<id>.json
       ▼
     /output/
```

### Stage 1 — Raw to Unified (`extractor.py`)

`convert_nuplan_scenario()` calls three extractors that populate the intermediate dictionary:

**Dynamic agents** (`extract_dynamic_agents`)
- Iterates over all tracked objects (vehicles, pedestrians, cyclists) and the ego vehicle across every timestep.
- Produces per-agent trajectory arrays: `position`, `heading`, `velocity`, `valid`, `length`, `width`, `height`.
- Ego velocity is computed by numerical differentiation of position (nuPlan does not expose it directly).

**Static map elements** (`extract_static_map_elements`)
- Queries a 250 m radius around the ego's initial position via the nuPlan map API.
- Extracts: lane centerlines & polygons, road-line boundaries, road edges, crosswalks, and fused road-surface boundaries (via Shapely `unary_union`).
- All coordinates are re-centered on the ego's initial position to avoid numerical issues at large GPS coordinates.

**Dynamic map elements** (`extract_dynamic_map_elements`)
- Reads `TrafficLightStatusData` from every timestep.
- Stores per-frame traffic-light states keyed by **integer** `lane_connector_id`.
- The position of each traffic light is taken from the start of the corresponding `LANE_CONNECTOR` baseline path.

### Stage 2 — Unified to GPUDrive (`convert_to_json.py`)

`convert()` assembles the final JSON from three sub-converters:

**`converter/state.py`** — builds the `objects` list  
Each agent becomes an object with per-timestep position/heading/velocity arrays and a `goalPosition` (last valid position). Agents are tested for road and inter-agent collisions using `trimesh`; colliding objects are flagged.

**`converter/roadgraph.py`** — builds the `roads` list  
Map features are typed and assigned numeric IDs. Road elements of type `ROAD_EDGE_SIDEWALK` and `DRIVEWAY` are filtered out. Road edges are also extracted as line segments to build a collision mesh for the agent-collision checks above.

**`converter/traffic_light.py`** — builds the `tl_states` dict  
One entry per `lane_id` (integer), containing lists of state strings, coordinates, and time indices – one element per timestep.

---

## Output JSON Schema

```jsonc
{
  "name": "<scenario_id>.json",
  "scenario_id": "abc123",

  // One entry per agent (including ego)
  "objects": [
    {
      "id": 0,
      "type": "vehicle",          // vehicle | pedestrian | cyclist | other
      "position": [{"x": 1.2, "y": 3.4, "z": 0.0}, ...],   // T timesteps
      "heading":  [0.42, ...],
      "velocity": [{"x": 0.5, "y": 0.1}, ...],
      "width": 2.0, "length": 4.5, "height": 1.6,
      "valid": [true, true, false, ...],
      "goalPosition": {"x": 10.0, "y": 20.0, "z": 0.0},
      "is_sdc": true,              // true for the ego vehicle
      "mark_as_expert": false,
      "total_distance_traveled": 35.2
    }
  ],

  // One entry per map element (lane, road edge, crosswalk, …)
  "roads": [
    {
      "id": 42,
      "type": "lane",             // lane | road_edge | road_line | crosswalk | …
      "map_element_id": 2,        // numeric type code (Waymax-compatible)
      "geometry": [[x, y, z], ...]
    }
  ],

  // One entry per traffic-light-controlled lane connector
  "tl_states": {
    "1234": {                      // integer lane_connector_id (as string key in JSON)
      "state":      ["stop", "stop", "go", ...],   // T states
      "x": [1.0, ...], "y": [2.0, ...], "z": [0.0, ...],
      "time_index": [0, 1, 2, ...],
      "lane_id":    [1234, 1234, 1234, ...]         // integer values
    }
  },

  "metadata": {
    "sdc_track_index": 0,
    "log_name": "2021.07.16.20.45.29_veh-35_01095_01486",
    "initial_lidar_timestamp": 1626468329450448,
    "map_name": "us-nv-las-vegas-strip",
    "objects_of_interest": [],
    "tracks_to_predict": [],
    "average_distance_traveled": 18.4,
    "scenario_type": "following_lane_with_lead"
  }
}
```

### Traffic-light state values

| Value | Meaning |
|---|---|
| `"go"` | Green |
| `"caution"` | Yellow |
| `"stop"` | Red |
| `"arrow_go"` / `"arrow_caution"` / `"arrow_stop"` | Directional arrows |
| `"flashing_stop"` / `"flashing_caution"` | Flashing lights |
| `"unknown"` | No data |

---

## Two-Stage Processing (Advanced)

For large datasets it can be useful to produce the intermediate unified format first and convert later:

```bash
# Stage 1 – raw → pickle
scenariomax-convert \
  --nuplan_src $NUPLAN_DATA_ROOT \
  --dst /intermediate \
  --target_format pickle \
  --num_workers 8

# Stage 2 – pickle → GPUDrive JSON
scenariomax-convert \
  --pickle_src /intermediate \
  --dst /output/gpudrive \
  --target_format gpudrive \
  --num_workers 8
```

This lets you reuse the extracted unified scenarios for multiple output formats (e.g., both `gpudrive` and `tfexample`) without re-reading the raw database.
