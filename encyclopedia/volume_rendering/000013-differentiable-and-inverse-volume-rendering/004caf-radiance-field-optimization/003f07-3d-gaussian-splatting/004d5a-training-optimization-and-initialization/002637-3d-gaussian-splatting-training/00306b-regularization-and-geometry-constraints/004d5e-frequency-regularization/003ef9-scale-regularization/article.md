# Scale Regularization

Also called: Lscales

## Overview
Scale regularization is a training-time technique that constrains the intrinsic scales of 4D Gaussians during deformation to prevent excessive dilation and improve anti-aliasing in video reconstruction [S1, unofficial]. It solves the problem where standard Gaussian Splatting methods suffer from artifacts or loss of detail when tested at sampling rates different from training, by jointly regulating the maximum sampling frequency relative to the Gaussian's scale constraint [S1, unofficial]. Unlike [Mip-Splatting](52-mip-splatting.md), which uses a fixed expansion scale, this method dynamically adapts frequency constraints, outperforming fixed-scale approaches in monocular video reconstruction [S1, unofficial].

## How it works
**Inputs:**
- Gaussian primitives $k$ with time-dependent scales $s_k(t)$ and centers $p_k(t)$ [S1, unofficial].
- Camera parameters for $N$ views: positions $c_n(t)$, viewing directions, and sampling frequencies $f_n(t)$ [S1, unofficial].
- Hyperparameters: minimum dilation ratio $\rho_{min}$, threshold $\rho_{thre}$, and loss weight $\lambda_1$ [S1, unofficial].

**Outputs:**
- Scale loss term $L_{scales}$ added to the total optimization loss [S1, unofficial].

**Data Structures:**
- $s_k(t)$: The scale parameter of Gaussian $k$ at time $t$, typically represented as the diagonal elements of the covariance matrix or a scalar multiplier if isotropic [G].
- $\hat{\nu}_k$: The maximum sampling frequency for Gaussian $k$ [S1, unofficial].
- $M_{1_k}$: Binary indicator mask for Gaussian $k$ [S1, unofficial].

**Step-by-Step Method:**

1.  **Compute Maximum Sampling Frequency ($\hat{\nu}_k$):**
    For each Gaussian $k$, calculate the maximum sampling frequency it can support without aliasing based on the camera views [S1, unofficial]. The maximum sampling frequency is derived by minimizing the ratio of depth to frequency across all visible cameras $n \in \{1, \dots, N\}$ [S1, unofficial]:
    $$
    \hat{\nu}_k = \min_{n \in \{1, \dots, N\}} \frac{1}{d_n(t) \cdot \mathbb{I}_n(p_k(t)) \cdot f_n(t)}
    $$
    Where:
    - $d_n(t)$ is the depth of primitive $k$ from camera $n$ at time $t$, approximated by the distance between the camera center and the primitive center $p_k(t)$ [S1, unofficial].
    - $f_n(t)$ is the sampling frequency of camera $n$, typically fixed at a constant interval [S1, unofficial].
    - $\mathbb{I}_n(p_k(t))$ is an indicator function that is 1 if primitive $k$ is visible to camera $n$, and 0 otherwise [S1, unofficial].

2.  **Compute Scale-Adaptive Dilation Coefficient ($\rho_{adapt}$):**
    Calculate the adaptive dilation coefficient based on the scale variation ratio, clipped to $[\rho_{min}, \rho_{max}]$ [S1, unofficial]:
    $$
    \rho_{adapt} = \text{clip}\left( \frac{s_k(t)^2}{s}, \rho_{min}, \rho_{max} \right)
    $$
    Where $s$ is a reference scale, often the initial scale or a global constant [S1, unofficial].

3.  **Determine Indicator Mask ($M_{1_k}$):**
    The loss is only applied to Gaussians whose scales fall within a specific critical range [S1, unofficial]. $M_{1_k}$ is 1 if:
    $$
    \rho_{thre} \cdot \sigma_{\hat{\nu}_k}^2 \cdot s < s_k(t)^2 < \rho_{min} \cdot \sigma_{\hat{\nu}_k}^2 \cdot s
    $$
    Otherwise, $M_{1_k} = 0$ [S1, unofficial]. The term $\sigma_{\hat{\nu}_k}^2$ acts as a scaling factor related to the frequency $\hat{\nu}_k$ [G].

4.  **Compute Scale Loss ($L_{scales}$):**
    Sum the squared difference between the normalized scale and the target minimum dilation ratio, weighted by the mask [S1, unofficial]:
    $$
    L_{scales} = \sum_k \left( \frac{s_k(t)^2}{\rho_{min} \cdot \hat{\nu}_k^2} - 1 \right)^2 \cdot M_{1_k}
    $$
    This term minimizes the dilation scale relative to the Gaussian's scale constraint [S1, unofficial].

5.  **Optimization Step:**
    Add $L_{scales}$ to the total loss $L$ with weight $\lambda_1$ [S1, unofficial]:
    $$
    L = L_{color} + \lambda_1 \cdot L_{scales}
    $$
    Optimize static Gaussians for the first 3000 iterations as a warm-up phase, then jointly optimize both the Gaussians and the deformation field [S1, unofficial].

