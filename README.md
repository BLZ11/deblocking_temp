# Supporting Data: Bond Energy Descriptors Enable Machine Learning with Limited Data: Design of Capping Agents for Thermoplastic Polyurethane Recycling

Code, data, and Jupyter notebooks that accompany:

> Rafiq, R.; Zulueta, B.; Zucco, H.; Suresh, R.; Shoemaker, J. E.; Call, M.;
> Sheppard, D.; Cormack, G.; Keith, J. A.; Veser, G.
> *Bond Energy Descriptors Enable Machine Learning with Limited Data: Design of Capping Agents for Thermoplastic Polyurethane Recycling.*
> (2026, under review).

This repository covers the full pipeline used in the paper: extracting bond
energy (BEBOP) and quantum mechanical descriptors from Gaussian 16 outputs,
augmenting them with RDKit geometric descriptors and with natural bond orbital
(NBO) and minimal-basis (MinPop) population descriptors, and training six
regression models (LASSO, Ridge, ordinary least squares, Random Forest,
Gaussian Process Regression, Gradient Boosting) under a fully nested
leave-one-out cross-validation to predict the deblocking temperature
($T_{\text{deblock}}$) of polyurethane capping agents.

## Repository contents

| File | Description |
|:-----|:------------|
| `qm_bebop_descriptors.ipynb` | Parses Gaussian 16 outputs, runs BEBOP-1, computes nucleophilicity, HOMO-LUMO gap, deprotonation energies, RDKit geometric descriptors, and the NBO and MinPop population descriptors. Writes the consolidated descriptor table. |
| `ml_training_stat_test.ipynb` | Loads the descriptor table and runs the complete analysis of the Supporting Information in three parts: (A) correlation analysis, nested descriptor selection, ablation, and selection stability; (B) six regression models under nested leave-one-out cross-validation with training, cross-validated, and external errors, BEBOP ablation, AIC/BIC, Williams plots, residuals, learning curves, paired t-tests, and bootstrap confidence intervals; (C) the same models with NBO and MinPop descriptors in place of the BEBOP ones. |
| `Paper_2_Data_Oct_14th_NBO_MinPop.xlsx` | Final descriptor table for 19 training compounds plus 2 held-out external test compounds, with experimental $T_{\text{deblock}}$, the BEBOP, QM, and RDKit descriptors, and the NBO and MinPop control descriptors. |
| `LICENSE` | MIT License for the code in this repository. |
| `README.md` | This file. |

## External resources

Two external resources are required to reproduce the work from raw inputs.

- **Gaussian 16 output files** (~409 MB compressed, ~5.3 GB extracted) are
  archived on Zenodo at https://doi.org/10.5281/zenodo.17883052. They contain
  geometry optimizations at B3LYP-D3/6-31G\*, B3LYP-D3/CBSB7, and G4MP2 for
  every capping agent and MDI-capped adduct, ROHF/CBSB3 single-point energies
  with the MinPop orbital populations used by BEBOP and by the MinPop control
  descriptors, and the NBO 7 analyses at the same level used for the NBO
  control descriptors.
- **BEBOP-1** is the bond energy / bond order code from the Keith group at
  the University of Pittsburgh: https://github.com/keithgroup/bebop-qc.

## Two ways to reproduce

The shortest route uses the already-tabulated descriptors in the Excel file
and only runs the ML notebook. The full route regenerates the descriptors
from the raw Gaussian outputs on Zenodo.

### Quick path: ML results only

If your goal is to reproduce the figures, statistics, and tables of the
manuscript and Supporting Information without recomputing descriptors:

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
next to the notebook. Every step that uses $T_{\text{deblock}}$ (descriptor
selection, standardization, penalty and kernel tuning) is repeated inside each
cross-validation fold, so the notebook is slower than a single-pass fit: expect
30 to 45 minutes on one CPU core, most of it in the nested ablation (Part A,
Section 10) and the learning curves (Part B, Section 29). Set
`RUN_LEARNING_CURVES = False` in Section 13.1 for a pass without the learning
curves (about 15 minutes).

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

The extracted archive provides the calculation subdirectories that the
descriptor notebook expects:

