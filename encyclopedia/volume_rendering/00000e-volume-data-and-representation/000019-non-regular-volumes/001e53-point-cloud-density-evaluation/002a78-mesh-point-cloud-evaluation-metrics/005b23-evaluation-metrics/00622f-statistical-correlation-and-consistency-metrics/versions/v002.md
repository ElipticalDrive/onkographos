# Statistical Correlation and Consistency Metrics

Also called: PCC, SROCC, RMSE

## Overview
Computes statistical measures to assess the linearity, monotonicity, accuracy, and consistency of objective metric predictions against subjective Mean Opinion Scores (MOS) [S1].

## How it works
**Inputs:**
- $MOS$: Vector of subjective Mean Opinion Scores for $N$ samples [S1].
- $MOS_p$: Vector of predicted objective metric scores for the same $N$ samples [S1].
- $x$: Objective metric score (PQR) used for regression [S1].

**Outputs:**
- PCC: Pearson linear correlation coefficient.
- SROCC: Spearman rank order correlation coefficient.
- RMSE: Root-mean-square error.
- OR: Outlier ratio.

**Data Structures:**
- Vectors $MOS$ and $MOS_p$ of size $N$, containing floating-point values [S1].

**Steps:**
1. **Regression (Optional for Monotonicity):** Estimate predicted MOS values from objective Point cloud Quality Rating (PQR) scores using monotonic cubic regression [S1]. Determine coefficients $a, b, c, d$ via least squares [S1].
   $$ MOS_p = a \cdot x^3 + b \cdot x^2 + c \cdot x + d $$
   where $x$ is the objective metric score (PQR) [S1].

2. **Compute PCC:** Calculate the Pearson linear correlation coefficient between $MOS$ and $MOS_p$ to assess linearity [S1].
   $$ PCC = \frac{\sum_{i=1}^{N} (MOS_i - \overline{MOS})(MOS_{p,i} - \overline{MOS_p})}{\sqrt{\sum_{i=1}^{N} (MOS_i - \overline{MOS})^2} \sqrt{\sum_{i=1}^{N} (MOS_{p,i} - \overline{MOS_p})^2}} $$
   where $\overline{MOS}$ and $\overline{MOS_p}$ are the means of the respective vectors [S1].

3. **Compute OR:** Calculate the outlier ratio based on standard error between $MOS$ and $MOS_p$ to assess consistency [S1].

4. **Compute SROCC:** Calculate the Spearman rank order correlation coefficient to assess monotonicity [S1].

5. **Compute RMSE:** Calculate the root-mean-square error to assess accuracy [S1].
   $$ RMSE = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (MOS_i - MOS_{p,i})^2} $$
   Note: For 3D model evaluation against a reference, RMSE is computed over spatial coordinates [S2]:
   $$ Total\ Error = \sqrt{\frac{1}{n}\sum_{i=1}^{n} \left[ (x_{i,est} - x_i)^2 + (y_{i,est} - y_i)^2 + (z_{i,est} - z_i)^2 \right]} $$
   where $x_i, y_i, z_i$ are input/reference coordinates and $(x_{i,est}, y_{i,est}, z_{i,est})$ are estimated coordinates for the $i$-th camera position [S2].

**Parameters:**
- $N$: Total number of samples or camera positions used in the evaluation [S1, S2].
- $\alpha$: Significance level for confidence intervals, fixed at $0.05$ [S1].

**Edge Cases and Pitfalls:**
- Division by zero in PCC if the variance of $MOS$ or $MOS_p$ is zero [G].
- Degenerate input where all predicted scores are identical [G].

## Subtopics

### [Normalized Mutual Information (NMI)](19094-normalized-mutual-information-nmi.md)
Normalized Mutual Information (NMI) is a statistical metric used to evaluate the correlation between two random variables by quantifying the amount of shared information, capturing both linear and non-linear dependencies. In the context of volume rendering and data evaluation, it serves as a distance measure between volumes (e.g., test vs. reference) or as a diagnostic tool to assess attribute independence in compressed representations. It sets itself apart from simple correlation coefficients by being sensitive to non-linear relationships and by normalizing the mutual information against the individual entropies of the variables, ensuring a bounded result.

### Dice Coefficient
Computes overlap statistics to measure similarity between two sets [S3, unofficial]. Formula: $Dice = \frac{2|A \cap B|}{|A| + |B|} = \frac{2 \cdot shared}{n_a + n_b}$ [S3, unofficial]. It is equivalent to $\frac{2 \cdot IoU}{1 + IoU}$ [S3, unofficial]. Dice is the segmentation-benchmark convention, while IoU is the detection convention; they are monotone in overlap and rank pairs identically [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Developed for perceptual quality assessment of voxelized point clouds [S1]. The methodology follows Recommendation ITU-T P.1401 for computing PCC, SROCC, RMSE, and OR [S1].

## Sources
[S1] A novel methodology for quality assessment of voxelized point clouds, https://doi.org/10.1117/12.2322741, fetched 2026-10-30
[S2] Assessment of 3D Model for Photogrammetric Purposes Using AI Tools Based on NeRF Algorithm, https://doi.org/10.3390/heritage6080301, fetched 2026-10-28
[S3] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
