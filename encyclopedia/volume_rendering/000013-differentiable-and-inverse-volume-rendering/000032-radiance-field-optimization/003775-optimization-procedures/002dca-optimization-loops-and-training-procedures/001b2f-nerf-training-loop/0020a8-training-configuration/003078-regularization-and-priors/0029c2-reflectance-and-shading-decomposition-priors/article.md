# Reflectance and Shading Decomposition Priors

Also called: Intrinsic Priors, Mean Reflectance Regularization, Shading Smoothness

## Overview
Reflectance and Shading Decomposition Priors constrain the learned radiance field by decoupling material properties (reflectance) from illumination effects (shading) [S1]. This method addresses the scale ambiguity inherent in intrinsic image decomposition, where scaling reflectance and inversely scaling shading produces identical observed colors, by applying regularization constraints that stabilize the separation [S1].

## How it works
**Inputs and Outputs:** The method takes sampled points along rays $\mathbf{r}$, their associated surface normals, and 3D coordinates as input, and outputs decomposed reflectance $\mathbf{R}(\mathbf{r})$ and shading $\mathbf{S}(\mathbf{r})$ fields for optimization [S1].

**Data Structures:** The neural network predicts two 3-dimensional RGB vectors per sampled point: reflectance $\mathbf{R}(\mathbf{r})$ representing intrinsic material color, and shading $\mathbf{S}(\mathbf{r})$ representing illumination brightness [S1].

**Numbered Steps:**
1. Sample points along rays and predict $\mathbf{R}(\mathbf{r})$, $\mathbf{S}(\mathbf{r})$, and surface normals [S1].
2. Compute the photometric reconstruction loss $\mathcal{L}$ and optional depth loss $\mathcal{L}_d$ [S1].
3. Compute the mean reflectance regularization $\mathcal{L}_{mean,r}$ by comparing the maximum RGB channel value of $\mathbf{R}(\mathbf{r})$ to $\beta_{avg,r}$ [S1].
4. Compute the shading smoothness loss $\mathcal{L}_s$ as the variance of $\mathbf{S}(\mathbf{r})$ across the batch [S1].
5. Compute the shading-normal consistency loss $\mathcal{L}_{sn}$ by weighting the absolute shading difference between point pairs by their normal cosine similarity and distance [S1].
6. Sum all weighted losses to update network parameters [S1].

**Formulas:**
The total training loss is defined as a weighted sum of eight terms [S1]:
$$
\mathcal{L}_{train} = \lambda_0 \mathcal{L} + \lambda_1 \mathcal{L}_d + \lambda_2 \mathcal{L}_{mean,r} + \lambda_3 \mathcal{L}_{chro} + \lambda_4 \mathcal{L}_{rs} + \lambda_5 \mathcal{L}_{nrs} + \lambda_6 \mathcal{L}_s + \lambda_7 \mathcal{L}_{sn}
$$
where all $\lambda_i$ are adjustable weights [S1]. The explicitly defined priors are:

1. **Mean Reflectance Regularization ($\mathcal{L}_{mean,r}$):** Constrains the maximum reflectance value across RGB channels to a target average brightness $\beta_{avg,r}$ to prevent unreasonable brightness scaling [S1].
$$
\mathcal{L}_{mean,r} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left| (\mathbf{R}(\mathbf{r}).max) - \beta_{avg,r} \right|
$$
where $\mathbf{R}(\mathbf{r}).max = \max(R_r, R_g, R_b)$ for the reflectance vector at ray $\mathbf{r}$ [S1].

2. **Shading Smoothness ($\mathcal{L}_s$):** Defined as the variance of shading values for all pixels in a single batch, encouraging spatially smooth illumination [S1].
$$
\mathcal{L}_s = \text{Var}(\{\mathbf{S}(\mathbf{x}) \mid \mathbf{x} \in \text{Batch}\})
$$

