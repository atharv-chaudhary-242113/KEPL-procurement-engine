# KEPL Procurement Engine

A deterministic, high-performance procurement analytics and demand forecasting engine built for Khokhar Electricals Pvt. Ltd. (KEPL). The system utilizes a hybrid architecture consisting of an offline-compiled, memory-safe Rust mathematical core linked via Python FFI (Foreign Function Interface) bindings to a PyQt6 presentation layer.

## Architecture

The system eliminates third-party runtime interpretation, dynamic memory bloating, and unverified compilation dependencies by routing execution through a strict mathematical engine.

- **Memory-Safe Ingestion:** Bifurcated stream loading utilizing the `calamine` crate for native, non-streaming `.xlsx` parsing and the `csv` crate for vectorized, chunked fallbacks. The engine isolates memory buffers to prevent Out-Of-Memory (OOM) operating system kills on 4GB host configurations.
- **Financial Precision:** Aggregation of ledger transaction values occurs entirely via fixed-point math using the `rust_decimal` crate and custom `i64` scaling. This eliminates IEEE 754 floating-point truncation, machine epsilon drift, and structural sorting anomalies during ABC Pareto classifications.
- **Deterministic Deduplication:** Duplicate vendor names are resolved using a deterministic, linear-time $O(N)$ Hash Map grouping algorithm. Regex-based stripping of branch details is restricted strictly to the `Particulars` column, leaving SKU designations in `Item Details` completely untouched.
- **Dynamic Forecasting Router:** Evaluates all active SKUs across a $2 \times 2$ Syntetos-Boylan Classification (SBC) grid using two structural metrics: Average Demand Interval (ADI) and Squared Coefficient of Variation ($CV^2$).
   - **Smooth Demand (**$\text{ADI} < 1.32$ **and** $CV^2 < 0.49$**):** Evaluated via native Rust Random Forest regressors compiled through `smartcore`.
   - **Erratic Demand (**$\text{ADI} < 1.32$ **and** $CV^2 \ge 0.49$**):** Modeled using pure Rust `ndarray` Generalized Linear Models (GLM) optimized with Iteratively Reweighted Least Squares (IRLS).
   - **Intermittent & Lumpy Demand (**$\text{ADI} \ge 1.32$**):** Projected via Teunter-Syntetos-Babai (TSB) discrete mathematical equations, preventing the artificial trend-decay failure typical of standard machine learning on zero-inflated arrays.

## Technical Specifications & Requirements

- **Rust Toolchain:** `rustc` and `cargo` (Latest stable release)
- **Python Runtime:** Python 3.12.x (Strictly enforced)
- **Build Pipeline Engine:** `maturin` (for Rust-to-Python wheel compilation)
- **Interface Layer:** `PyQt6` and `matplotlib` (for mathematical plot rendering)

## Workspace Manifest Configurations

To recreate the codebase, establish these exact configuration files inside your workspace root.

### 1. Workspace Configuration: `/Cargo.toml`

Defines the virtual membership of the compiled library and core computational engine.

```
[workspace]
members = ["core_engine", "bindings"]
resolver = "2"
```

### 2. Core Engine Manifest: `/core_engine/Cargo.toml`

Handles mathematical parsing, fixed-point operations, and native machine learning arrays.

```
[package]
name = "core_engine"
version = "0.1.0"
edition = "2021"

[lib]
name = "core_engine"
crate-type = ["rlib"]

[dependencies]
calamine = "0.24.0"
csv = "1.3.0"
rust_decimal = "1.34.0"
ndarray = "0.15.6"
smartcore = { version = "0.3.0", features = ["ndarray"] }
```

### 3. FFI Bindings Manifest: `/bindings/Cargo.toml`

Compiles the Rust crate into a C-compatible shared library (`.pyd` on Windows, `.so` on Unix) accessible via Python imports.

```
[package]
name = "bindings"
version = "0.1.0"
edition = "2021"

[lib]
name = "kepl_core"
crate-type = ["cdylib"]

[dependencies]
pyo3 = { version = "0.21.2", features = ["extension-module", "abi3-py312"] }
core_engine = { path = "../core_engine" }
```

### 4. Project Configuration & Python Tooling: `/pyproject.toml`

Determines the package build parameters and locks the Python dependencies.

