# Similarity Measures for Volume Registration

Also called: Correlation Ratio, Entropy Correlation Coefficient, Mutual Information, Normalized Mutual Information, Tsallis Mutual Information

## Overview
This family defines quantitative metrics to evaluate the alignment between a floating (transformed) volume and a reference (target) volume by analyzing the statistical dependence of their intensity values. It solves the problem of determining registration quality in monomodal 3D/3D scenarios, such as renal volume registration, where intensity distributions are similar but spatial alignment varies. What sets this family apart from sibling methods like [Correlation](52-name.md) is its reliance on joint probability distributions and entropy-based formulations rather than linear intensity assumptions, making it robust for non-linear intensity relationships.

## Shared principles
All measures in this family are computed over the overlapping spatial domain $\Omega = \Omega_1 \cap \Omega_2$ of the two volumes $X$ and $Y$. The core mechanism involves constructing a 2D joint histogram and deriving probability distributions from it.

1.  **Joint Histogram**: Compute $h_{XY}(i, j)$, the count of voxel pairs in $X$ and $Y$ having intensity combinations $(i, j)$, where $i \in [0, L_x-1]$ and $j \in [0, L_y-1]$ are the discrete intensity levels.
2.  **Joint Probability**: Normalize the histogram to obtain the joint probability distribution:
    $$ p_{XY}(i, j) = \frac{h_{XY}(i, j)}{\sum_{i,j} h_{XY}(i, j)} $$
3.  **Marginal Probabilities**: Derive marginal distributions by summing the joint probability over the respective dimensions:
    $$ p_X(i) = \sum_j p_{XY}(i, j) $$
    $$ p_Y(j) = \sum_i p_{XY}(i, j) $$
4.  **Shannon Entropy**: Calculate entropy for any distribution $p$ using:
    $$ H(p) = -\sum_k p(k) \log(p(k)) $$
    Specifically, $H(X) = H(p_X)$, $H(Y) = H(p_Y)$, and the joint entropy $H(X, Y) = H(p_{XY})$.

## Subtopics

### Mutual Information (MI)
MI measures the reduction in uncertainty about one image given knowledge of the other. It is defined as:
$$ MI(X, Y) = H(X) + H(Y) - H(X, Y) $$
MI is the primary metric used in the parent method [Rigid Registration via Mutual Information Maximization](21006-rigid-registration-via-mutual-information-maximization.md). It is chosen when maximizing statistical dependence is required for monomodal data.

### Entropy Correlation Coefficient (ECC)
ECC normalizes MI by the geometric mean of the individual entropies:
$$ ECC(X, Y) = \frac{MI(X, Y)}{\sqrt{H(X)H(Y)}} $$
ECC is chosen when a normalized measure bounded between 0 and 1 is preferred to compare alignment quality across different datasets.

### Normalized Mutual Information (NMI)
NMI normalizes MI by the sum of the individual entropies:
$$ NMI(X, Y) = \frac{H(X) + H(Y)}{H(X, Y)} $$
NMI is often preferred over ECC in medical imaging because it is less sensitive to the size of the overlapping region and provides a stable baseline for optimization.

### Correlation Ratio
The correlation ratio $\eta^2$ measures the ratio of the variance of the conditional means to the total variance. It is defined as:
$$ \eta^2(X, Y) = \frac{\sum_i p_X(i) (E[Y|X=i] - E[Y])^2}{Var(Y)} $$
where $E[Y|X=i]$ is the expected intensity in $Y$ for voxels with intensity $i$ in $X$. It is chosen when the relationship between intensities is assumed to be deterministic (functional) rather than stochastic.

### Tsallis Mutual Information
This variant uses Tsallis entropy $H_q(p) = \frac{1}{q-1}(1 - \sum_k p(k)^q)$ instead of Shannon entropy. The specific formulation depends on the parameter $q$. It is chosen when heavy-tailed intensity distributions or non-extensive statistical mechanics models are appropriate for the data.

**Comparison on Real Data** [S1, unofficial]:
| Metric | DO | CR | NOM | RON |
| :--- | :--- | :--- | :--- | :--- |
| MI | 0.006139 | 26.083168 | 0 | 509.01 |
| NMI | 0.005268 | 26.083168 | 0 | 137.99 |
| ECC | 0.006133 | 26.083168 | 0 | 588.60 |

MI and NMI showed small advantages in distinctiveness (DO) and capture range (CR) compared to ECC, Correlation, and Tsallis. All measures had 0 Accuracy (ACC) and 0 Number of Local Maxima (NOM) on synthetic data.

## Background
This family was implemented and evaluated in the context of a kidney preoperative planning system for rigid registration of renal volumes [S1, unofficial]. The work focuses on monomodal intra-subject registration where the organ is assumed rigid during the acquisition window. The metrics were selected to optimize registration parameters using a Downhill Simplex optimizer.

## Sources
[S1] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
