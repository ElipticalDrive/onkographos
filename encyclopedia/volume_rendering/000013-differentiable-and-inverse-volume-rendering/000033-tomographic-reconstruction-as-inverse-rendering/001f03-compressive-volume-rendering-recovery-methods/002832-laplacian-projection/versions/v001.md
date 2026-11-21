# Laplacian Projection

Also called: Laplacian Regularization for Sparse View Reconstruction

## Overview
Laplacian Projection is a physics-based tomographic reconstruction method that employs Laplacian regularization to solve the inverse rendering problem for sparse view reconstruction, particularly for volumetric smoke [S1]. It distinguishes itself from neural priors (like NeRF variants) and Gaussian splatting by enforcing physical smoothness constraints on the density field via the Laplacian operator, serving as a state-of-the-art baseline for physics-based smoke reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input views (images) $I_i$ with associated camera poses $P_i$. The method supports both single-view ($N=1$) and multi-view ($N=16$) configurations [S1].
*   **Outputs:** A reconstructed 3D volume representation (density field $\sigma_\Theta$).

**Data Structures:**
The volume is represented on a discrete grid. For the standard experimental setup, the volume resolution is $128 \times 196 \times 128$ [S1]. The density field $\sigma_\Theta(x)$ is parameterized, potentially by a neural network $\Theta$ or voxel values, defined over the domain $[-1, 1]^3$ [S2].

**Regularization Formula:**
The core differentiating component is the Laplacian regularizer $R_{Lap}(\Theta)$, which penalizes high-frequency oscillations in the density field to ensure physical plausibility [S2]. It is defined as the integral of the absolute Laplacian of the density:

$$
R_{Lap}(\Theta) = \int_{[-1,1]^3} |\Delta_x \sigma_\Theta(x)| \, dx
$$

In discrete implementation, this is approximated by a summation over the grid cells:

$$
R_{Lap}(\Theta) \approx \sum_{i=1}^N \sum_{j=1}^N \sum_{k=1}^N |\Delta_x \sigma_\Theta(\delta i, \delta j, \delta k)| \delta^3
$$

where $\Delta = \text{div} \cdot \nabla$ denotes the Laplacian operator, $\delta$ is the grid spacing, and $N$ is the number of grid points per dimension [S2].

**Optimization Steps:**
1.  **Initialization:** Initialize the volume parameters $\Theta$ (e.g., neural network weights or voxel values) [S2].
2.  **Forward Projection:** Cast rays through the volume for each input view. For each ray, compute the line integral of the density to generate a rendered projection $\hat{p}$ [S1].
    *   *Note:* The source recommends using 30 directional samples for pre-computation as an optimal balance between accuracy and performance [S1].
3.  **Loss Calculation:** Compute the reconstruction loss $\mathcal{L}$ between the rendered projection $\hat{p}$ and the ground truth image $p$ (e.g., MSE).
4.  **Regularization Calculation:** Compute the Laplacian regularizer $R_{Lap}(\Theta)$ using the discrete approximation [S2].
5.  **Gradient Computation:** Compute the total gradient with respect to $\Theta$:
    $$
    \frac{\partial \mathcal{L}_{total}}{\partial \Theta} = \frac{\partial \mathcal{L}}{\partial \Theta} + \lambda \frac{\partial R_{Lap}}{\partial \Theta}
    $$
    where $\lambda$ is the regularization weight. The term $\frac{\partial \mathcal{L}}{\partial \Theta}$ is computed via back-propagation through the differentiable forward projection operator, utilizing the adjoint (back-projection) operator $\mathcal{P}^T$ [G].
6.  **Update:** Update $\Theta$ using an optimizer (e.g., Adam) [S1].
7.  **Iteration:** Repeat steps 2-6 for a specified number of iterations. The source investigates iteration limits of 250, 1000, and 5000 to analyze quality-efficiency trade-offs [S1].

**Parameters:**
*   **Directional Samples:** 30 (recommended optimal balance) [S1].
*   **Volume Resolution:** $128 \times 196 \times 128$ (standard experimental setup) [S1].
*   **Iteration Limits:** 250, 1000, 5000 (investigated) [S1].
*   **Input Views:** 1 (single-view) or 16 (multi-view) [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Views:** The method is specifically designed for sparse view scenarios. Performance degrades if the number of views is insufficient to constrain the Laplacian smoothness prior effectively [S1].
*   **Precision:** The discrete approximation of the Laplacian integral requires careful handling of grid boundaries and the Laplacian kernel definition to avoid artifacts [S2].

## Subtopics

### Laplacian Regularization
A specific regularization term $R_{Lap}$ that enforces smoothness by minimizing the integral of the absolute Laplacian of the density field. It is chosen over Total Variation or other priors to specifically target the physical continuity of smoke volumes. It is implemented via discrete summation of the Laplacian operator over the grid [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Proportional to the volume resolution ($128 \times 196 \times 128$) and the number of parameters in $\Theta$ [S1].
*   **Time:** Depends on the number of iterations and directional samples. The source identifies 30 directional samples as optimal for performance [S1]. Iteration counts of 250-5000 are used, implying significant computational cost per reconstruction [S1].

**Measured Performance:**
*   Evaluated on an NVIDIA GeForce RTX 2080 Ti GPU [S1].
*   Uses the ScalarFlow dataset (multi-view smoke sequences, 300×531 pixels) [S1].
*   Provides competitive performance against 3D Gaussian Splatting and other physics-based methods [S1].

**Strengths and Limitations:**
*   **Strengths:** State-of-the-art for physics-based smoke reconstruction; effective for sparse view inputs; physically plausible results [S1].
*   **Limitations:** Computationally intensive due to iterative optimization and ray marching; requires careful tuning of iteration limits and regularization weights [S1].

**When to Use:**
Use when reconstructing volumetric smoke or similar physically smooth media from sparse views where physical plausibility is prioritized over raw speed or neural network flexibility [S1].

## Background
Laplacian Projection is a physics-based tomographic reconstruction method. It was introduced as a baseline for comparison in studies evaluating differentiable smoke reconstruction, demonstrating its effectiveness as a state-of-the-art physics-based approach [S1]. The regularization term is derived from the mathematical property that the Laplacian operator measures the local deviation from the mean, enforcing smoothness [S2].

## Sources
[S1] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S2] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
