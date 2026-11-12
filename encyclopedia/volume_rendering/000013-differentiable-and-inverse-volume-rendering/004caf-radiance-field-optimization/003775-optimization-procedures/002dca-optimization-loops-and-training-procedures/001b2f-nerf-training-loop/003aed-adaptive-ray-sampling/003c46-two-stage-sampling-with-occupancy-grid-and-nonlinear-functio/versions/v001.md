# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method is a sampling strategy for neural implicit surface reconstruction that combines an Occupancy Grid to skip empty regions during a coarse stage and a nonlinear piecewise exponential function to guide sampling points closer to the object surface within occupied regions during a fine stage [S1]. It solves the problem of inefficient sampling in neural radiance fields by reducing training time and sample count while maintaining high-quality surface reconstruction [S1]. Unlike NeuS, which uses uniform sampling followed by inverse transform sampling that scatters points randomly near the surface, and Instant-NGP, which uses uniform sampling within occupied grid cells, this method uses a nonlinear function to bias samples towards the surface within the occupied regions [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera parameters (position, direction).
*   Ray origin $o$ and direction $d$.
*   Occupancy Grid (binary array indicating occupied cells).
*   Geometry Network $F_\theta$ (predicts SDF and geometry features).
*   Appearance Network $F_\beta$ (predicts color).
*   Hash Encoding $\gamma$ (multi-resolution hash grid).

**Outputs:**
*   Rendered color $\hat{C}(x)$ for the ray.
*   Accumulated opacity $\hat{O}_k$.
*   Gradients for backpropagation.

### Data Structures

**Occupancy Grid:**
*   A binary array where each cell indicates whether an object surface intersects that spatial region [S1].
*   Used to skip sampling in empty cells during the coarse stage [S1].

**Hash Encoding $\gamma$:**
*   Multi-resolution hash grid with 16 levels, ranging from $2^5$ to $2^{16}$ [S1].
*   Hash feature size is 2 per level [S1].
*   Used to encode input positions for fast lookup and interpolation [S1].

**Network Architectures:**
*   **Geometry Network $F_\theta$:** A Multi-Layer Perceptron (MLP) with 1 layer and hidden size 64 [S1]. Input: Hash encoded position $\gamma(x)$. Output: Signed Distance Field (SDF) value $f_{sdf}(x)$ and geometry feature $enc$ [S1].
*   **Appearance Network $F_\beta$:** An MLP with 2 layers and hidden size 64 [S1]. Input: Position $x$, normal $n$, geometry feature $enc$, and direction $d$ encoded by Spherical Harmonics (SH) [S1]. Output: Color $c$ [S1].

### Numbered Steps

1.  **Coarse Stage (Occupancy Grid Sampling):**
    *   Cast the ray through the scene.
    *   Use the Occupancy Grid to identify occupied cells along the ray [S1].
    *   Skip sampling in empty cells to avoid invalid computations [S1].
    *   Perform uniform sampling within the occupied regions to generate coarse sample points $t_i$ [S1].
    *   For each coarse sample point $t_i$, compute the weight $w(t_i)$ using the accumulated opacity and transmittance, consistent with NeuS: $w(t_i) = T_i \alpha_i$ [S1]. Here, $T_i$ is the transmittance up to $t_i$, and $\alpha_i$ is the opacity at $t_i$ derived from the SDF [S1].

2.  **Fine Stage (Nonlinear Function Guided Sampling):**
    *   For each interval $[t_i, t_{i+1}]$ between coarse samples, define the weights at the endpoints: $a = w(t_i)$ and $b = w(t_{i+1})$ [S1]. Note that $a \ge 0$ and $b \le 1$ [S1].
    *   Define the nonlinear piecewise exponential function $\hat{w}(s)$ for a normalized depth $s \in [0, 1]$ within the interval:
        $$ \hat{w}(s) = a \left( \frac{b}{a} \right)^s $$
        [S1]
    *   This function interpolates weights within the interval $[t_i, t_{i+1}]$ rather than using constant weights, pushing samples closer to the surface where weight is high [S1].
    *   Compute the cumulative distribution function (CDF) for the interval by integrating $\hat{w}(s)$:
        $$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \ln b - \ln a $$
        [S1]
    *   Use inverse transform sampling with this CDF to generate fine sample points within the interval [S1].
    *   The adaptive nature of $\hat{w}(s)$ ensures that if $a \ll b$, the weight increases sharply towards $b$, biasing samples towards $t_{i+1}$; if $a \gg b$, it decreases sharply, biasing samples towards $t_i$ [S1].

