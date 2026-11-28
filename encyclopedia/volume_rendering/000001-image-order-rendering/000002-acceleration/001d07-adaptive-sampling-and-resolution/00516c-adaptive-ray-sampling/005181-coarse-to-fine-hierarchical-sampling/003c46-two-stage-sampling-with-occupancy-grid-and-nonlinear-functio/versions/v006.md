# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method addresses the inefficiency of uniform sampling in neural implicit surface reconstruction by combining spatial acceleration with density-aware sampling. It solves the problem of wasting computation on empty space and scattering samples randomly within occupied regions. Unlike [NeuS](8828-terminerf.md) which uses inverse transform sampling that scatters points randomly near the surface, and [Instant-NGP](3393-mip-nerf-360-sampling-and-training.md) which uses uniform sampling within occupied grid cells, this method uses an Occupancy Grid to skip empty regions and a nonlinear piecewise exponential function to bias samples toward the surface within occupied regions [S1].

## How it works
**Inputs:**
*   Ray origin $o$ and direction $d$.
*   Occupancy Grid (binary array).
*   Geometry Network $F_\theta$ (predicts SDF and geometry features).
*   Appearance Network $F_\beta$ (predicts color).
*   Hash Encoding $\gamma$ (multi-resolution hash grid).

**Outputs:**
*   Rendered color $\hat{C}(x)$ and accumulated opacity $\hat{O}_k$ for the ray.

**Data Structures:**
1.  **Occupancy Grid:** A binary array where each cell indicates whether an object surface intersects that spatial region. Cells are marked occupied if the SDF value is near zero or if the ray intersects the bounding box of the object [S1].
2.  **Hash Encoding:** Multi-resolution hash grid with 16 levels, ranging from $2^5$ to $2^{16}$. Each level has a hash feature size of 2 [S1].
3.  **Geometry Network:** A Multi-Layer Perceptron (MLP) with 1 layer and hidden size 64. Input is the hashed position $\gamma(x)$. Outputs SDF value $f_{sdf}(x)$ and geometry feature vector $enc$ [S1].
4.  **Appearance Network:** An MLP with 2 layers and hidden size 64. Inputs are position $x$, normal $n$, geometry feature $enc$, and direction $d$ encoded by Spherical Harmonics [S1].

**Numbered Steps:**

1.  **Coarse Stage (Occupancy Grid Filtering):**
    *   Cast the ray through the scene.
    *   Query the Occupancy Grid to identify intervals along the ray that intersect empty space.
    *   Skip sampling in empty cells. Only sample points within occupied cells [S1].

2.  **Fine Stage (Nonlinear Function Guided Sampling):**
    *   For each occupied interval $[t_i, t_{i+1}]$ identified in the coarse stage:
        *   Evaluate the weight $w(t)$ at the endpoints using the coarse samples: $a = w(t_i)$ and $b = w(t_{i+1})$, where $w(t) = T(t)\alpha(t)$ [S1].
        *   Define the nonlinear piecewise exponential function $\hat{w}(s)$ for $s \in [0, 1]$ (normalized interval):
            $$ \hat{w}(s) = a \left( \frac{b}{a} \right)^s $$
            where $a \ge 0$ and $b \le 1$ [S1].
        *   Compute the cumulative distribution function (CDF) integral for this interval:
            $$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \frac{b - a}{\ln b - \ln a} \quad (\text{if } a \neq b) $$
            If $a = b$, the integral is simply $a$ [S1].
        *   Use inverse transform sampling with $\hat{w}(s)$ to draw new sample points $s$ within $[0, 1]$, mapping them back to $[t_i, t_{i+1}]$. This biases samples toward the endpoint with the higher weight (typically the surface) [S1].

3.  **Network Evaluation:**
    *   For each sample point $x$:
        *   Compute normal $n$ using finite differences of the SDF to ensure smoothness:
            $$ \nabla f_{sdf}(x) \approx \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
            where $\epsilon$ is the hash grid size [S1].
            $$ n = \frac{\nabla f_{sdf}(x)}{\|\nabla f_{sdf}(x)\|} $$
        *   Compute geometry feature $enc = F_\theta(\gamma(x))$ [S1].
        *   Encode direction $d$ using Spherical Harmonics [S1].
        *   Compute color $c = F_\beta(x, n, enc, d)$ [S1].

