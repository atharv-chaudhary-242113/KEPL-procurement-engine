# KEPL Procurement Engine

A deterministic, high-performance procurement analytics and demand forecasting pipeline built for Khokhar Electricals Pvt. Ltd. (KEPL). The system utilizes a hybrid architecture: a memory-safe Rust mathematical core coupled with Python FFI bindings for graphical rendering via PyQt6.

## Architecture

The pipeline replaces legacy scripting abstractions with a static, zero-dependency mathematical execution engine.

* **Memory-Safe Ingestion:** Bifurcated data loading utilizing `calamine` for native `.xlsx` parsing and vectorized fallback for CSV dumps, operating within strict 4GB memory constraints.
* **Financial Precision:** Aggregations execute via `rust_decimal` and fixed-point `i64` arithmetic to eliminate IEEE 754 floating-point drift.
* **Deterministic Deduplication:** O(1) Hash Map entity grouping restricted to isolated columns to maintain SKU integrity.
* **Dynamic Forecasting Router:** Integrates the Syntetos-Boylan Classification (SBC) matrix to dynamically route item data based on Average Demand Interval (ADI) and Squared Coefficient of Variation (CV²).
    * **Smooth Demand:** Processed via native Rust `smartcore` Random Forests.
    * **Erratic Demand:** Processed via pure Rust `ndarray` Generalized Linear Models (GLM) optimized with IRLS.
    * **Intermittent/Lumpy Demand:** Processed via O(1) Teunter-Syntetos-Babai (TSB) extrapolation.

## Requirements

* Rust Toolchain (`rustc` and `cargo`, latest stable)
* Python 3.12.x
* PyQt6

## Build Protocol

The system is designed for isolated, offline compilation to prevent supply chain poisoning.

1. Clone the repository to the local filesystem.
2. Initialize the virtual environment and install Python dependencies:
```bash
   pip install PyQt6
```

3. Vendor all remote Rust crates to lock source code locally:

```bash
  cargo vendor > .cargo/config.toml
```


4. Compile the release binary:
```bash
maturin develop --release
```


*(Alternatively, execute `cargo build --release` targeting the bindings crate).*
5. Execute the graphical interface:
```bash
python run_gui.py

```



## Directory Structure

```text
kepl-procurement-engine/
├── .cargo/
│   └── config.toml               
├── vendor/                       
├── Cargo.toml                    
├── pyproject.toml                
├── run_gui.py                    
├── gui/                          
│   ├── app.py                    
│   ├── views/                    
│   └── widgets/                  
├── bindings/                     
│   ├── Cargo.toml                
│   └── src/
│       └── lib.rs                
└── core_engine/                  
    ├── Cargo.toml                
    └── src/
        ├── lib.rs                
        ├── ingest.rs             
        ├── financials.rs         
        ├── classifier.rs         
        └── forecaster/           

```