```
[build-system]
requires = ["maturin>=1.5,<2.0"]
build-backend = "maturin"

[project]
name = "kepl-procurement-engine"
version = "1.0.0"
requires-python = "==3.12.*"
classifiers = [
    "Programming Language :: Rust",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "PyQt6==6.6.1",
    "matplotlib>=3.7"
]

[tool.maturin]
features = ["pyo3/extension-module"]
manifest-path = "bindings/Cargo.toml"
module-name = "kepl_core"
```

## Input Data Specifications

The ingestion system implements strict column schema validation. Files loaded through the interface must match the following structures:

### 1. Purchase Order Vouchers (POV) & Purchase Vouchers (PV)

Must include these columns:

- `Date` — Format: `DD-MM-YYYY` (Parsed dynamically to timestamp)
- `Vch/Bill No` — Unique transaction identifier
- `Particulars` — Supplier entity name (Target for O(1) deduplication)
- `Item Details` — Material SKU identifier (Preserved exactly)
- `Qty.` — Numeric volume field
- `Price` — Unit cost (Parsed as `Decimal`)
- `Amount` — Total line expenditure (Parsed as `Decimal`)
- `Unit` — Material unit of measure

### 2. Goods Received Notes (GRN)

Must include these columns:

- `Date` — Actual date of delivery arrival
- `Vch/Bill No` — GRN voucher identifier
- `Particulars` — Supplier entity name
- `Item Details` — Material SKU identifier
- `Qty.` — Numeric physical volume received
- `Unit` — Material unit of measure

## Build & Local Installation Protocol

Execute these terminal commands sequentially to compile and link the native binary locally.

1. **Clone the Repository:**

    ```
    git clone https://github.com/your-username/kepl-procurement-engine.git
    cd kepl-procurement-engine
    ```

2. **Initialize Virtual Environment:**

    ```
    python -m venv .venv
    # On POSIX (macOS / Linux):
    source .venv/bin/activate
    # On Windows:
    .venv\Scripts\activate
    ```

3. **Install Base Tooling & Build Dependencies:**

    ```
    pip install maturin PyQt6 matplotlib
    ```

4. **Vendor Upstream Rust Packages (Offline Containment):**
   Locks the exact source code of all dependencies into the local filesystem to secure the compilation pathway against upstream supply chain poisoning.

    ```
    cargo vendor > .cargo/config.toml
    ```

5. **Compile & Link Native FFI Module:**

    ```
    maturin develop --release
    ```

6. **Run the Desktop GUI:**

    ```
    python run_gui.py
    ```


## Directory Structure

The repository maintains absolute decoupling across the presentation, serialization, and mathematical execution layers:

```
kepl-procurement-engine/
├── .cargo/
│   └── config.toml               # Relocates build dependencies to local vendor folder
├── vendor/                       # Vendored local Rust package dependencies (Git-ignored)
├── Cargo.toml                    # Root workspace coordinator
├── pyproject.toml                # Maturin compilation and python setup configuration
├── run_gui.py                    # GUI Entrypoint
├── gui/                          # PyQt6 Presentation Layer
│   ├── app.py                    # Main app constructor and shell configuration
│   ├── views/
│   │   ├── file_selection_view.py# Native source path file selection forms
│   │   ├── output_config_view.py # Operational configurations, targets, and launch trigger
│   │   └── results_view.py       # Tabular data grid showing ABC metrics and embedded charts
│   └── widgets/
│       └── abc_chart_widget.py   # Matplotlib canvas rendering log-scale Pareto plots
├── bindings/                     # PyO3 FFI Translation Bridge
│   ├── Cargo.toml                # Compiles target dynamic library
│   └── src/
│       └── lib.rs                # Python-to-Rust type bindings, GIL release hooks
└── core_engine/                  # Sovereign Mathematical Engine
    ├── Cargo.toml                # Static compilation manifest
    └── src/
        ├── lib.rs                # Crate entry point
        ├── ingest.rs             # Excel calamine parser & O(1) Hash Map grouping
        ├── financials.rs         # rust_decimal aggregation routines with safety overflows
        ├── classifier.rs         # SBC metric computation (ADI and CV2 engine)
        └── forecaster/           # Demand Forecasting Suite
            ├── mod.rs            # Compilation router using structural dispatch enums
            ├── random_forest.rs  # smartcore ensemble execution mapping
            ├── glm.rs            # ndarray iteratively reweighted least squares engine
            └── tsb.rs            # Teunter-Syntetos-Babai discrete extrapolation updates
```