3.  **Rendering and Loss Computation:**
    *   For each fine sample point $x$, compute the SDF $f_{sdf}(x) = F_\theta(\gamma(x))$ and geometry feature $enc$ [S1].
    *   Compute the normal $n$ using finite differences for smoother gradients:
        $$ \nabla f_{sdf}(x) = \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
        $$ n = \frac{\nabla f_{sdf}(x)}{\|\nabla f_{sdf}(x)\|} $$
        [S1]
        Here, $\epsilon$ is the hash grid size [S1].
    *   Compute the color $c = F_\beta(x, n, enc, d)$ where $d$ is the view direction encoded by Spherical Harmonics [S1].
    *   Compute the total loss $\mathcal{L}$:
        $$ \mathcal{L} = \mathcal{L}_{color} + \lambda_1 \mathcal{L}_{eik} + \lambda_2 \mathcal{L}_{mask} $$
        [S1]
    *   $\mathcal{L}_{color}$ is the sum of L2 loss between rendered color $\hat{C}(x)$ and ground truth $C(x)$ over all sample points [S1]:
        $$ \mathcal{L}_{color} = \sum_x (\hat{C}(x) - C(x))^2 $$
        [S1]
    *   $\mathcal{L}_{eik}$ is the Eikonal loss, ensuring the SDF gradient magnitude is 1:
        $$ \mathcal{L}_{eik} = \sum_x (\|\nabla f_{sdf}(x)\| - 1)^2 $$
        [S1]
    *   $\mathcal{L}_{mask}$ is the Binary Cross-Entropy (BCE) between the mask $M_k$ and accumulated opacity $\hat{O}_k$:
        $$ \mathcal{L}_{mask} = BCE(M_k, \hat{O}_k) $$
        [S1]
    *   Accumulated opacity $\hat{O}_k$ for a ray $k$ is:
        $$ \hat{O}_k = \sum_{i=1}^n T_{k,i} \alpha_{k,i} $$
        [S1]

### Parameters
*   **Hash Encoding Levels:** 16 levels, ranging from $2^5$ to $2^{16}$ [S1].
*   **Hash Feature Size:** 2 per level [S1].
*   **Geometry MLP Layers:** 1 layer [S1].
*   **Geometry MLP Hidden Size:** 64 [S1].
*   **Appearance MLP Layers:** 2 layers [S1].
*   **Learning Rate:** 0.001 for both Geometry Network $F_\theta$ and Appearance Network $F_\beta$ [S1].
*   **Optimizer Betas:** 0.9 and 0.99 [S1].
*   **Regularization Weights:** $\lambda_1$ (Eikonal) = 0.1, $\lambda_2$ (Mask) = 0.1 [S1].
*   **Progressive Training Update Step:** 1000 iterations [S1].
*   **Optimizer:** AdamW [S1].
*   **Learning Rate Schedule:** Constant factor decay for the first 5000 iterations, then exponential decay [S1].

### Edge Cases and Pitfalls
*   **Normal Computation:** Using analytic gradients for normals in NeuS can be noisy; this method uses finite differences for smoother normals [S1].
*   **Low Resolution Grid:** If the Occupancy Grid resolution is low, uniform sampling within occupied cells can still introduce many invalid samples; the nonlinear function helps mitigate this by biasing samples towards the surface [S1].
*   **Input Requirements:** Unlike DoNeRF, this method only requires RGB images, not RGBD images [S1].

## Operating characteristics

*   **Time and Memory Complexity:** The method reduces training time and sample count compared to NeuS and Instant-NGP by skipping empty regions and biasing samples towards the surface [S1]. The use of multi-resolution hash encoding and shallow MLPs (1 layer for geometry) makes the SDF calculation several orders of magnitude faster than deeper MLPs like in NeuS [S1].
*   **Measured Performance:** On the DTU dataset (14 scenes) and NeRF Synthetic dataset, the method achieves better Chamfer Distance (CD) and Peak Signal-to-Noise Ratio (PSNR) compared to NeuS, Neus-acc, and Neus-facto after 20,000 iterations [S1].
*   **Scaling:** The method scales well with the use of hash encoding and progressive training (updating hash levels every 1000 iterations) [S1].
*   **Strengths:** Efficient sampling, reduced training time, good surface reconstruction quality, works with RGB images only [S1].
*   **Limitations:** Relies on the quality of the Occupancy Grid and the nonlinear function's ability to accurately model the weight distribution [S1].
*   **When to use:** When efficient surface reconstruction from RGB images is required, and computational resources are limited [S1].
*   **When not to use:** When RGBD data is available and more complex depth-guided sampling (like DoNeRF) might be beneficial, or when extremely high-frequency details require deeper networks and more samples [S1].

## Background

*   **Origin:** Proposed by Siyuan He and Xinglin Liu from Wuyi University, Jiangmen, Guangdong [S1].
*   **Paper:** "Research on Efficient Sampling of Neural Implicit Surface Reconstruction" [S1].
*   **Date:** Received Feb. 20, 2024; accepted Mar. 21, 2024; published Mar. 29, 2024 [S1].
*   **History and Development:** The method builds on NeRF [1] and NeuS [2], incorporating ideas from Instant-NGP [5] for occupancy grids and NeuS2 [12]/Neuralangelo [14] for hash encoding and shallow MLPs [S1]. It addresses the inefficiencies of uniform sampling in Instant-NGP and the random scatter of inverse transform sampling in NeuS [S1].
*   **Usage:** Used for neural implicit surface reconstruction, surface recovery, and novel view synthesis [S1].

## Sources

[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
