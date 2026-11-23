# Event-time phase coherence for bearing fault detection: code and run records

This deposit holds the code and the per-record results behind the paper "Bearing Fault Detection by Event-Time Phase Coherence with a Shaft-Harmonic Veto and a Geometry Margin Rule" by Nermen Salama, A. M. Bassiuny, A. S. Tolba and Rowida Meligy (submitted to Scientific Reports). With it, a reader can rerun every number in the paper from the public datasets, or check every table against our run files without running anything.

The detector in the paper is version A15 (`code/acquisition_batch_06.py`). Every development, noise and held-out number in the paper comes from A15 and from the comparator described in the Methods. Each script checks the SHA-256 hashes of the files it imports and stops if any of them differs from the list below.

## What is in the deposit

```
README.md            this file
LICENSE
CITATION.cff
requirements.txt
code/                detector, comparator, scoring, and one script per analysis
manifests/           the exact record list for each dataset, in run order
results/             per-record run files (CSV) and the summary written by each run
supplementary/       the Smith and Randall grade table used for Supplementary Table S1
```

The raw vibration data are not included. They are public, each under its own terms, and the next section says where to get them.

## Datasets

| Dataset | Source | Records used |
|---|---|---|
| CWRU | Case Western Reserve University Bearing Data Center, https://engineering.case.edu/bearingdatacenter | 36 drive-end 12 kHz fault records (105–108, 118–121, 130–133, 169–172, 185–188, 197–200, 209–212, 222–225, 234–237) and the 4 normal baseline records 97–100 |
| Paderborn | Lessmeier et al. 2016, doi:10.36001/phme.2016.v3i1.1577 | K001–K006, KA01, KA03–KA09, KA15, KA16, KA22, KA30, KI01, KI03–KI05, KI07, KI08, KI14, KI16–KI18, KI21; operating conditions M01 and M07, first repetition (58 records) |
| MFPT | Society for Machinery Failure Prevention Technology bearing fault data set | 3 baseline and 17 fault records (7 inner race, 10 outer race) |
| XJTU-SY | Wang et al. 2020, doi:10.1109/TR.2018.2882682, data at https://biaowang.tech/xjtu-sy-bearing-datasets | first three and last three snapshots of 13 bearings, horizontal channel (78 records); bearings 1_4 and 2_3 are left out because their only failed element is the cage |
| University of Ottawa (held out) | Sehri et al. 2023, doi:10.1016/j.dib.2023.109327; data doi:10.17632/y2px5tg92h.5 | healthy state of all 20 bearings, developing and faulty states of the inner-race, outer-race and ball bearings (50 files) |

