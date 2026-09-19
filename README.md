# MNE-analysis

EEG analysis notebooks for a pressure pain experiment recorded with BCI2000. The notebooks use [MNE-Python](https://mne.tools) to study event related potentials (ERP) and power spectral density (PSD) in participant **P001, session S001, runs R02 to R05**.

Each notebook preprocesses the four runs independently and then concatenates their epochs, so all statistics are computed on the complete dataset (120 trials, about 40 per condition).

## Experiment

Every trial follows the same sequence of phases, stored in the BCI2000 state `StimulusPhase`:

**Fixation** (phase 0) ➜ **Inflating** (phase 1) ➜ **Holding** (phase 2, at least 10 s) ➜ deflating and rating

A pneumatic cuff on the arm is inflated to one of three pressure targets, read from the `PressureTarget` state:

| Condition | Pressure target |
| --- | --- |
| `none` | 8 mmHg |
| `low` | 46 mmHg |
| `high` | 111 mmHg |

Recordings have 32 EEG channels (standard 10/20 montage) sampled at 256 Hz.

## Notebooks

| Notebook | Locked to | What it does |
| --- | --- | --- |
| [`Analysis_P001S001_Fixation-ERP.ipynb`](https://colab.research.google.com/github/Albertomhz01/MNE-analysis/blob/main/Analysis_P001S001_Fixation-ERP.ipynb) | Fixation onset | Checks that there is a real visual evoked response over an occipital ROI. Includes split half reliability, a circular shift surrogate test, a one sample cluster permutation test against zero, a negative control by upcoming condition and a per run consistency check. |
| [`Analysis_P001S001_Holding-ERP.ipynb`](https://colab.research.google.com/github/Albertomhz01/MNE-analysis/blob/main/Analysis_P001S001_Holding-ERP.ipynb) | Holding onset | Compares `none`, `low` and `high` over a central ROI (0 to 9.5 s). Runs a spatiotemporal cluster permutation F test across the three conditions, then pairwise tests with Bonferroni correction (α = 0.05 / 3). |
| [`Analysis_P001S001_Holding-PSD.ipynb`](https://colab.research.google.com/github/Albertomhz01/MNE-analysis/blob/main/Analysis_P001S001_Holding-PSD.ipynb) | Holding onset, referenced to the fixation of the same trial | Spectral analysis of the holding window, following four MNE protocols (see below). |

### Holding PSD sections

1. **Spectrum and EpochsSpectrum classes.** Multitaper PSD of the holding window (2 to 40 Hz), butterfly plots per condition and band topographies (delta to gamma) on a common color scale. Based on [this tutorial](https://mne.tools/stable/auto_tutorials/time-freq/10_spectrum_class.html).
2. **Source PSD in a label.** dSPM inverse on the `fsaverage` template (no individual MRI is available), with the noise covariance taken from the fixation epochs. Spectra are computed for the postcentral gyrus (primary somatosensory cortex) in both hemispheres and saved as `.stc` files. Based on [this example](https://mne.tools/stable/auto_examples/time_frequency/source_power_spectrum.html).
3. **Sensor time frequency analysis.** Welch mean vs median, Morlet power and inter trial coherence per condition, expressed as log10(holding / fixation). Based on [this tutorial](https://mne.tools/stable/auto_tutorials/time-freq/20_sensors_time_frequency.html).
4. **Cluster statistics on single trial power.** Omnibus F test and pairwise tests on time frequency power averaged over the central ROI. Based on [this tutorial](https://mne.tools/stable/auto_tutorials/stats-sensor-space/50_cluster_between_time_freq.html).

## Pipeline

All three notebooks share the same preprocessing, applied to each run separately:

1. **Load** the `.dat` file with `mne.io.read_raw_bci2k`, rename channels from the BCI2000 header and set the `standard_1020` montage.
2. **Check units.** The samples are converted from µV to V only after confirming that `SourceChGain` is 1 and `SourceChOffset` is 0, because `read_raw_bci2k` does not apply them.
3. **Band pass filter** from 0.1 to 40 Hz.
4. **Detect bad channels** on the filtered signal using a robust z score of each channel's standard deviation (MAD, threshold 5.0) plus a flat channel test. Fp1 and Fp2 are only tested for flat or saturated signal, since their variance comes from blinks.
5. **Interpolate** bad channels, then apply an **average reference** projector. Interpolating first stops one disconnected electrode from contaminating every channel.
6. **ICA** (20 components) to remove blink components, found automatically with `find_bads_eog` using Fp1 or Fp2 as the EOG proxy.
7. **Epoch** and attach metadata (`run`, `trial`, `condition`).

After the loop, the epochs of the four runs are concatenated and cleaned once with **AutoReject**, and the notebooks check that the three conditions remain balanced.

Runs are preprocessed separately because bad channels differ between them (for example Fp2 in R04, and T8 in R03 and R05), and an average reference computed on the raw concatenation would be contaminated.

The notebooks also report a **marker timing check**. BCI2000 updates states once per sample block (32 samples, 125 ms at 256 Hz), which limits the timing precision of the event markers.

## Running the notebooks

The notebooks are set up for Google Colab. Click a notebook link above to open it there.

1. Upload the raw BCI2000 files to `/content/`:
   ```
   P001S001R02.dat
   P001S001R03.dat
   P001S001R04.dat
   P001S001R05.dat
   ```
   The data are **not included** in this repository. To run locally, change `ROOT` in the Settings cell to the folder that holds the files.
2. Run all cells. The first cell installs the dependencies:
   ```
   pip install numpy pandas matplotlib scipy mne autoreject BCI2kReader
   ```
   The source analysis in the PSD notebook also needs `nibabel`, and the first run downloads the `fsaverage` template (about 200 MB) to `~/mne_data`.

The PSD notebook writes its source spectra and a `cluster_summary.csv` to `ROOT/stc_holding_psd/`.

### Main settings

Every notebook starts with a Settings cell. The most relevant values are:

| Setting | Value | Meaning |
| --- | --- | --- |
| `RUNS` | R02 to R05 | Runs included |
| `L_FREQ`, `H_FREQ` | 0.1, 40 Hz | Band pass filter |
| `DECIM` | 2 (fixation), 3 (holding) | Decimation factor, keeps Nyquist above 40 Hz |
| `Z_THRESH_BAD` | 5.0 | Bad channel threshold |
| `N_PERMUTATIONS` | 1000 | Permutations for cluster tests |
| `SEED` | 97 | Random seed for ICA, AutoReject and permutations |
| `OCCIPITAL_ROI` | O1, O2, PO7, PO8, PO3, PO4 | Fixation ERP |
| `CENTRAL_ROI` | Cz, C3, C4, CPz, CP1, CP2, FC1, FC2 | Holding ERP and PSD |

## Author

Alberto Martinez Hernandez, NTLab (Laboratorio de Neurotecnología e Interfaces Cerebro Computador), Tecnológico de Monterrey, Campus Guadalajara.