# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method reduces training time and sample count in neural implicit surface reconstruction by combining an Occupancy Grid to skip empty regions (coarse stage) with a nonlinear piecewise exponential function to bias sampling points toward object surfaces within occupied regions (fine stage) [S1]. It differs from NeuS, which uses uniform coarse sampling followed by inverse transform sampling that scatters points randomly near the surface, and Instant-NGP, which uses uniform sampling within occupied grid cells [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters, RGB images, object masks (for real datasets like DTU), and a scene bounding box.
*   **Outputs:** Rendered colors, SDF values, and geometry features for sampled points along rays.

**Data Structures:**
1.  **Occupancy Grid:** A binary array where each cell indicates whether an object surface intersects that spatial region [S1]. Used to skip empty cells during ray marching.
2.  **Hash Encoding:** Multi-resolution hash grid with 16 levels, ranging from $2^5$ to $2^{16}$, with a feature size of 2 per level [S1].
3.  **Geometry Network ($F_\theta$):** A 1-layer MLP with hidden size 64, taking hash-encoded position $\gamma(x)$ as input, outputting SDF $f_{sdf}(x)$ and geometry feature $enc$ [S1].
4.  **Appearance Network ($F_\beta$):** A 2-layer MLP with hidden size 64, taking position $x$, normal $n$, geometry feature $enc$, and direction $d$ as input, outputting color $c$ [S1].

**Numbered Steps:**

1.  **Coarse Stage (Occupancy Grid Filtering):**
    *   Cast a ray from the camera through a pixel.
    *   March along the ray. Check the Occupancy Grid for each interval.
    *   If a cell is empty (binary value 0), skip sampling within that interval entirely [S1].
    *   If a cell is occupied, proceed to the fine stage for that interval.

2.  **Fine Stage (Nonlinear Function Sampling):**
    *   For an occupied interval $[t_i, t_{i+1}]$, calculate weights $w(t)$ from the coarse stage (or initial estimate). Let $a = w(t_i)$ and $b = w(t_{i+1})$, where $a \ge 0$ and $b \le 1$ [S1].
    *   Define the nonlinear weight function $\hat{w}(s)$ for a sample position $s \in [0, 1]$ within the normalized interval:
        $$ \hat{w}(s) = a \left( \frac{b}{a} \right)^s $$
        [S1]
    *   This function interpolates weights within the interval rather than using constant weights, pushing samples closer to the surface where weight is high [S1].
    *   Perform inverse transform sampling using $\hat{w}(s)$ to select sample points within $[t_i, t_{i+1}]$. The cumulative distribution function (CDF) normalization ensures:
        $$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \ln b - \ln a $$
        [S1]
    *   If $a=0$, handle the singularity by setting $\hat{w}(s)$ to a constant or small epsilon, or skip if no weight [G].

3.  **Network Evaluation:**
    *   For each sample point $x$:
        *   Compute SDF: $f_{sdf}(x) = F_\theta(\gamma(x))$ [S1].
        *   Compute Normal $n$ using finite differences to avoid noisy analytic gradients:
            $$ \nabla f_{sdf}(x) \approx \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
            where $\epsilon$ is the hash grid size [S1].
            $$ n = \frac{\nabla f_{sdf}(x)}{\|\nabla f_{sdf}(x)\|} $$
        *   Encode direction $d$ using Spherical Harmonics [S1].
        *   Compute Color: $c = F_\beta(x, n, enc, d)$ [S1].

4.  **Rendering and Loss Calculation:**
    *   Compute accumulated opacity $O_k$ for pixel $k$:
        $$ O_k = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i} $$
        [S1]
    *   Compute Loss:
        $$ \mathcal{L} = \mathcal{L}_{color} + \lambda_1 \mathcal{L}_{eik} + \lambda_2 \mathcal{L}_{mask} $$
        [S1]
    *   $\mathcal{L}_{color} = \sum_x (\hat{C}(x) - C(x))^2$ [S1].
    *   $\mathcal{L}_{eik} = \sum_x (\|\nabla f_{sdf}(x)\| - 1)^2$ [S1].
    *   $\mathcal{L}_{mask} = BCE(M_k, \hat{O}_k)$ [S1].

**Parameters:**
*   Hash Encoding Levels: 16 ($2^5$ to $2^{16}$) [S1].
*   Hash Feature Size: 2 per level [S1].
*   Geometry MLP: 1 layer, hidden size 64 [S1].
*   Appearance MLP: 2 layers, hidden size 64 [S1].
*   Learning Rate: 0.001 for both networks [S1].
*   Optimizer: AdamW, betas (0.9, 0.99) [S1].
*   Regularization Weights: $\lambda_1 = 0.1$, $\lambda_2 = 0.1$ [S1].
*   Progressive Training Update Step: 1000 iterations (update hash grid levels) [S1].

**Pitfalls:**
*   Using analytic gradients for normals in NeuS can be noisy; this method uses finite differences for smoother normals [S1].
*   The nonlinear function $\hat{w}(s)$ requires careful handling when $a=0$ to avoid division by zero or log(0) [G].

## Operating characteristics
*   **Time Complexity:** Reduced compared to uniform sampling due to skipping empty regions (Occupancy Grid) and concentrating samples near surfaces (Nonlinear Function) [S1].
*   **Memory Complexity:** Depends on the Occupancy Grid resolution and Hash Encoding size. Hash Encoding uses 16 levels with 2 features per level [S1].
*   **Performance:** Achieves better surface reconstruction (Chamfer Distance) and novel view synthesis (PSNR) on DTU and NeRF Synthetic datasets compared to NeuS and Instant-NGP [S1].
*   **Strengths:** Avoids invalid computations in empty regions; biases samples toward surfaces more effectively than uniform sampling within occupied cells [S1].
*   **Limitations:** Requires tuning of the nonlinear function parameters and hash grid resolution [S1].

## Background
Originated from research by Siyuan He and Xinglin Liu at Wuyi University, published in March 2024 [S1]. It builds on NeRF and NeuS, incorporating ideas from Instant-NGP (Occupancy Grid) and DoNeRF (nonlinear sampling) [S1].

## Sources
[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
