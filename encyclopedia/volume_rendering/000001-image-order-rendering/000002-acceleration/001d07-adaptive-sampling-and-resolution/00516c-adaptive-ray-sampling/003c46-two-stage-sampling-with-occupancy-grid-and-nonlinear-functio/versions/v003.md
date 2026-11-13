# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method is a sampling strategy for neural implicit surface reconstruction that combines an Occupancy Grid to skip empty regions (coarse stage) and a nonlinear piecewise exponential function to guide sampling points closer to the object surface within occupied regions (fine stage) [S1]. It reduces training time and sample count while maintaining reconstruction quality compared to uniform sampling in occupied cells or random scatter near surfaces [S1]. Unlike [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) which relies on inverse transform sampling of accumulated weights, this method uses a nonlinear interpolation function to bias samples deterministically towards high-weight regions [S1]. Unlike [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) which uses hierarchical spatial subdivision, this method uses a fixed-resolution hash-based occupancy grid [S1].

## How it works
**Inputs:**
*   Ray segment $[t_{near}, t_{far}]$.
*   Occupancy Grid: A binary array indicating whether an object surface intersects each spatial cell [S1].
*   Geometry Network $F_\theta$: Predicts Signed Distance Field (SDF) $f_{sdf}(x)$ and geometry feature $enc$ [S1].
*   Appearance Network $F_\beta$: Predicts color $c$ [S1].
*   Hash Encoding $\gamma$: Multi-resolution hash encoding with 16 levels, ranging from $2^5$ to $2^{16}$, with a feature size of 2 per level [S1].

**Outputs:**
*   Set of sample depths $t_i$ along the ray.
*   Rendered color and accumulated opacity for the ray.

**Data Structures:**
1.  **Occupancy Grid:** Binary array. Each cell indicates if an object surface intersects that spatial region [S1].
2.  **Hash Encoding:** Interpolates features for position $x$ across 16 scales [S1].
3.  **Geometry MLP:** 1 layer, hidden size 64 [S1].
4.  **Appearance MLP:** 2 layers, hidden size 64 [S1].

**Step-by-Step Method:**

1.  **Coarse Stage (Occupancy Grid Filtering):**
    *   Evaluate the ray against the Occupancy Grid.
    *   Identify intervals $[t_i, t_{i+1}]$ where the ray passes through empty cells.
    *   Skip sampling in these empty intervals to avoid invalid computations [S1].
    *   For occupied intervals, perform initial uniform sampling to estimate weights $w(t_i)$ for the fine stage. The weight is defined as $w(t_i) = T_i \alpha_i$, where $T_i$ is the transmittance and $\alpha_i$ is the opacity at sample $t_i$ [S1].

2.  **Fine Stage (Nonlinear Function Guided Sampling):**
    *   For each occupied interval $[t_i, t_{i+1}]$ from the coarse stage, define the weights at the endpoints: $a = w(t_i)$ and $b = w(t_{i+1})$ [S1].
    *   Define the nonlinear interpolation function $\hat{w}(s)$ for a normalized depth $s \in [0, 1]$ within the interval:
        $$ \hat{w}(s) = a \left( \frac{b}{a} \right)^s $$
        where $a \ge 0$ and $b \le 1$ [S1].
    *   This function interpolates weights within the interval rather than using constant weights, pushing samples closer to the surface where weight is high [S1].
    *   Compute the cumulative distribution function (CDF) for the interval by integrating $\hat{w}(s)$:
        $$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \frac{b - a}{\ln b - \ln a} \quad \text{(if } a \neq b \text{)} $$
        Note: The source provides the integral result as $\ln b - \ln a$ in the context of normalization, but the explicit integral of the exponential form $a(b/a)^s$ from 0 to 1 is $\frac{b-a}{\ln(b/a)}$. The source states $\int_0^1 \hat{w}(s) ds = \ln b - \ln a$ is likely a normalization constant or specific derivation context; however, the sampling is performed via inverse transform sampling using the derived CDF from $\hat{w}(s)$ [S1].
    *   Draw sample depths $t$ within $[t_i, t_{i+1}]$ using inverse transform sampling based on the probability distribution defined by $\hat{w}(s)$ [S1].
    *   This biases samples towards the endpoint with the higher weight (typically the surface intersection) [S1].