The MFPT page cited in the paper (https://www.mfpt.org/fault-data-sets) redirected to a page without the data in November 2026. The file names we used are listed in `manifests/mfpt_manifest.csv`, so the same records can be found in any copy of the data set.

The CWRU normal records 97–100 exist only at 48 kHz. The loader decimates them by four to 12 kHz to match the fault records. Paderborn files must be extracted from their archives first, and the Ottawa files are read as `.mat`.

## Software

The runs used Python 3.13.18, NumPy 2.4.6 and SciPy 1.16.3. A few synthetic runs used NumPy 2.5.3 and SciPy 1.18.1, and their summaries say so. `requirements.txt` pins the first set. No GPU is needed. Most runs were made in Google Colab with two worker processes. Run times are printed in each summary. One clean detector pass over the 196 development records takes about an hour at two workers.

## How to run

Put the extracted datasets in one folder per dataset. The scripts read the folder locations from command-line options, and their defaults are the Google Drive paths we used:

```
--work     folder with the code (default /content)
--cwru     CWRU .mat files
--pad      extracted Paderborn .mat files
--mfpt     MFPT .mat files
--xjtu     XJTU-SY folders Bearing1_1 ... Bearing3_5
--ottawa   Ottawa .mat files
--lab      folder holding earlier run outputs that a script reads
--out      output folder
--workers  number of processes
```

These options are defined once in `code/t3_common.py` and are shared by every `t3`–`t11` script. The older scripts (`c1_classical_noveto.py`, `s4_noise_sweep.py`, `s5_paired_tests.py`, `h2_ottawa_rerun.py`) have their own options, shown by `--help`.

Each script first checks the hashes, then runs a harness that must reproduce earlier rows exactly, and only then runs the analysis. Runs can be resumed, because rows already written to the output CSV are skipped. A script started with `--summary-only` rebuilds its summary from the CSV without running the detector.

To check one record by hand:

```python
import numpy as np, sys
sys.path.insert(0, 'code')
import acquisition_batch_06 as A15
x, fs, rpm = A15.LOADERS['CWRU']('CWRU_data/118.mat')
r = A15.classify_record(x, fs, 1797, 'CWRU_6205', verbose=True,
                        rng=np.random.default_rng(17), rpm_quality='nominal')
```

This prints the log shown in Supplementary Note S2 for CWRU 118 (seed 1, record index 16, so the generator seed is 1 + 16).

## Protocol facts a reader needs

- Seeds. Record i under seed s uses `numpy.random.default_rng(s + i)`, with i the record's position in its manifest. Seed 1 is the primary run and seed 2 the replication. The five independent seed streams (`t3b_seeds.py`) use `default_rng([s, i])` for s = 1 to 5, so there are seven runs per record in all.
- Speed. CWRU uses the load-table speed, Paderborn and Ottawa the recorded speed (search window ±0.2%), and MFPT and XJTU-SY the nominal speed (±1%).
- Spectral fallback. A15 contains a spectral fallback stage that is switched off (`SPECTRAL_FALLBACK = False`) in every run reported in the paper.
- Comparator. `classical_baseline_03.py` is run through `c1_classical_noveto.py`, which removes only its quarter-order shaft veto. The variants in Supplementary Table S14 (ball at 2×BSF, half-integer shaft veto, cepstrum pre-whitening) are runtime changes made in `t7a`, `t8m1` and `t11c`, and each script prints the change it makes.
- Scoring. `score_rules.py` scores each record as true negative, false alarm, miss, wrong class or correct class. A detection is any fault call on a faulty record.
- Tests. `s5_paired_tests.py` holds the exact McNemar test, the Holm adjustment and the false-alarm matching rule. The finer 0.1 dB matching used in Supplementary Table S14 is in `t8m1_vetoed_comparator.py`.

## Where each result comes from

| Paper item | Script | Main output |
|---|---|---|
| Development results, Table 2 clean rows, Fig. 2, Supplementary Table S8 | `t4_a15_verify.py`, `c1_classical_noveto.py`, `s5_paired_tests.py` | `results/t4_out/t4_runs.csv`, `results/c1_out/c1_runs.csv` |
| Table 2 noise rows, Supplementary Fig. S1 and Table S9 | `s4_noise_sweep.py`, `t9_noise_a15.py` (shows A15 gives the same verdict on all 1,960 noisy runs) | `results/t9_out/t9_runs.csv` |
| Five independent seed streams | `t3b_seeds.py` | `results/t3_out/t3b_runs.csv` |
| Paderborn ten noise seeds | `t3d_noise_seeds.py` | `results/t3_out/t3d_runs.csv` |
| Null-grid check, Supplementary Table S2 | `t5_null_grid.py` | `results/t5_out/` |
| Synthetic records, Fig. 3, Supplementary Table S10 | `t6b_synthetic.py`, `t11b_synthetic_c1v.py`, `t11c_cpw_comparator.py` | `results/t6_out/t6b_runs.csv`, `results/t11_out/` |
| Veto reach, Supplementary Table S15 | `t10_delta_r.py` | `results/t10_out/` |
| Supplied speed error, Fig. 5 | `t6d_speed_error.py` | `results/t6_out/t6d_runs.csv` |
| Ottawa, Fig. 6, Supplementary Tables S6 and S11 | `h2_ottawa_rerun.py`, `t4b_ottawa_a15.py`, `t3a_equal_prior.py` | `results/t4_out/t4_ottawa.csv` |
| Exploratory envelope test on Ottawa, Supplementary Table S7 | `e1b_ottawa_envelope.py` | `results/e1b_out/` |
| Confusion matrices and per-class false alarms, Supplementary Tables S4, S5, S12 | `t6e_desk.py`, `t11e_diagnostic.py` | `results/t6_out/t6e_tables.txt`, `results/t11_out/t11e_f1.csv` |
| Detector logs, Supplementary Note S2 | `t11d_wrong_logs.py` and the T4 harness | `results/t11_out/t11d_logs.txt` |
| Sensitivity runs, Supplementary Table S14 | `t7a_c1_ball2x.py`, `t7b_null1000.py`, `t8_comp_record.py`, `t8m1_vetoed_comparator.py`, `t11a_spaced_null.py`, `t11c_cpw_comparator.py`, `t11e_diagnostic.py` | `results/t7_out/`, `results/t8_out/t8_comp_runs.csv`, `results/t11_out/` |
| Smith and Randall comparison, Supplementary Table S1 and Fig. S2 | built from `t4_runs.csv` and the grades in Smith and Randall (2015), Table B2 | `supplementary/smith_randall_vs_a15.md` |

Table 1 uses equation (1) of the paper with the geometry entries in `GEOMETRY` in `code/acquisition_batch_06.py`.

## Code files and hashes

The hashes are the first 12 hexadecimal characters of SHA-256. Supplementary Note S1 of the paper lists the same values.

| File | Role | Hash |
|---|---|---|
| `acquisition_batch_06.py` | detector A15 | 504e43976b69 |
| `classical_baseline_03.py` | comparator | d69e9ce56f70 |
| `c1_classical_noveto.py` | runs the comparator without its quarter-order veto | 0670f27eb3e3 |
| `score_rules.py` | scoring | 275e7eb10621 |
| `s5_paired_tests.py` | paired tests | a99e3415ce38 |
| `colab_runner_03_02.py` | runner and CWRU/Paderborn manifests | 1dcd0fc53ad7 |
| `s4_noise_sweep.py` | noise sweep | 4c8b380f40b9 |
| `t3a_equal_prior.py` | comparator speed window | e3aebbedf600 |
| `t3b_seeds.py` | independent seed streams | 1ff831f648e6 |
| `t3c_lattice_ablation.py` | lattice stage as a positional veto | 926f2849e9ba |
| `t3d_noise_seeds.py` | Paderborn noise seeds | b9285626a02a |
| `t4_a15_verify.py` | A15 development verdicts | 68dfb70dd4f3 |
| `t4b_ottawa_a15.py` | A15 Ottawa configurations | 2944020709e7 |
| `t5_null_grid.py` | null-grid check | df465bda67b5 |
| `t6b_synthetic.py` | synthetic records | 9507a4ec83be |
| `t6d_speed_error.py` | speed-error sweep | f83c03ec8fd0 |
| `t6e_desk.py` | confusion matrices | 40500c50554e |
| `e1b_ottawa_envelope.py` | Ottawa envelope test | 00a016c118c9 |
| `t7a_c1_ball2x.py` | comparator ball at 2×BSF | 34777361c6a0 |
| `t7b_null1000.py` | detector with 1,000 null draws | 7d0b7a3b7ca9 |
| `t8_comp_record.py` | comparator recording run | 0893b1ccf2c5 |
| `t8m1_vetoed_comparator.py` | comparator with the shaft veto | f30793377bbd |
| `t9_noise_a15.py` | A15 on the noise rows | e1026b44f298 |
| `t10_delta_r.py` | veto-reach sweep | 22ec598d754a |
| `t11a_spaced_null.py` | null with minimum event spacing | 1e8a1b175fb0 |
| `t11b_synthetic_c1v.py` | synthetic records, comparator with veto | 972e233d4455 |
| `t11c_cpw_comparator.py` | pre-whitened comparator | 9d07dc4da9fb |
| `t11d_wrong_logs.py` | wrong-decision logs | 5054ad6dc1eb |
| `t11e_diagnostic.py` | diagnostic metrics | caa6e27b01db |

These files are needed only because the scripts import them for loading and manifests:

| File | Role | Hash |
|---|---|---|
| `acquisition_batch_05.py` | A14, from which A15 was derived; the shared harness loads it and swaps in A15 for the detector call | a7e9834c6286 |
| `t3_common.py` | shared harness | 46bc8956f8a1 |
| `s1b_a13_protocol.py` | CWRU and Paderborn manifest | f4d44a81fb45 |
| `dataset_extensions_02.py` | MFPT and XJTU-SY loaders and manifests | 3e161f41be41 |
| `h1_ottawa_heldout.py` | Ottawa manifest and sampling rate | 80181c034b82 |
| `h2_ottawa_rerun.py` | Ottawa loader with the recorded speed | c4b7a816f566 |

`t4_a15_verify.py` shows that A14 and A15 give the same verdict on all 392 development runs, and `t4b_ottawa_a15.py` shows they differ on one Ottawa run (H_19_0, seed 2, where A15's veto removes a false alarm). The A14 file is in the deposit for that check and for the shared loaders. The paper reports A15 only.

## Result files

Each detector row has the record key, the true label, the seed, the refined shaft frequency `fr`, the event verdict `pred_event`, and for each class (BPFO, BPFI, Ball) the statistic `z`, its threshold `z_thr`, the offset `off` of the refined peak in percent, the lattice flag, and whether the class passed. Comparator rows have the comb SNR and offset of each class and the verdicts at 4, 6 and 8 dB. Some of these files carry their own hash, printed in the summaries that read them:

| File | Hash |
|---|---|
| `t4_runs.csv` (A15, development, seeds 1 and 2) | e08e7121b575 |
| `s1b_runs_L2_s1.csv`, `s1b_runs_L2_s2.csv` (A14 clean runs read by `s5_paired_tests.py`) | printed in the S5 summary |
| `t4_ottawa.csv` (A15, Ottawa) | 16a69f0cf2a5 |
| `t9_runs.csv` (A15, noise rows) | b7fcd33e5ce9 |
| `t8_comp_runs.csv` (comparator recording run) | 2c06ec9628da |
| `t6b_runs.csv` (synthetic records) | fb0061fcb181 |

## Quick check

The seed-1 clean row of Table 2 should come out as 105 of 138 faulty records detected by A15 against 70 for the comparator at its matched 8 dB threshold, with discordant pairs 39 and 4 and 7 against 8 healthy false alarms. `s5_paired_tests.py` prints it from the clean run files `s1b_runs_L2_s1.csv` and `s1b_runs_L2_s2.csv` (made with A14, which `t4_a15_verify.py` shows to give the same verdicts as A15) and `c1_runs.csv`. `t8m1_vetoed_comparator.py` reproduces the same row from `t4_runs.csv` as its first check.

## License and citation

The code is released under the license in `LICENSE`. If you use it, please cite the paper and this deposit (`CITATION.cff`).

Corresponding author: Nermen Salama, nermen.salama@h-eng.helwan.edu.eg

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24462231.svg)](https://doi.org/10.5281/zenodo.24462231)


