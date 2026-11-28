# 2D Metrics for Rendered 3D Quality Assessment

Results accompanying **Evaluating 2D Quality Metrics on Videos Rendered from Compressed 3D Scene Representations**, by Chaojun Hu, Jiayi Wang, and Lu Yu (Zhejiang University). Research manuscript; publication status is not implied. Results snapshot: 22 October 2026.

## Scope

27 metric outputs, including all three UniPercept outputs. Primary compression databases: Explicit-NeRF-QA (336 videos, 21 groups) and GScomp-QA (305 videos, 13 scenes). Applicability boundaries: MCL-3D (648 left-view images paired with stereoscopic scores, 9 contents), MUGSQA (2,413 videos, 55 objects), NeRF-QA (48 videos, 8 scenes), and NVS-QA (76/88 videos, 16 scenes).

**NVS-QA coverage:** 25 outputs have 76 scores; FDIM has 27 scores and 7 valid within-content groups; VMAF is unavailable. FDIM must not be ranked against the complete-coverage results. The 12 excluded videos are Mip-NeRF 360 outputs. NeRF-QA and NVS-QA share 44 distorted-video filenames; these are not independent replications.

## Downloads and tables

- [All six-database correlations](all_metric_correlations.csv): unrounded SROCC, direct PLCC, median within-content SROCC/PLCC, counts.
- [Complete result bundle](results.zip): every released CSV plus protocol.
- [Pair accuracy](pair_accuracy.csv), [gap-bin accuracy](gap_bin_accuracy.csv), [gap-standardized comparisons](gap_standardized_accuracy.csv).
- [Matched reconstruction conditions](matched_summary.csv), [matched pairs](matched_pairs.csv), [matched robustness](matched_robustness.csv).
- [Content-level correlations](sequence_correlations.csv), [content-level pair accuracy](within_pair_by_object.csv).
- [SROCC bootstrap intervals](bootstrap_srocc.csv), [paired differences](bootstrap_differences.csv), [pair-accuracy intervals](object_accuracy_ci.csv), [small-gap intervals](mugs_gap_ci.csv).
- [Protocol and input hashes](protocol.json). Other CSV files retain detailed content-level outputs and diagnostic candidates.

## Interpretation

Higher scores denote better quality. LPIPS, DISTS, GMSD, NIQE, BRISQUE, ILNIQE and PI use negated distance/damage scores. NeRF-QA DMOS is negated; GScomp-QA DMOS is already quality-oriented; NVS-QA uses MOS_synth − MOS_ref + 5. PLCC is direct, with no logistic fitting. D denotes pooled correlation; S denotes the median within-content correlation over nonconstant groups of at least three samples. No cross-database mean is reported.

Pair analyses cover the five complete databases (3,750 samples), excluding NVS-QA. Subjective ties are excluded and objective ties receive half credit. Within-content accuracy equally weights contents; cross-content accuracy equally weights content combinations. Gaps are normalized by each database's observed subjective-score range; they are not common perceptual or JND thresholds. The 2,000-replicate content bootstrap is exploratory, without multiplicity adjustment or participant resampling. Matched MUGSQA comparisons fix object, method, and the other three recorded input factors; they do not establish causal effects.

The `sequence_correlations.csv` file contains SROCC only; both median within-content correlation types are available in the six-database summary. Bootstrap intervals cover ten focused outputs; full point estimates retain all 27 outputs. Diagnostic candidates were screened using the original selected metrics, and are not a failure-rate estimate or visual artifact attribution. Source videos were unavailable. Absolute workstation paths have been replaced by sample filenames in this release. Missing run metadata prevents a verified zero-shot claim.

## Database sources

