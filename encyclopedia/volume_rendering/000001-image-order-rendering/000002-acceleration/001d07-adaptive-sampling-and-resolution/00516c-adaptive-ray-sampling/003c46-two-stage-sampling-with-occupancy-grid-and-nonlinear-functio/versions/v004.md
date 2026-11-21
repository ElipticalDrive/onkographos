# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method reduces training time and sample count in neural implicit surface reconstruction by combining an Occupancy Grid to skip empty regions (coarse stage) with a nonlinear piecewise exponential function to bias sampling points toward the object surface within occupied regions (fine stage) [S1]. Unlike NeuS, which uses uniform sampling followed by inverse transform sampling that scatters points randomly near the surface, and Instant-NGP, which uses uniform sampling within occupied grid cells, this method uses a nonlinear function to guide sampling closer to the surface where weight is high [S1].

## How it works
**Inputs:** RGB images, camera parameters, and optionally object masks.
**Outputs:** Rendered images, reconstructed 3D mesh (via Marching Cubes).

**Data Structures:**
1.  **Occupancy Grid:** A binary array where each cell indicates whether an object surface intersects that spatial region. Used to skip empty cells during ray marching [S1].
2.  **Hash Encoding:** Multi-resolution hash grid with 16 levels, ranging from $2^5$ to $2^{16}$. Each level has a feature size of 2 [S1].
3.  **Geometry Network ($F_\theta$):** A 1-layer MLP with hidden size 64. Inputs are the hash-encoded position $\gamma(x)$. Outputs the Signed Distance Field (SDF) value $f_{sdf}(x)$ and a geometry feature vector $enc$ [S1].
4.  **Appearance Network ($F_\beta$):** A 2-layer MLP with hidden size 64. Inputs are position $x$, normal $n$, geometry feature $enc$, and direction $d$ (encoded via Spherical Harmonics) [S1].

**Step 1: Coarse Stage (Occupancy Grid Sampling)**
The ray is marched through the scene. The Occupancy Grid is queried to identify occupied cells. Sampling points are generated only within occupied regions, skipping empty space to avoid invalid computations [S1]. Uniform sampling is typically used within these occupied intervals to calculate initial weights [S1].

**Step 2: Weight Calculation**
For each coarse sample point $t_i$, calculate the weight $w(t_i)$, which represents the contribution to the final color. This is defined as:
$$ w(t_i) = T_i \alpha_i $$
where $T_i$ is the transmittance and $\alpha_i$ is the opacity at sample $i$ [S1].

**Step 3: Fine Stage (Nonlinear Function Guided Sampling)**
Within each occupied interval $[t_i, t_{i+1}]$, a nonlinear function $\hat{w}(s)$ is used to guide the placement of fine samples. This function interpolates weights within the interval rather than using constant weights (as in NeuS) or uniform distribution (as in Instant-NGP) [S1].

The nonlinear function is defined as:
$$ \hat{w}(s) = a \left( \frac{b}{a} \right)^s $$
where $s \in [0, 1]$ is the normalized position within the interval, $a = w(t_i)$, and $b = w(t_{i+1})$ [S1].

The cumulative distribution function (CDF) for this interval is derived by integrating $\hat{w}(s)$:
$$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \ln b - \ln a $$
Note: This integral assumes the normalization such that the total probability mass is preserved. Inverse transform sampling is then used to generate sample points $s$ based on this CDF, pushing samples closer to the endpoint with higher weight (typically the surface) [S1].

**Step 4: Normal Estimation**
Normals $n$ are estimated using finite differences to provide smoother gradients compared to analytic gradients [S1]:
$$ \nabla f_{sdf}(x) \approx \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
where $\epsilon$ is the hash grid size [S1].

**Step 5: Rendering and Loss Computation**
Render the color $\hat{C}(x)$ using the fine samples. Compute the total loss:
$$ \mathcal{L} = \mathcal{L}_{color} + \lambda_1 \mathcal{L}_{eik} + \lambda_2 \mathcal{L}_{mask} $$
*   $\mathcal{L}_{color}$: Sum of L2 loss between rendered color $\hat{C}(x)$ and ground truth $C(x)$ over all sample points [S1].
*   $\mathcal{L}_{eik}$: Sum of squared difference between SDF gradient magnitude and 1: $\sum_x (\|\nabla f_{sdf}(x)\| - 1)^2$ [S1].
*   $\mathcal{L}_{mask}$: Binary Cross-Entropy (BCE) between the object mask $M_k$ and accumulated opacity $\hat{O}_k$: $\text{BCE}(M_k, \hat{O}_k)$ [S1].
*   Accumulated Opacity $\hat{O}_k = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i}$ [S1].

**Parameters:**
*   Hash Encoding Levels: 16 ($2^5$ to $2^{16}$) [S1].
*   Hash Feature Size: 2 per level [S1].
*   Geometry MLP: 1 layer, hidden size 64 [S1].
*   Appearance MLP: 2 layers, hidden size 64 [S1].
*   Learning Rate: 0.001 for both networks [S1].
*   Optimizer: AdamW, betas (0.9, 0.99) [S1].
*   Regularization Weights: $\lambda_1 = 0.1$, $\lambda_2 = 0.1$ [S1].
*   Progressive Training Update Step: 1000 iterations (hash levels updated every 1000 steps) [S1].

**Pitfalls:**
*   Using analytic gradients for normals in NeuS can be noisy; this method uses finite differences for smoother normals [S1].
*   If the occupancy grid resolution is too low, even with the nonlinear function, many invalid samples may still be introduced within large occupied cells [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to uniform sampling methods like NeuS because empty regions are skipped via the Occupancy Grid. The nonlinear function reduces the number of samples needed near the surface to achieve high quality.
*   **Memory Complexity:** Depends on the Occupancy Grid size and Hash Encoding size. The hash encoding uses 16 levels with feature size 2, which is memory efficient compared to dense voxel grids.
*   **Performance:** Achieves good surface reconstruction quality (Chamfer Distance) and novel view synthesis (PSNR) on DTU and NeRF Synthetic datasets [S1].
*   **Strengths:** Avoids invalid sampling in empty space (Occupancy Grid) and biases samples toward the surface (Nonlinear Function), leading to faster convergence and better detail recovery than NeuS or Instant-NGP [S1].
*   **Limitations:** Requires tuning of the hash grid levels and the nonlinear function parameters. The finite difference normal estimation adds computational overhead compared to analytic gradients but improves smoothness [S1].

## Background
This method was proposed by Siyuan He and Xinglin Liu from Wuyi University, published in March 2024 [S1]. It builds on Neural Radiance Fields (NeRF) and NeuS, aiming to address the long training times and inefficient sampling of previous works. It incorporates ideas from Instant-NGP (Occupancy Grid) and NeuS (SDF-based rendering) but introduces a novel nonlinear sampling function to improve efficiency [S1].

## Sources
[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