3. **Shading-Normal Consistency ($\mathcal{L}_{sn}$):** Enforces consistency between shading and surface normals by penalizing large shading differences between nearby points with similar orientations [S1].
$$
\mathcal{L}_{sn} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \omega_n(\mathbf{r}, \mathbf{r}') \cdot |\mathbf{S}(\mathbf{r}) - \mathbf{S}(\mathbf{r}')| \cdot (1 - \tanh(\text{Dis}(\mathbf{r}, \mathbf{r}')))
$$
where $\omega_n(\mathbf{r}, \mathbf{r}')$ measures the cosine similarity of the surface normals at $\mathbf{r}$ and $\mathbf{r}'$ [G], and $\text{Dis}(\mathbf{r}, \mathbf{r}')$ is the Euclidean distance between their respective 3D points [S1].

4. **Other Priors:** The source lists $\mathcal{L}_{chro}$ (reflectance chromaticities), $\mathcal{L}_{rs}$ (sparsity), and $\mathcal{L}_{nrs}$ (non-local sparsity) as common intrinsic image decomposition constraints used in the optimization, but does not provide their explicit mathematical definitions [S1].

**Parameters:**
* $\beta_{avg,r}$ is a hyperparameter typically set to $0.6$ for most scenes [S1].
* $\lambda_i$ are adjustable weights [S1].

**Edge Cases and Pitfalls:**
* **Scale Ambiguity:** A critical pitfall is the scale ambiguity between shading and reflectance; scaling reflectance by $k$ and shading by $1/k$ yields the same observed image [S1]. The mean reflectance regularization mitigates unreasonable brightness changes but does not fully resolve the ambiguity without additional constraints [S1].
* **Sampling Density:** Sparse sampling on objects can cause missing details, which may affect the accuracy of the shading and normal consistency terms [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_intrinsic_priors_loss(reflectance_batch, shading_batch, normal_batch, points_3d, beta_avg_r, lambdas):
    # reflectance_batch: list of [R_r, R_g, R_b]
    # shading_batch: list of [S_r, S_g, S_b]
    # normal_batch: list of [N_x, N_y, N_z]
    # points_3d: list of [x, y, z]
    # beta_avg_r: float, typically 0.6 [S1]
    # lambdas: dict of weights lambda_0 to lambda_7 [S1]
    n = len(reflectance_batch)

    # 1. Photometric loss L: formula not provided in sources [S1]
    loss_L = 0.0

    # 2. Depth loss L_d: formula not provided in sources [S1]
    loss_d = 0.0

    # 3. Mean Reflectance Regularization L_mean,r [S1]
    max_reflectance = [max(r) for r in reflectance_batch]
    loss_mean_r = sum(abs(m - beta_avg_r) for m in max_reflectance) / n

    # 4. Shading Smoothness L_s [S1]
    # Variance of shading for all pixels in a single batch
    mean_shading = [sum(shading_batch[i][c] for i in range(n)) / n for c in range(3)]
    var_shading = [sum((shading_batch[i][c] - mean_shading[c])**2 for i in range(n)) / n for c in range(3)]
    loss_s = sum(var_shading) / 3.0

    # 5. Shading-Normal Consistency L_sn [S1]
    # Note: Source formula sums over r in R but uses r'; implementation assumes pairwise comparison
    loss_sn = 0.0
    for i in range(n):
        for j in range(n):
            # Cosine similarity of normals
            dot_prod = sum(normal_batch[i][k] * normal_batch[j][k] for k in range(3))
            norm_i = math.sqrt(sum(x**2 for x in normal_batch[i]))
            norm_j = math.sqrt(sum(x**2 for x in normal_batch[j]))
            cos_sim = dot_prod / (norm_i * norm_j) if norm_i > 1e-8 and norm_j > 1e-8 else 0.0
            # Distance between 3D points
            dist = math.sqrt(sum((points_3d[i][k] - points_3d[j][k])**2 for k in range(3)))
            # Weighted shading difference
            weight = cos_sim * (1 - math.tanh(dist))
            loss_sn += sum(abs(shading_batch[i][c] - shading_batch[j][c]) for c in range(3)) * weight
    loss_sn /= n

    # 6. Other priors: L_chro, L_rs, L_nrs [S1]
    # Formulas for L_chro, L_rs, and L_nrs are not given in the sources.
    loss_chro = 0.0
    loss_rs = 0.0
    loss_nrs = 0.0

    # 7. Total Loss [S1]
    total_loss = (lambdas['lambda_0'] * loss_L +
                  lambdas['lambda_1'] * loss_d +
                  lambdas['lambda_2'] * loss_mean_r +
                  lambdas['lambda_3'] * loss_chro +
                  lambdas['lambda_4'] * loss_rs +
                  lambdas['lambda_5'] * loss_nrs +
                  lambdas['lambda_6'] * loss_s +
                  lambdas['lambda_7'] * loss_sn)
    return total_loss
```

## Operating characteristics
* **Complexity:** Adds computational overhead due to the calculation of batch variance and pairwise normal/shading consistency terms [S1]. The shading-normal consistency term $\mathcal{L}_{sn}$ is particularly expensive as it involves pairwise comparisons or local neighborhood searches [S1].
* **Performance:** The regularization is described as crucial for complex scenes, significantly reducing noise in normal maps and improving the smoothness of intrinsic components [S1].
* **Use Case:** Best suited for applications requiring consistent object insertion and relighting [S1].

## Background
This method originates from the work on "Relighting Scenes With Object Insertions in Neural Radiance Fields" [S1]. It adapts traditional intrinsic image decomposition constraints (chromaticity, sparsity, non-local sparsity) to the NeRF backbone to enable the separation of scene and object intrinsic properties for compositing [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
