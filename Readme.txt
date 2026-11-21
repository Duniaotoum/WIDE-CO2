Files
  WIDE_Random.ipynb: the complete WIDE and Random code.
  Input.csv: 942,021 levels. These levels we obtained after matching the AFGL assignments from CDSD to Ames, then trimmed to limit extrapolation.
  Predictions.csv: same rows and original columns as the input file, followed by WIDE_Predictions, Ran_Predictions and MARVEL_Distance.

How to run
1. Keep all four files in the same folder.
2. Use Python 3.12 with Jupyter Notebook or JupyterLab. Install the numerical
   packages used for the supplied results:
   python -m pip install numpy==2.5.1 pandas==3.0.3 lightgbm==4.7.0 scikit-learn==1.9.0
3. Open WIDE_Random.ipynb in that folder and select its Python environment.
4. The first cell contains all settings: directory, filenames, quantum
   numbers, parity labels, isotope masses, seed and model parameters.
   DIRECTORY = Path.cwd() works when Jupyter starts in this folder.
   Otherwise set DIRECTORY to the full path of this folder.
QUANTUM_NUMBERS = ["J", "v1", "v2", "v3", "r", "p"] : This line specifies the columns that contain the necessary quantum numbers.
BAND_QUANTUM_NUMBERS = ["v1", "v2", "v3"] : This line specifies the columns that contain the vibrational quantum numbers only, for the calculation of distance.
STATE_KEY = ["ISO"] + QUANTUM_NUMBERS : Together these columns identify a physical energy level. "ISO" contains the AFGL isotopologue ID: 626 stands for 16O-12C-16O, 638 stands for 16O-13C-18O, etc. The "ISO" column will also be used for the calculation of masses.
MASS_FEATURES = ["m_C", "m_O_sum"] : These are the names of the two mass features calculated internally from ISO. They do not need to be included in Input.csv.
FEATURES = QUANTUM_NUMBERS + MASS_FEATURES : These are the quantum-number and derived mass columns used as model inputs.
PARITY_LABELS = {"e": 0, "f": 1} : This maps the parity labels to numeric values.
C_MASS = {2: 12.0000000, 3: 13.00335484, 4: 14.0032420} : Carbon masses in atomic mass units (u). The keys 2, 3, 4 denote 12C, 13C, 14C.
O_MASS = {6: 15.99491462, 7: 16.99913170, 8: 17.99915961} : Oxygen masses in u. The keys 6, 7, 8 denote 16O, 17O, 18O.
In AFGL isotopologue labels, each digit must be interpreted in the context of the element it represents. For example, 2 can denote either carbon-12 or deuterium. The digit alone does not uniquely identify the element or isotope, the user must specify the atomic masses carefully.
5. Run the five cells from top to bottom. This trains 17 models (12 WIDE + 5 Random), using seed 2023, and writes Predictions.csv, replacing the supplied output after a successful run.
The final cell displays basic statistics and MAEs in the notebook. No figures, extra output files, model files or prediction caches are created. Training may take several minutes or longer, depending on the computer.


How the mass features are calculated:-
In the AFGL code of CO2 ISO = abc, the first digit a identifies one oxygen atom, the middle digit b identifies carbon, and the last digit c identifies the other oxygen atom. The second code cell extracts these digits:

  a = ISO // 100
  b = (ISO // 10) % 10
  c = ISO % 10
  m_C = C_MASS[b]
  m_O_sum = O_MASS[a] + O_MASS[c]

Both features are in u; m_O_sum is a sum, not an average. For example:
  ISO 626: m_C = 12.0000000; m_O_sum = 15.99491462 + 15.99491462 = 31.98982924.
  ISO 638: m_C = 13.00335484; m_O_sum = 15.99491462 + 17.99915961 = 33.99407423.
Every level of the same isotopologue receives the same pair of masses. The features exist in the internal model frame; the output retains the original input columns and appends only the two predictions and distance.

Predictions
Both methods learn the correction (EMARVEL - EAmes) using J, v1, v2, v3, r, p and two mass features: carbon mass (m_C) and the sum of both oxygen masses (m_O_sum). ISO uses the AFGL three-digit isotopologue code; its middle digit identifies carbon and its outer digits identify oxygen. Parity e/f is encoded internally as 0/1. Original input text is preserved in the output.

WIDE excludes one entire isotopologue from fitting and early stopping, then predicts every level of that isotopologue. Random uses five shuffled
folds of the MARVEL-known levels. Each known level is predicted by the fold model that excluded it; each unmeasured level receives the mean of all five fold models. Each outer training set is split 80/20 for fitting and early stopping, with no subsequent refit.

Both procedures use only seed 2023 (SEED = 2023 in the first cell).
Each output prediction is EAmes plus the learned correction, in cm^-1, from that seed. Random predictions for unmeasured levels still average its five fold models within seed 2023. Every row has both predictions, including rows with MARVEL energies; those energies are not substituted for model predictions.
The notebook uses the same training settings as the supplied original run. For closest numerical reproducibility, retain the stated package versions and the default eight threads. Other versions or hardware can produce small numerical differences.

MARVEL_Distance label:
For each row, find every band with at least one finite MARVEL energy within the SAME isotopologue in Input.csv. Represent a band for this distance by its three vibrational quantum numbers (v1, v2, v3).
The label is the smallest Manhattan distance to those reference bands:

  MARVEL_Distance = min(|v1 - v1_ref| + |v2 - v2_ref| + |v3 - v3_ref|).

0 = the vibrational triplet occurs among MARVEL-known levels. Every measured level has label 0. An unmeasured level in a MARVEL-known vibrational triplet also has label 0; this does not make it measured.
1 = an unseen vibrational triplet one total quantum step from the nearest MARVEL-known triplet in that isotopologue.
2, 3, ... = two, three, ... total quantum steps from the nearest triplet.
This is a descriptive metadata, not a model feature, uncertainty estimate or measured-energy flag.
For example, if the nearest known triplet is (1, 2, 0), then (1, 3, 0) has distance 1 and (2, 3, 0) has distance 2. This definition ignores J, l2, r and parity, even when the original Band text distinguishes them.
It uses the supplied MARVEL coverage, independently of the model folds.
 To identify measured levels, check whether EMARVEL is present. The original spectroscopic Label column is retained.
An isotopologue without any MARVEL-known band receives the literal text "N/A" in MARVEL_Distance. The code continues running and still produces both predictions for those levels.

Basic statistics and MARVEL accuracy
The fifth cell reads Predictions.csv and displays one row per isotopologue
and a Total row. After running the initialization cell, you can run this
last cell on the saved output without retraining.

Total_levels counts input levels; Known_levels counts levels with a finite
EMARVEL energy, including zero-energy levels; Unmeasured_levels counts the
remaining levels. Total_predictions counts energy predictions from both
methods combined (two per input level).

The highly accurate target is MARVEL (EMARVEL). MARVEL_available_% is
100 * Known_levels / Total_levels, calculated per isotopologue and overall.
WIDE_MAE_cm-1 and Random_MAE_cm-1 compare seed-2023 predictions with MARVEL
only where that target is available. The Total MAEs pool absolute errors
across all evaluated levels, weighting each isotopologue by its known-level
count. They are not equal-weight averages of isotopologue MAEs.
An isotopologue with no MARVEL target has 0% availability and N/A for both
MAEs. These statistics are displayed only; no additional file is saved.