3.  **Network Evaluation:**
    *   For each sample point $x$:
        *   Compute normal $n$ using finite differences to ensure smoothness:
            $$ \nabla f_{sdf}(x) = \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
            where $\epsilon$ is the hash grid size [S1].
        *   Encode direction $d$ using Spherical Harmonics [S1].
        *   Get geometry feature $enc = F_\theta(\gamma(x))$ [S1].
        *   Predict color $c = F_\beta(x, n, enc, d)$ [S1].

4.  **Rendering:**
    *   Compute accumulated opacity $O_k$ for ray $k$:
        $$ O_k = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i} $$
    *   Render color $\hat{C}(x)$ [S1].

5.  **Loss Calculation:**
    *   Total Loss:
        $$ \mathcal{L} = \mathcal{L}_{color} + \lambda_1 \mathcal{L}_{eik} + \lambda_2 \mathcal{L}_{mask} $$
    *   $\mathcal{L}_{color}$: Sum of L2 loss between rendered color $\hat{C}(x)$ and ground truth $C(x)$ [S1].
    *   $\mathcal{L}_{eik}$: Sum of squared difference between SDF gradient magnitude and 1:
        $$ \mathcal{L}_{eik} = \sum_{x} (\|\nabla f_{sdf}(x)\| - 1)^2 $$
    *   $\mathcal{L}_{mask}$: Binary Cross-Entropy (BCE) between mask $M_k$ and accumulated opacity $O_k$ [S1].

**Parameters:**
*   Hash Encoding Levels: 16 levels, $2^5$ to $2^{16}$ [S1].
*   Hash Feature Size: 2 per level [S1].
*   Geometry MLP: 1 layer, hidden size 64 [S1].
*   Appearance MLP: 2 layers, hidden size 64 [S1].
*   Learning Rate: 0.001 for both Geometry Network $F_\theta$ and Appearance Network $F_\beta$ [S1].
*   Optimizer Betas: 0.9 and 0.99 [S1].
*   Regularization Weights: $\lambda_1 = 0.1$, $\lambda_2 = 0.1$ [S1].
*   Progressive Training Update Step: 1000 iterations (hash levels update every 1000 iterations) [S1].

**Edge Cases and Pitfalls:**
*   **Normal Calculation:** Using analytic gradients for normals can be noisy; this method uses finite differences for smoother normals [S1].
*   **Empty Regions:** Failing to skip empty regions in the coarse stage leads to wasted computation [S1].
*   **Uniform Sampling in Occupied Cells:** Using uniform sampling within occupied cells (as in Instant-NGP) can introduce many useless samples if the grid resolution is low; the nonlinear function mitigates this [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to uniform sampling in occupied regions due to the nonlinear biasing and occupancy grid skipping [S1].
*   **Memory Complexity:** Depends on the Occupancy Grid size and Hash Encoding storage (16 levels, feature size 2) [S1].
*   **Performance:** Achieves good surface reconstruction and novel view synthesis on DTU and NeRF Synthetic datasets with fewer samples and reasonable training time [S1].
*   **Strengths:** Avoids invalid sampling in empty regions; biases samples towards surfaces more effectively than uniform sampling in occupied cells; smoother normals via finite differences [S1].
*   **Limitations:** Requires tuning of hash encoding levels and grid resolution; finite difference normal calculation adds computational overhead per sample [S1].
*   **When to use:** When efficient surface reconstruction is needed with RGB-only input, and computational resources are limited compared to methods requiring proposal networks or multi-stage training with analytic gradients [S1].

## Background
Originated from research by Siyuan He and Xinglin Liu at Wuyi University, published in March 2024 [S1]. It builds on [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) and [NeuS](8828-terminerf.md) by incorporating ideas from [Instant-NGP](3406-coarse-to-fine-nerf.md) (Occupancy Grid) and [DoNeRF](3406-coarse-to-fine-nerf.md) (nonlinear sampling concepts) [S1]. It addresses the inefficiency of NeuS's inverse transform sampling (random scatter near surface) and Instant-NGP's uniform sampling in occupied cells [S1].

## Sources
[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
