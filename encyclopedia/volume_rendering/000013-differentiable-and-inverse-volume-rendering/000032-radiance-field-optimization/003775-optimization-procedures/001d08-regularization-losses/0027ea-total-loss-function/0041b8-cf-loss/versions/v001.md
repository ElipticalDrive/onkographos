# CF Loss
Also called: Closed-form color loss

## Overview
CF Loss is a regularization term used in radiance-field optimization to mitigate shape-radiance ambiguity by providing explicit geometric supervision [S1, unofficial]. Unlike standard photometric or ray-entropy losses, which are largely insensitive to false geometry, CF Loss focuses significantly on correcting geometric errors by comparing rendered colors against a closed-form color estimate derived from the density field [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of rays $\mathcal{R}_{cf}$ sampled from training views; the current neural radiance field parameters (density $\sigma$ and color $\mathbf{c}$); the weight factor $\lambda$.
*   **Outputs:** A scalar loss value $L_{cf}$ used for backpropagation.

**Data Structures**
The method operates on standard ray-buffer structures. For each ray $r \in \mathcal{R}_{cf}$, the following values are computed along the ray segment:
*   Density $\sigma_i$ at sample points $i$.
*   Color $\mathbf{c}_i$ at sample points $i$.
*   Transmittance $T_i = \exp(-\sum_{j < i} \sigma_j \delta_j)$, where $\delta_j$ is the distance between samples.

**Method Steps**
1.  **Sample Rays:** Select $|\mathcal{R}_{cf}|$ rays from the training camera views.
2.  **Compute Closed-Form Color:** For each ray, compute a "closed-form" color $\mathbf{c}_{cf}$. This is derived by assuming the color is constant or by integrating the color field weighted by the density, effectively decoupling color optimization from the specific transmittance artifacts that cause ambiguity. The source notes that this loss reflects errors in the "rendered image by the closed-form color field" [S1, unofficial].
    *   *Note:* The exact mathematical formula for $\mathbf{c}_{cf}$ is not explicitly provided in the source text, but it is described as a method to estimate color independent of the standard volume rendering integral's transmittance weighting, allowing it to highlight geometric errors [S1, unofficial].
3.  **Compute Photometric Error:** Calculate the difference between the standard rendered color $\mathbf{c}_{rendered}$ (from the standard volume rendering equation) and the closed-form color $\mathbf{c}_{cf}$.
    $$ L_{ray} = \| \mathbf{c}_{rendered} - \mathbf{c}_{cf} \|^2 $$
4.  **Aggregate Loss:** Average the ray losses and scale by $\lambda$.
    $$ L_{cf} = \lambda \cdot \frac{1}{|\mathcal{R}_{cf}|} \sum_{r \in \mathcal{R}_{cf}} L_{ray} $$

**Parameters**
*   $\lambda$ (Weight factor): Controls the influence of CF loss. Typical values: 1–10. Values $\ge 20$ degrade performance [S1, unofficial].
*   $|\mathcal{R}_{cf}|$ (Number of rays): Rays used per iteration. Typical values: 10–25. Higher values improve geometry but increase cost [S1, unofficial].

**Pitfalls**
*   **Dominance:** If $\lambda$ is too large (e.g., 20), CF loss dominates training. Since CF loss primarily supervises the density field (geometry) while the color field lacks equivalent direct supervision in this term, the color field cannot be well optimized, causing PSNR to decrease [S1, unofficial].
*   **Cost:** Increasing $|\mathcal{R}_{cf}|$ linearly increases training time.

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** Training time increases with $|\mathcal{R}_{cf}|$. On the DTU dataset, baseline training takes 17 minutes. Adding CF loss with 10 rays increases this to 32 minutes; with 25 rays, it increases to 43 minutes [S1, unofficial].
*   **Memory:** No significant additional memory overhead is reported beyond storing the ray samples and intermediate color fields [S1, unofficial].

**Measured Performance**
*   **DTU Dataset (Average):**
    *   $\lambda=1, |\mathcal{R}_{cf}|=10$: PSNR 32.00 / IMRC 16.18
    *   $\lambda=2, |\mathcal{R}_{cf}|=10$: PSNR 32.04 / IMRC 16.12
    *   $\lambda=5, |\mathcal{R}_{cf}|=10$: PSNR 32.02 / IMRC 16.29
    *   $\lambda=10, |\mathcal{R}_{cf}|=10$: PSNR 32.06 / IMRC 16.26
    *   $\lambda=1, |\mathcal{R}_{cf}|=25$: PSNR 32.03 / IMRC 16.44
    *   $\lambda=2, |\mathcal{R}_{cf}|=25$: PSNR 32.07 / IMRC 16.50
    *   $\lambda=5, |\mathcal{R}_{cf}|=25$: PSNR 32.04 / IMRC 16.59
    *   $\lambda=10, |\mathcal{R}_{cf}|=25$: PSNR 32.08 / IMRC 16.66 [S1, unofficial]
*   **DTU Scan 24 (Specific Case):**
    *   $\lambda=0$: PSNR 27.72 / IMRC 14.96
    *   $\lambda=1$: PSNR 27.79 / IMRC 15.46
    *   $\lambda=5$: PSNR 27.81 / IMRC 16.10
    *   $\lambda=10$: PSNR 27.79 / IMRC 16.47
    *   $\lambda=20$: PSNR 27.73 / IMRC 16.73 [S1, unofficial]

**Strengths and Limitations**
*   **Strengths:** CF Loss accounts for 28% of the total loss in regions with false geometry (e.g., a falsely reconstructed top surface of a bottom rock), compared to only 4.7% for photometric loss and 12.4% for per-point RGB loss. It is significantly more sensitive to geometric errors than common ray-based losses [S1, unofficial].
*   **Limitations:** It does not directly supervise the color field, leading to potential color degradation if $\lambda$ is too high. It adds computational overhead (nearly doubling training time with 25 rays) [S1, unofficial].

## Background
CF Loss was introduced in the context of reducing shape-radiance ambiguity in radiance fields, specifically evaluated on Plenoxels [S1, unofficial]. It addresses the limitation that minimizing photometric loss alone cannot assure correct geometry due to the inherent ambiguity between shape and radiance [S1, unofficial].

## Sources
[S1] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26 (unofficial)