- [Explicit-NeRF-QA](https://arxiv.org/abs/2407.08165)
- [GScomp-QA](https://arxiv.org/abs/2605.26880)
- [MCL-3D](https://arxiv.org/abs/1405.1403)
- [MUGSQA](https://arxiv.org/abs/2511.06830)
- [NeRF-QA](https://arxiv.org/abs/2305.03176)
- [NVS-QA](https://arxiv.org/abs/2405.20078)

This repository distributes derived evaluation results, not source videos, individual participant ratings, model weights, or MPEG proposals. Refer to the original database providers for their materials and terms. This archive supports independent inspection of the results; it is not a conference supplementary submission.

## Six-database results

Each entry below is **pooled SROCC / direct PLCC**. For counts and within-content results use the downloadable CSV. NVS-QA FDIM is marked partial.

| Metric | Explicit-NeRF-QA | GScomp-QA | MCL-3D | MUGSQA | NeRF-QA | NVS-QA |
|---|---:|---:|---:|---:|---:|---:|
| PSNR | 0.871 / 0.847 | 0.619 / 0.619 | 0.831 / 0.802 | 0.620 / 0.625 | 0.585 / 0.581 | 0.690 / 0.647 |
| SSIM | 0.830 / 0.795 | 0.636 / 0.613 | 0.780 / 0.572 | 0.309 / 0.303 | 0.455 / 0.295 | 0.547 / 0.415 |
| MS-SSIM | 0.902 / 0.775 | 0.713 / 0.677 | 0.872 / 0.707 | 0.579 / 0.525 | 0.513 / 0.333 | 0.673 / 0.521 |
| FSIM | 0.876 / 0.760 | 0.771 / 0.757 | 0.929 / 0.861 | 0.619 / 0.578 | 0.480 / 0.315 | 0.672 / 0.518 |
| GMSD | 0.921 / 0.905 | 0.719 / 0.720 | 0.886 / 0.858 | 0.522 / 0.486 | 0.572 / 0.527 | 0.670 / 0.598 |
| VSI | 0.921 / 0.807 | 0.761 / 0.717 | 0.919 / 0.822 | 0.688 / 0.645 | 0.636 / 0.527 | 0.697 / 0.526 |
| VMAF | 0.917 / 0.918 | 0.799 / 0.810 | 0.847 / 0.822 | 0.430 / 0.396 | 0.619 / 0.610 | — |
| LPIPS | 0.900 / 0.757 | 0.811 / 0.771 | 0.829 / 0.761 | 0.420 / 0.365 | 0.472 / 0.376 | 0.718 / 0.644 |
| DISTS | 0.917 / 0.888 | 0.860 / 0.854 | 0.849 / 0.833 | 0.510 / 0.477 | 0.720 / 0.683 | 0.839 / 0.814 |
| TOPIQ-FR | 0.934 / 0.898 | 0.842 / 0.856 | 0.933 / 0.889 | 0.601 / 0.567 | 0.702 / 0.721 | 0.802 / 0.793 |
| FDIM | 0.944 / 0.928 | 0.827 / 0.824 | 0.897 / 0.883 | 0.556 / 0.498 | 0.594 / 0.617 | 0.841 / 0.852 (n=27) |
| NIQE | -0.112 / -0.084 | 0.285 / 0.308 | 0.603 / 0.663 | 0.052 / -0.043 | 0.271 / 0.261 | 0.054 / 0.118 |
| BRISQUE | 0.180 / 0.179 | 0.172 / 0.201 | 0.647 / 0.662 | 0.105 / 0.083 | 0.066 / 0.063 | -0.242 / -0.181 |
| ILNIQE | -0.049 / -0.029 | 0.182 / 0.220 | 0.640 / 0.686 | 0.126 / 0.080 | -0.009 / -0.026 | -0.118 / -0.021 |
| PI | -0.032 / -0.004 | 0.246 / 0.269 | 0.523 / 0.578 | 0.112 / 0.034 | 0.403 / 0.391 | 0.315 / 0.371 |
| HyperIQA | 0.401 / 0.382 | 0.533 / 0.515 | 0.211 / 0.178 | 0.109 / 0.086 | 0.382 / 0.336 | 0.663 / 0.652 |
| DBCNN | 0.080 / 0.090 | 0.481 / 0.505 | 0.139 / 0.147 | 0.205 / 0.143 | 0.258 / 0.258 | 0.637 / 0.670 |
| PaQ-2-PiQ | 0.598 / 0.566 | 0.400 / 0.429 | 0.415 / 0.491 | 0.359 / 0.316 | 0.494 / 0.429 | 0.494 / 0.613 |
| NIMA | 0.113 / 0.090 | 0.179 / 0.231 | 0.527 / 0.542 | 0.206 / 0.178 | 0.163 / 0.148 | 0.466 / 0.444 |
| MANIQA | 0.397 / 0.340 | 0.621 / 0.599 | 0.130 / 0.046 | 0.144 / 0.120 | 0.425 / 0.402 | 0.750 / 0.726 |
| MUSIQ | 0.252 / 0.282 | 0.597 / 0.641 | 0.480 / 0.476 | 0.360 / 0.322 | 0.527 / 0.509 | 0.756 / 0.778 |
| LIQE | 0.317 / 0.260 | 0.755 / 0.714 | 0.377 / 0.292 | 0.154 / 0.119 | 0.566 / 0.503 | 0.680 / 0.678 |
| CLIP-IQA+ | 0.585 / 0.565 | 0.503 / 0.521 | 0.519 / 0.513 | 0.314 / 0.281 | 0.478 / 0.485 | 0.594 / 0.698 |
| Q-Align | 0.460 / 0.449 | 0.748 / 0.743 | 0.747 / 0.748 | 0.300 / 0.230 | 0.830 / 0.703 | 0.625 / 0.694 |
| UniPercept-IAA | 0.038 / 0.082 | 0.641 / 0.645 | 0.563 / 0.553 | 0.341 / 0.315 | 0.688 / 0.684 | 0.683 / 0.688 |
| UniPercept-IQA | 0.460 / 0.442 | 0.758 / 0.774 | 0.628 / 0.621 | 0.383 / 0.346 | 0.513 / 0.490 | 0.635 / 0.717 |
| UniPercept-ISTA | -0.016 / -0.036 | 0.256 / 0.333 | 0.610 / 0.633 | 0.163 / 0.167 | -0.039 / 0.038 | 0.196 / 0.349 |