4.  **Rendering:**
    *   Compute transmittance $T$ and opacity $\alpha$ from SDF values (using a sigmoid or similar mapping as per standard NeuS/NeRF conventions, though exact mapping is not detailed in [S1], it is implied by the weight calculation $w=T\alpha$).
    *   Accumulate color and opacity:
        $$ \hat{C}(x) = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i} c_i $$
        $$ \hat{O}_k = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i} $$
        where $T_{k,i} = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].

**Parameters:**
*   **Hash Encoding:** 16 levels, $2^5$ to $2^{16}$, feature size 2 per level [S1].
*   **Geometry MLP:** 1 layer, hidden size 64 [S1].
*   **Appearance MLP:** 2 layers, hidden size 64 [S1].
*   **Learning Rate:** 0.001 for both Geometry Network $F_\theta$ and Appearance Network $F_\beta$ [S1].
*   **Optimizer:** AdamW with betas $(0.9, 0.99)$ [S1].
*   **Regularization Weights:** $\lambda_1 = 0.1$ (Eikonal), $\lambda_2 = 0.1$ (Mask) [S1].
*   **Progressive Training Update Step:** 1000 iterations. The hash grid levels are updated every 1000 iterations to refine from coarse to fine [S1].

**Edge Cases and Pitfalls:**
*   **Normal Noise:** Using analytic gradients for normals in NeuS can be noisy. This method uses finite differences for smoother normals [S1].
*   **Logarithm Singularity:** If $a=0$ or $b=0$ in the nonlinear function, $\ln a$ or $\ln b$ is undefined. The sources do not specify a guard for this case [G: Add a small epsilon to $a$ and $b$ before taking logarithms, or handle $a=0$ or $b=0$ as a linear interpolation case].
*   **Empty Regions:** If the Occupancy Grid is inaccurate, valid surface points might be skipped. The grid must be updated or initialized correctly based on the current SDF estimate [G].

## Operating characteristics
*   **Time Complexity:** Reduced compared to uniform sampling due to skipping empty regions (Occupancy Grid) and fewer samples needed for convergence (Nonlinear Function). The hash encoding provides $O(1)$ lookup time for features [S1].
*   **Memory Complexity:** Depends on the hash grid size (16 levels, $2^5$ to $2^{16}$) and the Occupancy Grid resolution. The sources do not specify the exact memory footprint [S1].
*   **Performance:** On DTU and NeRF Synthetic datasets, the method achieves better Chamfer Distance (CD) and Peak Signal-to-Noise Ratio (PSNR) than NeuS and Neus-acc with fewer iterations (20,000) [S1].
*   **Strengths:** Efficient sampling reduces training time. Nonlinear function biases samples toward the surface, improving reconstruction quality. Finite difference normals provide smoother geometry [S1].
*   **Limitations:** Requires an Occupancy Grid, which adds complexity to the pipeline. The nonlinear function assumes monotonic weight changes within intervals, which may not hold for complex geometry [G].
*   **When to use:** When training time is a constraint and high-quality surface reconstruction is required. Suitable for single-view or multi-view RGB inputs without depth priors [S1].
*   **When not to use:** When depth priors are available (e.g., DoNeRF) or when the scene has large empty spaces that are not well-captured by the Occupancy Grid [S1].

## Background
*   **Origin:** Proposed by Siyuan He and Xinglin Liu from Wuyi University, published in March 2024 [S1].
*   **History:** Builds on NeRF [1] and NeuS [2]. NeuS uses uniform sampling followed by inverse transform sampling, which scatters points randomly. Instant-NGP [5] uses Occupancy Grids for skipping empty space but uses uniform sampling within occupied cells. This method combines both ideas, adding a nonlinear function to guide sampling within occupied cells [S1].
*   **Usage:** Used for neural implicit surface reconstruction, surface recovery, and novel view synthesis on DTU and NeRF Synthetic datasets [S1].

## Sources
[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
