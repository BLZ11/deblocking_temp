# Supporting Data: Physics-Informed Descriptors Enable Machine Learning in Data-Sparse Chemical Systems

Code, data, and Jupyter notebooks that accompany:

> Rafiq, R.; Zulueta, B.; Zucco, H.; Suresh, R.; Shoemaker, J. E.; Call, M.;
> Sheppard, D.; Cormack, G.; Keith, J. A.; Veser, G.
> *Physics-Informed Descriptors Enable Machine Learning in Data-Sparse Chemical Systems.*
> Nature Chemistry (2026, under review).

This repository covers the full pipeline used in the paper: extracting bond
energy (BEBOP) and quantum mechanical descriptors from Gaussian 16 outputs,
augmenting them with RDKit geometric descriptors, and training four regression
models (LASSO, Random Forest, Gaussian Process Regression, Gradient Boosting)
to predict the deblocking temperature ($T_{\text{deblock}}$) of polyurethane
capping agents.

## Repository contents

| File | Description |
|:-----|:------------|
| `qm_bebop_descriptors.ipynb` | Parses Gaussian 16 outputs, runs BEBOP-1, computes nucleophilicity, HOMO-LUMO gap, deprotonation energies, and RDKit geometric descriptors. Writes the consolidated descriptor table. |
| `ml_training_stat_test.ipynb` | Loads the descriptor table, runs the Pearson correlation analysis, trains and benchmarks four regression models under leave-one-out cross-validation, generates parity / residual / learning-curve figures, and reports paired t-tests and bootstrap confidence intervals. |
| `Paper 2 Data Oct 14th.xlsx` | Final descriptor table for 19 training compounds plus 2 held-out external test compounds, with experimental $T_{\text{deblock}}$. |
| `LICENSE` | MIT License for the code in this repository. |
| `README.md` | This file. |

## External resources

Two external resources are required to reproduce the work from raw inputs.

- **Gaussian 16 output files** (~409 MB compressed, ~5.3 GB extracted) are
  archived on Zenodo at https://doi.org/10.5281/zenodo.17883052. They contain
  geometry optimizations at B3LYP-D3/6-31G\*, B3LYP-D3/CBSB7, and G4MP2 for
  every capping agent and MDI-capped adduct, plus HF/CBSB3 single-point
  energies for the MinPop orbital populations used by BEBOP.
- **BEBOP-1** is the bond energy / bond order code from the Keith group at
  the University of Pittsburgh: https://github.com/keithgroup/bebop-qc.

## Two ways to reproduce

The shortest route uses the already-tabulated descriptors in the Excel file
and only runs the ML notebook. The full route regenerates the descriptors
from the raw Gaussian outputs on Zenodo.

### Quick path: ML results only (~2 minutes)

If your goal is to reproduce the figures, statistics, and tables of the
manuscript without recomputing descriptors:

```bash
git clone https://github.com/BLZ11/deblocking_temp.git
cd deblocking_temp

# Create and activate a fresh environment (conda or venv)
conda create -n deblock python=3.11 -y
conda activate deblock

# Install the ML dependencies
pip install numpy pandas scipy scikit-learn matplotlib seaborn openpyxl jupyter

# Launch the notebook
jupyter notebook ml_training_stat_test.ipynb
```

Run all cells from top to bottom. Outputs (PNG and PDF figures) are written
next to the notebook.

### Full path: regenerate descriptors from Gaussian outputs

This route covers everything in the manuscript, including the descriptor
extraction step.

#### 1. Set up the environment

```bash
git clone https://github.com/BLZ11/deblocking_temp.git
cd deblocking_temp

conda create -n deblock python=3.11 -y
conda activate deblock

pip install numpy pandas scipy scikit-learn matplotlib seaborn openpyxl jupyter
pip install rdkit
pip install git+https://github.com/keithgroup/bebop-qc.git
```

The last line installs the BEBOP-1 package directly from the Keith group
repository. If you would rather clone it first and install in editable mode:

```bash
git clone https://github.com/keithgroup/bebop-qc.git
pip install -e ./bebop-qc
```

#### 2. Download the Gaussian outputs from Zenodo

```bash
# From the deblocking_temp/ directory
mkdir -p zenodo_data
cd zenodo_data
wget https://zenodo.org/records/17883052/files/gaussian.tar.gz
tar -xzf gaussian.tar.gz
cd ..
```

The extracted archive provides three calculation subdirectories that the
descriptor notebook expects:

```
zenodo_data/
└── calculations/
    ├── b3lyp_cbsb7/   # B3LYP-D3/CBSB7 geometries + HF/CBSB3 single points
    ├── b3lyp_631g*/   # B3LYP-D3/6-31G* geometries
    └── g4mp2/         # G4MP2 thermochemistry for deprotonation
```

