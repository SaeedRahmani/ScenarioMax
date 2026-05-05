# ScenarioMax

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> A high-performance toolkit for autonomous vehicle scenario-based testing and dataset conversion

ScenarioMax is an extension to [ScenarioNet](https://github.com/metadriverse/scenarionet) that transforms various autonomous driving datasets into standardized formats. Like ScenarioNet, it first converts different datasets (Waymo, nuPlan, nuScenes) to a unified pickle format. ScenarioMax then extends this process with additional pipelines to convert this unified data into formats compatible with [Waymax](https://github.com/waymo-research/waymax), [V-Max](https://github.com/valeoai/V-Max), and [GPUDrive](https://github.com/Emerge-Lab/gpudrive).


<div align="center">
  <img src="docs/scheme.png" alt="Scheme" width="100%" />
</div>

## 🚀 Key Features

- **Multi-Dataset Support**: Unified interface for Waymo Open Motion Dataset, nuScenes, nuPlan, and OpenScenes
- **Flexible Output Formats**: Convert to TFExample (Waymax/V-Max), JSON (GPUDrive), or unified pickle format
- **High Performance**: Parallel processing with memory optimization and progress monitoring
- **Two-Stage Architecture**: Raw → Unified → Target format pipeline for maximum flexibility
- **Enhanced Scenarios**: Optional scenario enhancement with customizable processing steps

## 📋 Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage Examples](#usage-examples)
- [Supported Datasets](#supported-datasets)
- [Output Formats](#output-formats)
- [Architecture](#architecture)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## 🛠️ Installation

### Prerequisites

- Python 3.10
- [uv](https://docs.astral.sh/uv/) for fast dependency management
- Access to at least one supported dataset (Waymo, nuPlan, or nuScenes)
- Sufficient disk space for dataset processing

### Basic Installation

```bash
# Clone the repository
git clone https://github.com/valeoai/ScenarioMax.git
cd ScenarioMax

# Create and activate virtual environment
uv venv -p 3.10
source .venv/bin/activate

# Install ScenarioMax with dataset support
make womd          # Waymo Open Motion Dataset
make nuplan        # nuPlan dataset
make nuscenes      # nuScenes dataset
make all           # All datasets
make dev           # Development environment
```

### Manual Installation

```bash
# For specific datasets
uv pip install -e ".[womd]"      # Waymo support
uv pip install -e ".[nuplan]"    # nuPlan support
uv pip install -e ".[nuscenes]"  # nuScenes support
uv pip install -e ".[dev]"       # Development tools
uv pip install -e ".[all]"       # All datasets support
```

### Environment Setup

For nuPlan dataset, set required environment variables:

```bash
export NUPLAN_MAPS_ROOT=/path/to/nuplan/maps
export NUPLAN_DATA_ROOT=/path/to/nuplan/data
```

## 🚀 Quick Start

### Basic Dataset Conversion

```bash
# Convert Waymo dataset to TFRecord format
scenariomax-convert \
  --waymo_src /path/to/waymo/data \
  --dst /path/to/output \
  --target_format tfexample \
  --num_workers 8

# Convert nuScenes to GPUDrive format
scenariomax-convert \
  --nuscenes_src /path/to/nuscenes \
  --dst /path/to/output \
  --target_format gpudrive

# Multi-dataset conversion
scenariomax-convert \
  --waymo_src /data/waymo \
  --nuscenes_src /data/nuscenes \
  --dst /output \
  --target_format tfexample
```

## 📊 Usage Examples

### Use Case 1: Raw Data to Pickle (Unified Format)

```bash
# Create unified format for later processing
scenariomax-convert \
  --waymo_src /data/waymo \
  --dst /unified_output \
  --target_format pickle \
  --num_workers 8
```

### Use Case 2: Enhanced Processing Pipeline

```bash
# Raw → Enhanced → TFRecord with scenario enhancement
scenariomax-convert \
  --waymo_src /data/waymo \
  --dst /output \
  --target_format tfexample \
  --enable_enhancement \
  --num_workers 8
```

### Use Case 3: Batch Processing with Multiple Datasets

```bash
# Process multiple datasets with sharding
scenariomax-convert \
  --waymo_src /data/waymo \
  --nuplan_src /data/nuplan \
  --nuscenes_src /data/nuscenes \
  --dst /output \
  --target_format tfexample \
  --shard 1000 \
  --num_workers 16
```

### Use Case 4: Two-Stage Processing

```bash
# Stage 1: Raw → Pickle
scenariomax-convert \
  --waymo_src /data/waymo \
  --dst /intermediate \
  --target_format pickle

# Stage 2: Pickle → Enhanced → TFRecord
scenariomax-convert \
  --pickle_src /intermediate \
  --dst /final_output \
  --target_format tfexample \
  --enable_enhancement
```

## 🗂️ Supported Datasets


| Dataset | Version | Link | Status |
|---------|---------|------|--------|
| Waymo Open Motion Dataset | v1.3.0 | [Site](https://waymo.com/open/download/) | ✅ Full Support |
| nuPlan | v1.1 | [Site](https://www.nuscenes.org/nuplan) | ✅ Full Support |
| nuScenes | v1.0 | [Site](https://www.nuscenes.org/nuscenes) | 🚧 WIP|
| Argoverse | v2.0 | [Site](https://www.argoverse.org/av2.html#forecasting-link) | 🚧 WIP |

### Dataset-Specific Options

```bash
# nuScenes with specific split
scenariomax-convert \
  --nuscenes_src /data/nuscenes \
  --split v1.0-trainval \
  --dst /output \
  --target_format tfexample

# nuPlan with direct log parsing
scenariomax-convert \
  --nuplan_src /data/nuplan \
  --nuplan_direct_from_logs \
  --dst /output \
  --target_format gpudrive
```

## 📤 Output Formats

### TFRecord (TensorFlow/Waymax)

```bash
--target_format tfexample
```

- **Use Case**: Training neural networks with Waymax/V-Max
- **Output**: `training.tfrecord` files with sharding support

### GPUDrive JSON

```bash
--target_format gpudrive
```

- **Use Case**: GPU-accelerated simulation and training
- **Output**: JSON files compatible with GPUDrive simulator

### Unified Pickle Format

```bash
--target_format pickle
```

- **Use Case**: Intermediate format for custom processing
- **Features**: Full scenario data preservation, Python-native
- **Output**: `.pkl` files with complete scenario information

## 🏗️ Architecture

ScenarioMax uses a **two-stage pipeline architecture**:

```
Raw Data → Unified Format → Target Format
    ↓            ↓              ↓
[Dataset]   [Enhancement]  [ML Ready]
```

### Pipeline Stages

1. **Raw to Unified**: Dataset-specific parsers convert native formats to standardized Python dictionaries
2. **Enhancement** (Optional): Apply transformations, filtering, or augmentation
3. **Unified to Target**: Convert to training-ready formats (TFRecord, JSON, etc.)

### Key Components

- **`pipeline.py`**: Main orchestrator with multi-dataset support
- **`dataset_registry.py`**: Dynamic dataset configuration system
- **`raw_to_unified/`**: Dataset-specific extractors and converters
- **`unified_to_*/`**: Target format converters
- **`core/write.py`**: Parallel processing with memory management

## 🔧 Configuration

### Command Line Options

```bash
# Processing options
--num_workers 8              # Parallel workers (default: 8)
--shard 1000                 # Output sharding
--num_files 100              # Limit files processed
--enable_enhancement         # Enable scenario enhancement

# Dataset options
--split v1.0-trainval        # nuScenes data split
--nuplan_direct_from_logs    # Alternative nuPlan parsing

# Output options
--tfrecord_name training     # TFRecord filename
--log_level INFO             # Logging verbosity
--log_file /path/to/log      # Log file location
```


## 📚 Additional Resources

- [Architecture Documentation](docs/ARCHITECTURE.md)
- [API Reference](docs/API.md)
- [nuPlan → GPUDrive Tutorial](docs/nuplan_to_gpudrive.md)


## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