**Parameters:**
- $\rho_{min}$: Minimum dilation ratio. Controls the lower bound of the adaptive filter. A smaller $\rho_{min}$ reduces the dilation scale but may cause a loss of Gaussian blurring on high-frequency information at lower sampling rates [S1, unofficial].
- $\rho_{thre}$: Threshold for scale activation. Determines the critical scale threshold below which Gaussians are prevented from dilation and subjected to $L_{scales}$ constraints [S1, unofficial].
    - Typical values: 0.05 for D3DGS (monocular), $5 \times 10^{-6}$ for 4DGaussians (multi-view) [S1, unofficial].
- $\lambda_1$: Loss weight. Typical value 0.1 [S1, unofficial].
- $\sigma_s$: Mip-filter sigma. Set to 0.2 to maintain consistency with Mip-Splatting [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Inaccurate Deformation:** If a Gaussian fails to deform accurately during training, scale regularization can erroneously enlarge redundant small Gaussians, causing negative optimization effects [S1, unofficial].
- **Hyperparameter Sensitivity:** The method is sensitive to $\rho_{thre}$ [S1, unofficial]. A lower $\rho_{thre}$ imposes stricter frequency constraints but reduces sensitivity to illumination variations, while a higher $\rho_{thre}$ weakens frequency constraints and restores lighting sensitivity [S1, unofficial].
- **Visibility:** The indicator function $\mathbb{I}_n$ ensures invisible Gaussians do not contribute to the frequency calculation or loss, preventing unnecessary splatting [S1, unofficial].

## Subtopics

### Hyperparameter ρthre Tuning
A sensitivity control mechanism for the scale regularization pipeline that balances frequency constraints against illumination sensitivity by adjusting the critical dilation threshold [S1, unofficial]. Decreasing $\rho_{thre}$ imposes stricter frequency constraints on Gaussians, which simultaneously reduces the model's sensitivity to illumination variations [S1, unofficial]. Increasing $\rho_{thre}$ weakens frequency constraints on 4DGS, restoring sensitivity to lighting changes [S1, unofficial]. Typical ablation values tested include 0.05, 0.0005, and 0.000005 [S1, unofficial].

### Gaussian Scale Regularization
A loss function to prevent large Gaussians caused by large triangles or scale training by forcing the largest scale of a Gaussian to be within a threshold of the triangle's scaled circumradius [S2, unofficial]. The regularization term is based on the circumradius of mesh triangles [S2, unofficial]. The loss function is defined as:
$$
L_{sc} = \sum_{i \in N_G} \max(\max(s^i) - \kappa_{r_c} r^{i}_{sc}, 0)
$$
where $N_G$ is the total number of Gaussians, $s^i$ are the scales in three directions for Gaussian $i$, $r^{i}_{sc}$ is the circumradius for the $i$-th Gaussian, and $\kappa_{r_c}$ is a constant scale factor set to 0.2 [S2, unofficial]. The circumradius is computed as:
$$
r_c = \frac{\|a\|\|b\|\|a-b\|}{2\|a \times b\|}
$$
where $a$ and $b$ are two edge vectors sharing a vertex in the triangle as a starting point [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Outperforms Mip-Splatting in monocular video reconstruction by providing stronger frequency constraints during deformation [S1, unofficial]. In multi-view reconstruction, it removes artifacts and significantly reduces redundant Gaussians, improving reconstruction quality [S1, unofficial].
- **Strengths:** Effective anti-aliasing at varying sampling rates; reduces redundant Gaussians; preserves Gaussian anisotropy via the 4D Scale-Adaptive Filter [S1, unofficial].
- **Limitations:** Sensitive to $\rho_{thre}$; risk of enlarging redundant small Gaussians if deformation is inaccurate; may reduce sensitivity to lighting changes if $\rho_{thre}$ is too low [S1, unofficial].
- **When to use:** Monocular video reconstruction (e.g., D-NeRF) and multi-view dynamic scene reconstruction (e.g., N3DV) where sampling rate variations or deformation artifacts are problematic [S1, unofficial].
- **When not to use:** Static scenes where frequency constraints are less critical, or when illumination sensitivity is paramount and $\rho_{thre}$ tuning cannot recover it without sacrificing anti-aliasing [S1, unofficial].

## Background
Originated in the paper "Alias-free 4D Gaussian Splatting" [S1, unofficial], which proposed the 4D Scale-Adaptive Filter and Scale Regularization to effectively constrain Gaussian frequencies in 4DGS [S1, unofficial]. The method was integrated into state-of-the-art frameworks D3DGS and 4DGaussians to address artifacts caused by sampling rate changes in dynamic scenes [S1, unofficial]. It builds on the intuition that minimizing the filter's dilation scale by distributing it between the filter and the Gaussian's scale constraint improves reconstruction quality [S1, unofficial].

## Sources
[S1] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
