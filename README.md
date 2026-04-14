# diffraction-scatter-skills

> pyFAI 衍射/散射数据处理技能包 — AI 编码agent专用工具集
> pyFAI-based diffraction/scattering data processing skill package for AI coding agents

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![pyFAI](https://img.shields.io/badge/pyFAI-2024.1%2B-orange.svg)](https://www.silx.org/doc/pyFAI/)
[![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey.svg)](#installation)
[![Last Commit](https://img.shields.io/github/last-commit/TianyiMa96/diffraction-scatter-skills/main.svg)](https://github.com/TianyiMa96/diffraction-scatter-skills/commits/main)

[English](#english) | [中文](#中文)

---

![Processing Demo](docs/images/workflow.gif)

---

## Table of Contents / 目录

- [Overview / 概览](#overview--概览)
- [Key Features / 核心功能](#key-features--核心功能)
- [Directory Structure / 目录结构](#directory-structure--目录结构)
- [5 Integration Modes / 五种积分模式](#5-integration-modes--五种积分模式)
- [Quick Start / 快速开始](#quick-start--快速开始)
- [CLI Reference / CLI 参考](#cli-reference--cli参考)
- [Input & Output Formats / 输入输出格式](#input--output-formats--输入输出格式)
- [Installation / 安装指南](#installation--安装指南)
- [If You Don't Have a .poni File / 如果你没有.poni文件](#if-you-dont-have-a-poni-file--如果你没有poni文件)
- [Workflow Diagram / 工作流程图](#workflow-diagram--工作流程图)
- [Architecture / 架构](#architecture--架构)
- [Acknowledgments / 致谢](#acknowledgments--致谢)
- [License / 许可证](#license--许可证)

---

<a id="overview--概览"></a>

## Overview / 概览

**diffraction-scatter-skills** 是一套面向 AI 编码 agent（OpenCode、ClawHub、SkillHub 等）的 pyFAI 衍射/散射数据处理技能包。它提供开箱即用的 CLI 脚本和 Python 库，用于后校准阶段的 X 射线衍射数据（WAXS/SAXS/GIWAXS/XRD）方位角积分。

**diffraction-scatter-skills** is a pyFAI-based diffraction/scattering data processing skill package designed for AI coding agents (OpenCode, ClawHub, SkillHub, etc.). It provides ready-to-use CLI scripts and a Python library for post-calibration azimuthal integration of X-ray diffraction data (WAXS/SAXS/GIWAXS/XRD).

本技能包是**个人项目**，并非 ESRF 或 pyFAI/silx/fabio 团队的官方发布。它基于 pyFAI 的成熟积分引擎，为 AI agent 封装了从几何校准文件（.poni）到一维/二维积分结果的完整批处理工作流。

This skill package is a **personal project**, not an official release from ESRF or the pyFAI/silx/fabio teams. It wraps pyFAI's mature integration engine into a complete batch-processing workflow — from a .poni geometry file to 1D/2D integration results — tailored for AI agents.

---

<a id="key-features--核心功能"></a>

## Key Features / 核心功能

| Feature / 功能 | Description / 描述 |
|---|---|
| **Streaming / 流式处理** | 逐帧处理，突破内存瓶颈，支持海量 HDF5 / Eiger 4D 数据 frame-by-frame processing, never loads entire batch into memory, handles large HDF5/Eiger 4D |
| **5 Integration Modes / 五种积分模式** | radial1d / azimuthal1d / sector1d / cake2d / fiber2d |
| **Multi-format Input / 多格式输入** | EDF, TIFF, HDF5（含 Eiger 4D 多通道 LowThresholdData/HighThresholdData/DiffData），NPY |
| **Multi-format Output / 多格式输出** | CSV, HDF5（含完整元数据），NPZ，manifest.jsonl |
| **Corrections / 校正** | mask（值域+死像素+自定义），dark，flat，polarization，solid-angle |
| **Error Models / 误差模型** | 支持 pyFAI error_model 参数 |
| **JSONL Progress / JSONL 进度** | 实时进度事件输出到 stdout，便于 pipeline 集成 |
| **GIWAXS / Fiber** | 专用 fiber2d 模式，q_ip × q_oop 二维积分 |

---

<a id="directory-structure--目录结构"></a>

## Directory Structure / 目录结构

```
diffraction-scatter/
├── __init__.py              # Package marker / 包标记
├── SKILL.md                 # Skill manifest (trigger description + workflow) / 技能清单
├── lib/
│   ├── __init__.py
│   ├── common.py            # Shared IO, frame iteration, mask building, HDF5 handling
│   │                       # 共享 IO、帧迭代、掩膜构建、HDF5 处理
│   ├── integrate.py         # Streaming integration runner (5 modes) / 流式积分运行器
│   ├── inspect.py           # PONI & detector data inspector / PONI 和探测器数据检查器
│   └── install_env.py      # pyFAI environment setup helper / pyFAI 环境安装辅助
├── scripts/
│   ├── integrate_with_poni.py   # CLI entry: streaming integration / CLI 入口：流式积分
│   ├── inspect_poni.py          # CLI entry: inspect .poni & data / CLI 入口：检查.poni和数据
│   └── install_pyfai_env.py     # CLI entry: create pyFAI venv / CLI 入口：创建 pyFAI 虚拟环境
├── references/
│   ├── quickstart.md        # Quick start guide / 快速入门指南
│   ├── modes.md             # Integration mode reference / 积分模式参考
│   └── streaming.md         # Large dataset streaming strategy / 大数据集流式处理策略
├── docs/
│   └── images/
│       └── workflow.gif   # Processing demo / 处理过程演示
└── .gitignore
```

---

<a id="5-integration-modes--五种积分模式"></a>

## 5 Integration Modes / 五种积分模式

| Mode / 模式 | Description / 描述 | Core pyFAI Call |
|---|---|---|
| `radial1d` | Standard 1D radial → I(q) / I(2θ) / 标准1D径向积分 | `integrate1d_ng` |
| `azimuthal1d` | I(χ) inside a radial range / 固定 q 范围的 χ 积分 | `integrate_radial` |
| `sector1d` | 1D in an azimuthal sector / 扇区 1D | `integrate1d_ng(..., azimuth_range=...)` |
| `cake2d` | Full q-χ / 2θ-χ map / 完整 2D cake | `integrate2d_ng` |
| `fiber2d` | GIWAXS q_ip × q_oop / 纤维 GIWAXS 2D | `FiberIntegrator.integrate2d_grazing_incidence` |

### Unit Selection / 单位选择

| Experiment Type / 实验类型 | Recommended Unit / 推荐单位 |
|---|---|
| WAXS / WAXD / wide-angle / 广角 | `2th_deg` (2θ degrees) |
| SAXS / small-angle / 小角 | `q_nm^-1` (q, nm⁻¹) or `q_A^-1` |
| GIWAXS / fiber / grazing-incidence / 掠入射 | `qip_nm^-1` × `qoop_nm^-1` (fiber2d only) |

---

<a id="quick-start--快速开始"></a>

## Quick Start / 快速开始

### Prerequisites / 前置条件

- Python ≥ 3.9
- .poni geometry file (see [below](#if-you-dont-have-a-poni-file--如果你没有poni文件) if you don't have one)

### Step 1 — Create pyFAI Environment / 步骤 1 — 创建 pyFAI 环境

```bash
python scripts/install_pyfai_env.py --venv .venv-pyfai
```

### Step 2 — Inspect .poni and Data / 步骤 2 — 检查 .poni 和数据

```bash
# Activate environment first
# Linux/macOS
source .venv-pyfai/bin/activate
# Windows
# .venv-pyfai\Scripts\activate

python scripts/inspect_poni.py --poni geometry.poni sample.h5
```

### Step 3 — Run Integration / 步骤 3 — 运行积分

```bash
# 1D radial (WAXS, 2θ in degrees)
python scripts/integrate_with_poni.py \
  --mode radial1d --poni geometry.poni -i "data/**/*.edf" -o output/ \
  --unit 2th_deg --npt 2000

# 1D radial (SAXS, q in nm^-1)
python scripts/integrate_with_poni.py \
  --mode radial1d --poni geometry.poni -i sample.h5 -o output/ \
  --unit q_nm^-1 --npt 2000

# Azimuthal I(χ) at fixed q range
python scripts/integrate_with_poni.py \
  --mode azimuthal1d --poni geometry.poni -i sample.h5 -o output/chi \
  --radial-unit q_nm^-1 --radial-min 2 --radial-max 18 --azimuthal-unit chi_deg

# Sector 1D
python scripts/integrate_with_poni.py \
  --mode sector1d --poni geometry.poni -i "frames/*.tif" -o output/sector \
  --unit q_nm^-1 --azimuth-min -20 --azimuth-max 20

# 2D cake
python scripts/integrate_with_poni.py \
  --mode cake2d --poni geometry.poni -i sample.edf -o output/cake \
  --unit q_nm^-1 --npt-rad 800 --npt-azim 360

# GIWAXS / Fiber 2D
python scripts/integrate_with_poni.py \
  --mode fiber2d --poni geometry.poni -i sample.edf -o output/fiber \
  --unit-ip qip_nm^-1 --unit-oop qoop_nm^-1 \
  --incident-angle 0.2 --sample-orientation 1
```

---

<a id="cli-reference--cli参考"></a>

## CLI Reference / CLI参考

### integrate_with_poni.py / 流式积分主脚本

```bash
python scripts/integrate_with_poni.py [options]
```

#### Core Options / 核心选项

| Option | Description / 描述 | Default / 默认值 |
|---|---|---|
| `--mode` | Integration mode: `radial1d`, `azimuthal1d`, `sector1d`, `cake2d`, `fiber2d` | **Required / 必填** |
| `--poni PATH` | Path to .poni geometry file / .poni 几何文件路径 | **Required / 必填** |
| `-i, --inputs PATH` | Input file/pattern (supports glob and HDF5 paths) / 输入文件或模式 | **Required / 必填** |
| `-o, --output DIR` | Output directory / 输出目录 | **Required / 必填** |
| `--unit UNIT` | Integration unit (e.g., `q_nm^-1`, `2th_deg`, `r_mm`) / 积分单位 | `q_nm^-1` |
| `--npt N` | Number of radial points / 径向点数 | `1000` |

#### Mode-Specific Options / 模式特定选项

**azimuthal1d:**

| Option | Description / 描述 |
|---|---|
| `--radial-unit UNIT` | Radial unit / 径向单位 |
| `--radial-min FLOAT` | Minimum radial value / 最小径向值 |
| `--radial-max FLOAT` | Maximum radial value / 最大径向值 |
| `--azimuthal-unit UNIT` | Azimuthal unit (e.g., `chi_deg`) / 方位角单位 |
| `--npt-rad N` | Radial points for 2D integration / 2D 积分径向点数 |

**sector1d:**

| Option | Description / 描述 |
|---|---|
| `--azimuth-min DEG` | Minimum azimuthal angle (degrees) / 最小方位角 |
| `--azimuth-max DEG` | Maximum azimuthal angle (degrees) / 最大方位角 |

**cake2d:**

| Option | Description / 描述 |
|---|---|
| `--npt-rad N` | Number of radial points / 径向点数 |
| `--npt-azim N` | Number of azimuthal points / 方位角点数 |

**fiber2d:**

| Option | Description / 描述 |
|---|---|
| `--unit-ip UNIT` | In-plane unit (must be `qip_nm^-1`) / 面内单位 |
| `--unit-oop UNIT` | Out-of-plane unit (must be `qoop_nm^-1`) / 面外单位 |
| `--incident-angle DEG` | Incident angle (degrees) / 入射角 |
| `--sample-orientation INT` | Sample orientation / 样品取向 |

#### Correction Options / 校正选项

| Option | Description / 描述 |
|---|---|
| `--mask PATH` | Custom mask file (NPY/TIF/EDF/H5) / 自定义掩膜文件 |
| `--dark PATH` | Dark current file / 暗电流文件 |
| `--flat PATH` | Flat field file / 平场文件 |
| `--valid-min VAL` | Minimum valid intensity / 有效强度下限 |
| `--valid-max VAL` | Maximum valid intensity / 有效强度上限 |
| `--polarization FACTOR` | Polarization factor (-1 to 1) / 偏振因子 |
| `--correct-solid-angle` | Enable solid-angle correction / 启用立体角校正 (default: on) |
| `--no-correct-solid-angle` | Disable solid-angle correction / 关闭立体角校正 |
| `--error-model MODEL` | Error model for uncertainty / 误差模型 |

#### Performance Options / 性能选项

| Option | Description / 描述 | Default / 默认值 |
|---|---|---|
| `--method METHOD` | Integration method: `csr` (fast), `splitpixel` (accurate), `bbox`, `lut` / 积分方法 | `csr` |
| `--no-recursive` | Disable recursive directory scan / 禁止递归扫描子目录 | off |

### inspect_poni.py / .poni 检查脚本

```bash
python scripts/inspect_poni.py --poni geometry.poni [data_file]
```

Displays PONI parameters, detector configuration, wavelength, and optionally inspects HDF5 structure and frame metadata.

显示 PONI 参数、探测器配置、波长，并可选择检查 HDF5 结构和帧元数据。

### install_pyfai_env.py / 环境安装脚本

```bash
python scripts/install_pyfai_env.py --venv PATH [--python PYTHON]
```

Creates an isolated Python virtual environment with pyFAI and all dependencies pre-installed.

创建包含 pyFAI 及所有依赖的独立 Python 虚拟环境。

---

<a id="input--output-formats--输入输出格式"></a>

## Input & Output Formats / 输入输出格式

### Supported Input Formats / 支持的输入格式

| Format | Extension | Details |
|---|---|---|
| European Data Format | `.edf` | Read via fabio |
| Tagged Image File | `.tif` / `.tiff` | Read via fabio |
| Hierarchical Data Format | `.h5` / `.hdf5` | Read via h5py; supports Eiger 4D with `LowThresholdData` / `HighThresholdData` / `DiffData` channels |
| NumPy Array | `.npy` / `.npz` | Read via numpy |

### Output Formats / 输出格式

| Mode | Primary Output | Metadata |
|---|---|---|
| 1D (`radial1d`, `azimuthal1d`, `sector1d`) | `.csv` (tab-separated) + `.h5` (with full metadata) | x (q/2θ/r), I, I sigma, npr |
| 2D (`cake2d`, `fiber2d`) | `.npz` + `.h5` (with full metadata) | q/2θ array, χ array, 2D intensity array |
| Every run | `manifest.jsonl` | Per-frame output records, errors, timing |

---

<a id="installation--安装指南"></a>

## Installation / 安装指南

### As an AI Agent Skill / 作为 AI Agent 技能安装

Copy the entire `diffraction-scatter/` directory into your AI agent's skills folder. The directory structure must be preserved: `lib/` and `scripts/` must remain under the same parent directory.

将整个 `diffraction-scatter/` 目录复制到 AI agent 的 skills 文件夹中。必须保留目录结构：`lib/` 和 `scripts/` 必须在同一父目录下。

| Platform / 平台 | Installation Path / 安装路径 |
|---|---|
| **OpenCode (Windows)** | `C:\Users\<user>\.config\opencode\skills\` |
| **OpenCode (macOS/Linux)** | `~/.config/opencode/skills/` |
| **SkillHub** | `~/.skillhub/skills/` |
| **ClawHub** | `~/.clawhub/skills/` |
| **OpenClaw** | `~/.openclaw/skills/` |

#### Installation Steps / 安装步骤

```bash
# 1. Copy the diffraction-scatter/ directory to your skills folder
# Example for OpenCode on Windows:
copy-item -Recurse diffraction-scatter "C:\Users\<your-user>\.config\opencode\skills\"

# 2. Verify installation
python C:\Users\<your-user>\.config\opencode\skills\diffraction-scatter\scripts\integrate_with_poni.py --help

# 3. Create pyFAI environment (one-time setup)
python C:\Users\<your-user>\.config\opencode\skills\diffraction-scatter\scripts\install_pyfai_env.py --venv .venv-pyfai
```

### As a Python Library / 作为 Python 库安装

```bash
# Clone the repository
git clone https://github.com/TianyiMa96/diffraction-scatter-skills.git
cd diffraction-scatter-skills

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate  # Windows

# Install dependencies
pip install pyFAI>=2024.1 fabio>=2024.1 h5py>=3.7 numpy>=1.22
```

---

<a id="if-you-dont-have-a-poni-file--如果你没有poni文件"></a>

## If You Don't Have a .poni File / 如果你没有.poni文件

A `.poni` file is the output of geometric calibration. It contains detector distance, pixel size, wavelength, beam center, and detector tilt parameters. Follow these steps to get one:

`.poni` 文件是几何校准的输出，包含探测器距离、像素大小、波长、光束中心和探测器倾斜参数。按以下步骤获取：

### Step 1 — Install pyFAI / 步骤 1 — 安装 pyFAI

```bash
python scripts/install_pyfai_env.py --venv .venv-pyfai
source .venv-pyfai/bin/activate  # Linux/macOS
# .venv-pyfai\Scripts\activate  # Windows
```

If Python itself is not installed / 如果没有 Python：
- **Windows**: [Python.org installer](https://www.python.org/downloads/windows/) or Miniconda
- **macOS**: Homebrew (`brew install python`) or Python.org
- **Linux**: `apt install python3 python3-venv` or mamba/conda

### Step 2 — Learn Calibration / 步骤 2 — 学习校准

| Language | Resource / 资源 |
|---|---|
| 中文视频 / Chinese video | [pyFAI 校准教程（B站）](https://www.bilibili.com/video/BV144zuBPEEm/) |
| English video | [pyFAI Calibration Tutorial (silx.org, 15 min)](http://www.silx.org/pub/pyFAI/video/Calibration_15mn.mp4) |
| Documentation | [pyFAI calibration docs](https://www.silx.org/doc/pyFAI/usage/calibration/) |

### Step 3 — Calibrate and Export .poni / 步骤 3 — 校准并导出 .poni

Use pyFAI-calibration2 or the `autocalib` module from the parent [waxs-saxs-manager](https://github.com/TianyiMa96/waxs-saxs-manager) project:

使用 pyFAI-calibration2 或父项目 [waxs-saxs-manager](https://github.com/TianyiMa96/waxs-saxs-manager) 中的 `autocalib` 模块：

```bash
# Full auto-calibration pipeline (Phase 1 → 4)
python -m autocalib /path/to/image.edf LaB6 \
    --detector Pilatus1M \
    --wavelength 0.8 \
    --run-phase3 \
    --run-phase4 \
    --output-poni calibrated.poni

# GUI calibration
pyFAI-calibration2
```

Once you have a `.poni` file, return here and use it with the integration scripts above.

获得 `.poni` 文件后，回到上方使用积分脚本。

---

<a id="workflow-diagram--工作流程图"></a>

## Workflow Diagram / 工作流程图

```
Raw Detector Image                      .poni Geometry File
(.edf / .tif / .h5 / .npy)      ──▶    (detector, wavelength,
       │                                 beam center, tilt, distance)
       │                                       │
       ▼                                       ▼
┌──────────────────────────────────────────────────────────────┐
│                  integrate_with_poni.py                       │
│                                                               │
│  1. Load frame (fabio / h5py / numpy)                         │
│  2. Apply corrections (dark, flat, mask, polarization)        │
│  3. Integrate with pyFAI (mode-specific)                      │
│     ├─ radial1d     →  I(q) or I(2θ)                          │
│     ├─ azimuthal1d  →  I(χ) at fixed q                        │
│     ├─ sector1d     →  I(q) in sector                         │
│     ├─ cake2d       →  2D q-χ map                             │
│     └─ fiber2d      →  q_ip × q_oop GIWAXS map                │
│  4. Write output (CSV/HDF5/NPZ + manifest.jsonl)               │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
  Integration Results
  (1D curves / 2D maps + metadata + manifest)
```

---

<a id="architecture--架构"></a>

## Architecture / 架构

```
diffraction-scatter/
│
├── scripts/                    # CLI entry points / CLI 入口
│   ├── integrate_with_poni.py  #  ← main integration runner
│   ├── inspect_poni.py         #  ← .poni & HDF5 inspector
│   └── install_pyfai_env.py    #  ← environment setup
│
└── lib/                        # Core library modules / 核心库模块
    ├── common.py               #  Frame iteration, IO, mask building
    │                           #  帧迭代、IO、掩膜构建
    ├── integrate.py            #  5-mode integration engine
    │                           #  五模式积分引擎
    ├── inspect.py              #  PONI/detector inspection utilities
    │                           #  PONI/探测器检查工具
    └── install_env.py          #  pyFAI environment installer
                                #  pyFAI 环境安装器
```

### Module Responsibilities / 模块职责

| Module | Responsibility |
|---|---|
| `lib/common.py` | File I/O, HDF5/Eiger handling, frame iteration, mask building |
| `lib/integrate.py` | Streaming runner with 5 integration modes, error models, corrections |
| `lib/inspect.py` | PONI parameter extraction, detector info, HDF5 structure inspection |
| `lib/install_env.py` | Isolated pyFAI venv creation with dependency pinning |
| `scripts/integrate_with_poni.py` | CLI orchestration, argument parsing, output management |
| `scripts/inspect_poni.py` | CLI for .poni and data inspection |
| `scripts/install_pyfai_env.py` | CLI for environment setup |

---

<a id="acknowledgments--致谢"></a>

## Acknowledgments / 致谢

This work is built upon three core open‑source projects:

- **pyFAI** – The fast azimuthal integration Python library.  
  Citation: G. Ashiotis, A. Deschildre, Z. Nawaz, J. P. Wright, D. Karkoulis, F. E. Picca and J. Kieffer; *Journal of Applied Crystallography* (2015) 48(2), 510‑519.  
  DOI: [10.1107/S1600576715004306](https://doi.org/10.1107/S1600576715004306)

- **fabio** – I/O library for 2D X‑ray detector images.  
  Citation: E. B. Knudsen, H. O. Sørensen, J. P. Wright, G. Goret and J. Kieffer; *Journal of Applied Crystallography* (2013) 46, 537‑539.  
  DOI: [10.1107/S0021889813000150](https://doi.org/10.1107/S0021889813000150)

- **silx** – Collection of Python packages for data assessment, reduction and analysis at synchrotron radiation facilities.  
  Citation: silx releases can be cited via their DOI on Zenodo: [10.5281/zenodo.591709](https://doi.org/10.5281/zenodo.591709)

We thank the ESRF and all contributors of these projects for making this work possible.

感谢 ESRF 及上述项目的所有贡献者。

---

<a id="license--许可证"></a>

## License / 许可证

```
MIT License

Copyright (c) 2025 TianyiMa96 (tyma507@iccas.ac.cn)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

[Back to Top / 返回顶部](#diffraction-scatter-skills)