```
zenodo_data/
└── calculations/
    ├── b3lyp_cbsb7/   # B3LYP-D3/CBSB7 geometries + ROHF/CBSB3 single points (MinPop populations)
    ├── b3lyp_631g*/   # B3LYP-D3/6-31G* geometries
    ├── g4mp2/         # G4MP2 thermochemistry for deprotonation
    └── nbos/          # NBO 7 analyses at ROHF/CBSB3, one directory per structure
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
2. runs BEBOP-1 to compute bond energies, hybridization energies, and resonance energies,
3. computes Domingo's nucleophilicity index $N$ and HOMO-LUMO gaps from the B3LYP/CBSB7 single points,
4. computes the gas-phase deprotonation enthalpy at G4MP2,
5. extracts XYZ coordinates,
6. computes RDKit-based radius of gyration $R_G$ and molar volume $V$,
7. reads the NBO (Wiberg bond indices, natural 2s populations) and MinPop
   (minimal-basis bond orders and 2s populations) descriptors from the
   ROHF/CBSB3 outputs,
8. exports the consolidated descriptor table.

#### 4. Run the ML notebook

After step 3 finishes, open `ml_training_stat_test.ipynb` and execute all
cells. The notebook reads from `Paper_2_Data_Oct_14th_NBO_MinPop.xlsx` and
writes the following figures alongside the notebook:

| Output | Description |
|:-------|:------------|
| `combined_correlation_figure.{png,pdf}` | Pearson correlations and descriptor heatmap |
| `ablation_heatmap.{png,pdf}` | Change in nested LOOCV RMSE when each candidate descriptor is removed, all six models |
| `descriptor_spread_cv.png` | Coefficient of variation of each candidate descriptor |
| `parity_bebop_comparison_all_models.{png,pdf}` | 2×4 parity grid (LASSO, RF, GPR, GBR) with and without BEBOP descriptors |
| `parity_bebop_comparison_ridge_ols.{png,pdf}` | 2×2 parity grid for Ridge and OLS |
| `bebop_ablation_8descriptor.{png,pdf}` | Removing one BEBOP descriptor at a time from the eight-descriptor model, with bootstrap intervals |
| `williams_SI.{png,pdf}`, `williams_ridge_ols.{png,pdf}` | Williams plots (leverage against standardized residual) for all six models |
| `residual_comparison_SI.{png,pdf}`, `residual_comparison_ridge_ols.{png,pdf}` | Residual plots for all six models |
| `learning_curves_SI.{png,pdf}`, `learning_curves_ridge_ols.{png,pdf}` | RMSE versus training set size (100 random draws per size) |
| `reorganization_descriptor_agreement.png` | BEBOP hybridization energy against the NBO and MinPop 2s populations |
| `parity_frameworks_lasso.png` | LASSO parity plots with BEBOP, NBO, and MinPop descriptors |

Numerical outputs in the notebook include the nested cross-validation ladder
(selection, scaler, and penalty leakage closed one at a time), selection
stability across folds, the LASSO equation in standardized features, the
training, nested LOOCV, and external-set errors (RMSE, R², MAE) for every
model, effective parameter counts and AIC/BIC values, held-out external test
predictions, leverage and applicability-domain summaries, paired t-test
statistics, bootstrap 95% confidence intervals on pairwise differences in
RMSE, and the statistical comparison of the BEBOP, NBO, and MinPop models.

## Reproducibility notes

- All random procedures fix `random_state = 42`, including the LASSO and
  Ridge regularization paths, the Random Forest and Gradient Boosting
  ensembles, the Gaussian Process kernel optimizer restarts, the bootstrap
  resampling for $\Delta\mathrm{RMSE}$ confidence intervals, the Monte Carlo
  perturbations used for generalized degrees of freedom, and the 100 random
  subsamples per training-set size used in the learning curves.
- Cross-validation is fully nested: descriptor selection (Part A), the
  standardization scaler, and every hyperparameter that depends on
  $T_{\text{deblock}}$ (LASSO and Ridge penalties, Gaussian Process kernel)
  are refit inside each leave-one-out fold. Reported cross-validated errors are
  therefore higher than a single-pass fit on the same data would give; the
  training-set errors reported alongside them show that the models did not
  change.
- Total wall-clock time for `ml_training_stat_test.ipynb` is approximately
  30 to 45 minutes on a single modern CPU core. No GPU is required.
- The descriptor notebook is also CPU-bound and runs in approximately 5 to
  10 minutes once the Gaussian and NBO outputs are on disk.

## Data file

`Paper_2_Data_Oct_14th_NBO_MinPop.xlsx` is the single source of truth for the
machine learning step. It includes 21 capping agents:

- **19 training compounds** used for nested leave-one-out cross-validation.
- **2 held-out external test compounds** (2-octanone oxime and
  2-hydroxyethyl methacrylate) used to assess interpolative and
  extrapolative prediction.

Each row contains the experimental $T_{\text{deblock}}$ (in °C), the
candidate descriptors from which the eight-descriptor model is selected
(five conventional descriptors and the BEBOP gross bond energies,
hybridization energy, and resonance energy), the NBO and MinPop control
descriptors (Wiberg bond indices, natural 2s populations, minimal-basis bond
orders and 2s populations), and several precursor columns retained for
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
  title        = {Bond Energy Descriptors Enable Machine Learning with Limited Data: Design of Capping Agents for Thermoplastic Polyurethane Recycling},
  year         = {2026},
  note         = {under review}
}

@dataset{Rafiq2026zenodo,
  author       = {Rafiq, Remsha and Zulueta, Barbaro and Zucco, Hannah and
                  Suresh, Ramakrishna and Shoemaker, Jason E. and Call, Michael and
                  Sheppard, Daylan and Cormack, Glenn and Keith, John A. and Veser, Götz},
  title        = {Supporting Data: Bond Energy Descriptors Enable Machine Learning with Limited Data: Design of Capping Agents for Thermoplastic Polyurethane Recycling},
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