#### 3. Run the descriptor notebook

```bash
jupyter notebook qm_bebop_descriptors.ipynb
```

Before running all cells, update the `DATA_PATH` variable in the **Setup and
Configuration** section to point at your extracted Zenodo download:

```python
DATA_PATH = Path("./zenodo_data")   # or the absolute path on your system
```

The notebook then sequentially:

1. defines the resonance-bond table for each compound,
2. runs BEBOP-1 to compute bond energies, hybridization indices, and resonance energies,
3. computes Domingo's nucleophilicity index $N$ and HOMO-LUMO gaps from the B3LYP/CBSB7 single points,
4. computes the gas-phase deprotonation enthalpy at G4MP2,
5. extracts XYZ coordinates,
6. computes RDKit-based radius of gyration $R_G$ and molar volume $V$,
7. exports the consolidated descriptor table.

#### 4. Run the ML notebook

After step 3 finishes, open `ml_training_stat_test.ipynb` and execute all
cells. The notebook reads from `Paper 2 Data Oct 14th.xlsx` and writes four
publication figures alongside the notebook:

| Output | Description |
|:-------|:------------|
| `combined_correlation_figure.{png,pdf}` | Pearson correlations and descriptor heatmap |
| `parity_bebop_comparison_all_models.{png,pdf}` | 2×4 parity grid: with and without BEBOP descriptors |
| `residual_comparison_SI.{png,pdf}` | 2×4 residual plots (SI figure) |
| `learning_curves.{png,pdf}` | RMSE versus training set size (100 random draws per size) |

Numerical outputs in the notebook include the LASSO equation in standardized
features, LOOCV metrics (RMSE, R², MAE) for every model, AIC and BIC values,
held-out external test predictions, paired t-test statistics, and bootstrap
95% confidence intervals on pairwise differences in RMSE.

## Reproducibility notes

- All random procedures fix `random_state = 42`, including the LASSO
  regularization path, the Random Forest and Gradient Boosting ensembles,
  the bootstrap resampling for $\Delta\mathrm{RMSE}$ confidence intervals,
  and the 100 random subsamples per training-set size used in the learning
  curves.
- Total wall-clock time for `ml_training_stat_test.ipynb` is approximately
  1 to 2 minutes on a single modern CPU. No GPU is required.
- The descriptor notebook is also CPU-bound and runs in approximately 5 to
  10 minutes once the Gaussian outputs are on disk.

## Data file

`Paper 2 Data Oct 14th.xlsx` is the single source of truth for the
machine learning step. It includes 21 capping agents:

- **19 training compounds** used for leave-one-out cross-validation.
- **2 held-out external test compounds** (Octanone Oxime and
  2-Hydroxyethylmethacrylate) used to assess interpolative and
  extrapolative prediction.

Each row contains the experimental $T_{\text{deblock}}$ (in °C), the nine
features used by the ML models, and several precursor columns retained for
transparency. The notebook selects features by name in the `FEATURE_COLS`
and `FEATURE_COLS_NO_BEBOP` lists.

## License

- **Code** in this repository (notebooks and scripts) is released under the
  MIT License. See `LICENSE` for the full text.
- **Gaussian outputs on Zenodo** are released under
  [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/).

If you reuse this code or data, please cite the manuscript above and the
Zenodo deposit (DOI 10.5281/zenodo.17883052).

## Citation

A formatted BibTeX entry will be added once the manuscript is accepted.
Until then, please cite as a manuscript in review:

```bibtex
@article{Rafiq2026deblock,
  author       = {Rafiq, Remsha and Zulueta, Barbaro and Zucco, Hannah and
                  Suresh, Ramakrishna and Shoemaker, Jason E. and Call, Michael and
                  Sheppard, Daylan and Cormack, Glenn and Keith, John A. and Veser, Götz},
  title        = {Physics-Informed Descriptors Enable Machine Learning in Data-Sparse Chemical Systems},
  year         = {2026},
  note         = {Under review}
}

@dataset{Rafiq2026zenodo,
  author       = {Rafiq, Remsha and Zulueta, Barbaro and Zucco, Hannah and
                  Suresh, Ramakrishna and Shoemaker, Jason E. and Call, Michael and
                  Sheppard, Daylan and Cormack, Glenn and Keith, John A. and Veser, Götz},
  title        = {Supporting Data: Physics-Informed Descriptors Enable Machine Learning in Data-Sparse Chemical Systems},
  year         = {2026},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.17883052},
  url          = {https://doi.org/10.5281/zenodo.17883052}
}
```

## Contact

Questions, bug reports, and reproducibility issues are best raised through
the GitHub issue tracker:
https://github.com/BLZ11/deblocking_temp/issues
