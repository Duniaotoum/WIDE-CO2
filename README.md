# WIDE: ML calibration of CO₂ rovibrational energy levels

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24422946.svg)](https://doi.org/10.5281/zenodo.24422946)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Python 3.12](https://img.shields.io/badge/python-3.12-blue.svg)
![LightGBM](https://img.shields.io/badge/model-LightGBM-2ea44f.svg)

Code accompanying *WIDE: Machine-learning-assisted calibration of rovibrational energy levels of CO₂ isotopologues* (Ibrahim, Alatoom, El-Kork, Yurchenko & Tennyson).

A LightGBM model learns the correction **ΔE = E<sub>MARVEL</sub> − E<sub>Ames</sub>** for the twelve stable CO₂ isotopologues from eight inputs: the quantum labels *J*, *v*₁, *v*₂, *v*₃, *r*, *p*, the carbon mass and the summed oxygen mass. No energy is used as an input. The corrected energy is E<sub>Ames</sub> + ΔÊ.

Two validation schemes are run:

| Scheme | What is held out | What it measures |
|---|---|---|
| **WIDE** (Withheld Isotopologue Delta Estimation) | One whole isotopologue, excluded from fitting and early stopping | Transfer to an unseen isotopologue |
| **Random** | Five shuffled folds of the MARVEL-known levels | Interpolation among species already seen |

## Repository contents

| File | Description |
|---|---|
| `WIDE_Random.ipynb` | Complete WIDE and Random training and prediction code (five cells) |
| `requirements.txt` | Package versions used for the published results |
| `CITATION.cff` | Citation metadata |

The data files are archived on Zenodo, [doi:10.5281/zenodo.24422946](https://doi.org/10.5281/zenodo.24422946):

| File | Description |
|---|---|
| `Input.csv` | 942,021 levels with *J* ≤ 123 and *v*₁+*v*₂+*v*₃ ≤ 8, matched between Ames-2021 and CDSD (AFGL labels from CDSD), with MARVEL energies where available |
| `Predictions.csv` | The same rows and columns, followed by `WIDE_Predictions`, `Ran_Predictions` and `MARVEL_Distance` |

## Quick start

```bash
git clone https://github.com/Duniaotoum/WIDE-CO2.git
cd WIDE-CO2
python -m pip install -r requirements.txt
```

1. Download `Input.csv` from [Zenodo](https://doi.org/10.5281/zenodo.24422946) into the repository folder.
2. Open `WIDE_Random.ipynb` in Jupyter Notebook or JupyterLab, started from this folder.
3. Run the five cells from top to bottom.

The notebook trains 17 models (12 WIDE + 5 Random) with seed 2023 and writes `Predictions.csv`. Training takes several minutes or longer, depending on the computer. The last cell prints the level counts and MAEs against MARVEL for each isotopologue. It can be run on its own, after the first cell, to summarise an existing `Predictions.csv`.

With the pinned versions, the pooled MAEs against MARVEL should be close to:

| | MAE (cm⁻¹) |
|---|---|
| WIDE, seed 2023 | 0.00359 |
| Random, seed 2023 | 0.00175 |
| Ames-2021 (baseline) | 0.02089 |
| CDSD-296 (reference) | 0.00144 |

Other package versions or hardware can produce small numerical differences. For the closest match, keep the default eight threads.

## Data columns

| Column | Meaning |
|---|---|
| `ISO` | AFGL isotopologue code, e.g. 626 = ¹⁶O¹²C¹⁶O, 638 = ¹⁶O¹³C¹⁸O |
| `Label` | State label, `J-v1-v2-l2-v3-r-p` |
| `Band` | Vibrational band label, `v1-v2-l2-v3-r-p` |
| `polyad` | Polyad number *P* = 2*v*₁ + *v*₂ + 3*v*₃ |
| `J`, `v1`, `v2`, `l2`, `v3`, `r`, `p` | AFGL quantum numbers; *r* is the Fermi rank, *p* the e/f parity, and ℓ₂ = *v*₂ |
| `EAmes` | Original Ames-2021 variational energy (cm⁻¹) |
| `ECDSD` | CDSD energy (cm⁻¹), used for comparison only |
| `EMARVEL` | MARVEL energy (cm⁻¹); blank if not measured |
| `Unc`, `Trs` | MARVEL uncertainty (cm⁻¹) and number of transitions |
| `WIDE_Predictions`, `Ran_Predictions` | Corrected energies E<sub>Ames</sub> + ΔÊ (cm⁻¹), seed 2023 |
| `MARVEL_Distance` | Distance to the nearest measured band (see below) |

MARVEL energies are never substituted for predictions: every row has both predictions, including measured levels. To identify measured levels, check whether `EMARVEL` is present.

## Method details

**Mass features.** For an AFGL code `ISO = abc`, *a* and *c* identify the two oxygen atoms and *b* the carbon atom. The notebook computes `m_C = C_MASS[b]` and `m_O_sum = O_MASS[a] + O_MASS[c]`, in u. They are not read from `Input.csv`. For example, 626 gives `m_C` = 12.0000000 and `m_O_sum` = 31.98982924. The digit alone does not identify an isotope (2 can mean ¹²C or deuterium), so set the masses carefully if you adapt the code to another molecule.

**WIDE.** For each isotopologue in turn, the model is trained on the MARVEL-known levels of the other eleven and predicts every level of the withheld one.

**Random.** Five shuffled folds of the MARVEL-known levels. Each measured level is predicted by the fold model that excluded it. Each unmeasured level receives the mean of the five fold models.

In both schemes, each training set is split 80/20 for fitting and early stopping, with no refit afterwards.

**`MARVEL_Distance`** is the smallest Manhattan distance, |Δ*v*₁| + |Δ*v*₂| + |Δ*v*₃|, from a level's vibrational triplet to any triplet with MARVEL data in the same isotopologue. 0 means the triplet is measured (this does not mean the level itself is measured); 1, 2, … are total quantum steps away; `N/A` means the isotopologue has no MARVEL data. It ignores *J*, ℓ₂, *r* and parity. It is descriptive metadata, not a model input or an uncertainty estimate.

## Configuration

All settings are in the first cell: the directory (`Path.cwd()` by default), file names, quantum-number columns, parity map, isotope masses, seed and LightGBM parameters. The benchmark settings are Huber loss with α = 0.02 cm⁻¹, learning rate 0.05, maximum depth 4, at least 10 levels per leaf, and early stopping after 200 rounds without improvement, with up to 50,000 trees.

## Citation

If you use this code or data, please cite the paper and the Zenodo archive. GitHub's **Cite this repository** button provides the citation in BibTeX and APA.

## License

The code is released under the [MIT License](LICENSE). The data on Zenodo carry their own licence.
