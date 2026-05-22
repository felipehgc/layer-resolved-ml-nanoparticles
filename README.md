# Layer-Resolved Machine Learning of Nanoparticle Stability

Data, descriptors, and code for the paper **"Interpretable Machine Learning of Nanoparticle Stability through Topological Layer Embeddings"** ([arXiv:2602.17528](https://arxiv.org/abs/2602.17528)).

This repository provides a data-efficient and interpretable machine-learning framework for ranking the stability of chemically complex metallic nanoparticles. The approach is built on a fragmented, layer-resolved descriptor that decomposes each nanoparticle into surface, intermediate, and core environments using a topology-driven definition, coupled with gradient-boosted decision trees and a ranking-based learning strategy.

## Overview

The target system is a multicomponent Al-based nanoparticle with composition Al₇₀Co₁₀Fe₅Ni₁₀Cu₅, modeled as a 55-atom two-shell Mackay icosahedron. Reference energies come from DFT relaxations performed with SIESTA (DZP basis, PBE functional, 250 Ry mesh cutoff, forces relaxed below 0.02 eV/Å). The framework learns to rank configurations by total energy using only a few hundred reference calculations, and exposes the physical drivers of stability through layer-weighting and SHAP-based interpretability.

The descriptor is computed from a chemical connectivity graph. Two atoms are bonded when their distance satisfies `d_ij <= lambda * (r_i + r_j)`, where `r` are ASE natural cutoff radii and `lambda` is a global multiplier (1.20 in this work). Topological shells are then defined by connectivity distance from the surface, which separates surface from interior atoms without arbitrary geometric cutoffs.

## Repository structure

```
.
├── descriptors.py            # build the layer-resolved descriptor table from atomic structures
├── make_file.py              # merge descriptors + DFT energies into the model-ready table
├── sidebyside.py             # alternative merge (descriptors + energies, structure order preserved)
├── make_split.py             # fixed train/test split (seed 42)
├── make_split_10.py          # split variants (10- and 20-structure test sets)
├── make_split_20.py
│
├── rank.py                   # XGBoost + Optuna ranking and active-learning candidate generation
├── param_rank.txt            # configuration for rank.py
├── inter.py                  # layer-resolved interpretability / region analysis
├── param_inter.txt           # configuration for inter.py
├── um.py                     # PCA diagnostics, baseline models, Optuna-tuned XGBoost (main figure)
│
├── train_and_validate.py     # single train/validate run reporting Spearman, Recall@k, Regret@k
├── train_and_validate_k10.py # k-variant
├── run_pipeline.py           # learning-curve driver (train sizes x repeats)
├── run_pipeline_k10.py       # k=10 driver
├── run_all.py                # full sweep over test sizes, k values, and train sizes
├── run_all.sh                # layer-weighting sweep (full / surface / core / intermediate)
│
├── plot_1_spearman.py        # learning-curve plots
├── plot_2_recall5.py
├── plot_3_regret5.py
├── paper_fig.py              # combined article figure
├── supp.py                   # supplementary figures
├── figure_3_alt.py
├── figure_4.py
├── energy_hist.py
│
├── descriptors.csv           # layer-resolved descriptor table (one row per structure)
├── ENERGIAS_norm.csv         # DFT total energies (id, Etot)
├── file.csv                  # merged, standardized table used by the models (id, Etot, features)
├── train_ids.csv / test_ids.csv / train_pool.csv
├── learning_curve_results.csv
├── ranking_xgboost.csv       # predicted energies and stability ranking
├── ranking_new_validation.csv
├── new_structures_log.csv    # active-learning candidates linked to parent templates
│
├── main.tex / supplementary.tex
└── figures (*.pdf, *.png)
```

The relaxed atomic structures are read from a `structures/` directory by `descriptors.py`. Newly proposed candidates from the active-learning step are written to `structures_new/`.

## Requirements

Python 3.9 or newer, with:

```bash
pip install numpy pandas scipy scikit-learn xgboost optuna ase matplotlib
```

Reproducing the SHAP interpretability figure additionally requires the `shap` package.

## Data files

The descriptor table is layer-resolved into three feature blocks identified by prefix:

- `total_*` summarizes the whole nanoparticle
- `bulk_*` summarizes interior (core) atoms
- `surface_*` summarizes outermost-shell atoms

Within each block the descriptor records composition fractions, coordination (graph degree) statistics, bond-length statistics, pairwise neighbor probabilities `P` and Warren-Cowley-type short-range-order parameters `alpha`, a local chemical entropy term, and per-pair bond counts and bond-length means and standard deviations. The number of atoms assigned to each region is reported in `n_atoms_total`, `n_atoms_bulk`, and `n_atoms_surface`.

`file.csv` is the model-ready table produced by merging `descriptors.csv` with `ENERGIAS_norm.csv`. It carries an integer `id`, the DFT target `Etot`, and the standardized descriptor columns.

## Usage

The pipeline runs in the following order. Each step reads the outputs of the previous one.

1. Build the descriptor table from the relaxed structures:

```bash
python descriptors.py        # structures/ -> descriptors.csv
```

2. Merge descriptors with DFT energies and standardize features:

```bash
python make_file.py          # descriptors.csv + ENERGIAS_norm.csv -> file.csv
```

3. Create a fixed train/test split:

```bash
python make_split.py         # -> train_ids.csv, test_ids.csv, train_pool.csv
```

4. Run ranking and generate active-learning candidates (configured through `param_rank.txt`, 300 Optuna trials, 5-fold CV, 80/20 split, seed 42):

```bash
python rank.py               # -> ranking_xgboost.csv, structures_new/, new_structures_log.csv
```

5. Generate the data-efficiency learning curves (Spearman correlation, Recall@k, Regret@k):

```bash
python run_pipeline.py       # -> learning_curve_results.csv
python plot_1_spearman.py
python plot_2_recall5.py
python plot_3_regret5.py
```

6. Reproduce diagnostics and interpretability analyses:

```bash
python um.py                 # PCA, baselines, Optuna-tuned XGBoost
python inter.py              # layer-resolved region analysis (param_inter.txt)
```

To reproduce the layer-weighting comparison (whole particle versus surface, core, and intermediate emphasis), use `run_all.sh`, which runs the model under each weighting scheme.

## Outputs

- `ranking_xgboost.csv`: predicted energies and the resulting stability ranking
- `new_structures_log.csv` and `structures_new/`: active-learning candidates and their parent templates
- `learning_curve_results.csv`: ranking metrics as a function of training-set size and seed
- Figures in PDF and PNG for the main text and supplementary material

## Citation

If you use this code or data, please cite:

```bibtex
@misc{hawthorne2026interpretablemachinelearningnanoparticle,
      title={Interpretable Machine Learning of Nanoparticle Stability through Topological Layer Embeddings},
      author={Felipe Hawthorne and Leandro Seixas and James M. Almeida and Cristiano F. Woellner and Raphael M. Tromer},
      year={2026},
      eprint={2602.17528},
      archivePrefix={arXiv},
      primaryClass={cond-mat.mtrl-sci},
      url={https://arxiv.org/abs/2602.17528},
}
```

Hawthorne, F., Seixas, L., Almeida, J. M., Woellner, C. F., and Tromer, R. M. *Interpretable Machine Learning of Nanoparticle Stability through Topological Layer Embeddings.* arXiv:2602.17528 (2026).

## License

Add a license file (for example MIT or CC-BY-4.0) to clarify reuse terms for the code and data.
