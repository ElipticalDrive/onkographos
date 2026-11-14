# Efficient Multi-View Training for 3D Gaussian Splatting

Also called: Multi-View Mini-Batch Training for 3DGS, Thread-Efficient Partial Rendering for Multi-View 3DGS

## Overview
This method enables multi-view mini-batch training for 3D Gaussian Splatting (3DGS) to reduce gradient variance and stabilize optimization, addressing the limitation that standard 3DGS rasterization forces single-view processing [S1, unofficial]. It differs from standard 3DGS by allowing multiple images to contribute to a single gradient update, while overcoming the naive multi-view overhead through a thread-efficient partial rendering pipeline and adapting the Adaptive Density Control (ADC) mechanism to handle multi-plane gradients [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with corresponding camera poses; 3D Gaussian primitives defined by mean $\mu$, scale $S$, rotation $R$, opacity $o$, and spherical harmonic coefficients for color $c$.
*   **Outputs:** Updated Gaussian parameters $w_{t+1}$ and rendered images.

**Data Structures:**
The rasterizer organizes computation using a tile-based structure [S1, unofficial]:
*   **Tiles:** The image plane is split into fixed-size tiles.
*   **Blocks:** A single thread block is allocated to a single tile.
*   **Threads:** Threads within a block cooperate to preprocess Gaussians and render pixels for their assigned tile. A thread is freed only when all peer threads in the same block complete their work [S1, unofficial].

**Method Steps:**

1.  **Mini-Batch Selection:** Randomly sample a mini-batch $B_k$ containing pixels from multiple distinct training images (multi-view), rather than a single image [S1, unofficial].
2.  **Thread-Efficient Partial Rendering:**
    *   For each image in the mini-batch, identify the subset of Gaussians that project into the image's tiles (culling) [S1, unofficial].
    *   Project 3D Gaussians to 2D using the camera projection matrix. The projected 2D Gaussian $G_i(p)$ for pixel $p$ is computed as:
        $$ G_i(p) = \exp\left(-\frac{1}{2}(p - \mu'_i)^T (\Sigma'_i)^{-1} (p - \mu'_i)\right) $$
        where $\mu'$ and $\Sigma'$ are the projected 2D mean and covariance derived from the 3D parameters [S1, unofficial].
    *   Render the color $\hat{C}(p)$ for each pixel $p$ in the mini-batch using alpha-blending of sorted Gaussians $\{G_j\}_{j=1}^M$:
        $$ \hat{C}(p) = \sum_{i=1}^{M} c_i T_i o_i G_i(p) $$
        where $T_i = \prod_{k=1}^{i-1} (1 - o_k G_k(p))$ is the accumulated transmittance [S1, unofficial].
    *   **Optimization:** Unlike naive partial rendering which causes idle threads when peer threads in a block finish early, this method ensures threads are utilized efficiently across the multi-view batch to mitigate overhead [S1, unofficial].
3.  **Loss Computation:** Compute the loss $\phi_i(w)$ for each pixel using a differentiable loss function $L$ (e.g., L1 or D-SSIM) between the rendered color $\hat{C}(p)$ and the ground truth $C_i$ [S1, unofficial].
4.  **Gradient Aggregation:** Compute the mini-batch gradient $\nabla \bar{\phi}_k$ by averaging the gradients of all pixels in $B_k$:
    $$ \nabla \bar{\phi}_k (w_t) = \frac{1}{|B_k|} \sum_{i \in B_k} \nabla \phi_i (w_t) $$
    where $\phi_i(w) = L(w, r_i, c_i)$ and $r_i$ is the ray for pixel $i$ [S1, unofficial].
5.  **Parameter Update:** Update the weights $w$ using the learning rate $\eta_t$:
    $$ w_{t+1} = w_t - \eta_t \nabla \bar{\phi}_k (w_t) $$
    [S1, unofficial].
6.  **Adaptive Density Control (ADC) Adaptation:**
    *   Standard ADC assumes all 2D positional gradients reside within a single image plane [S1, unofficial].
    *   In multi-view training, gradients come from different image planes. The method must adapt the densification logic to avoid performing arithmetic between vectors in different spaces, which leads to suboptimal densification [S1, unofficial].
    *   *Note:* The specific implementation details of the adapted ADC are not fully specified in the provided notes, only that the standard assumption is invalid and a new approach is required [S1, unofficial].

**Parameters:**
*   $\eta_t$: Learning rate. Typical values are not given in the sources.
*   $K$: Number of sampled mini-batches used for estimating gradient variance via Monte Carlo sampling during analysis [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Idle Threads:** Naive multi-view rendering incurs significant overhead because the tile-based rasterizer allocates one block per tile. If some threads finish early, peer threads wait, causing idle time [S1, unofficial].
*   **Gradient Space Mismatch:** Applying standard ADC in multi-view mode risks arithmetic operations between gradient vectors from different image planes, resulting in suboptimal densification [S1, unofficial].

## Operating characteristics
*   **Gradient Variance:** Multi-view mini-batch training yields a smaller variance of mini-batch gradients compared to single-view training, leading to more stable optimization [S1, unofficial].
*   **Overhead:** Naive multi-view rendering has considerable overhead due to the rasterization pipeline's design. The thread-efficient partial rendering mitigates this, but specific performance metrics (FPS, memory usage) are not provided in the sources [S1, unofficial].
*   **Scalability:** The method allows leveraging the stability benefits of multi-view learning (common in NeRF) for 3DGS, which traditionally suffers from higher gradient variance due to single-view updates [S1, unofficial].

## Background
*   **Origin:** Proposed by Minhyuk Choi, Injae Kim, and Hyunwoo J. Kim from Korea University and KAIST [S1, unofficial].
*   **History:** Published as an arXiv preprint in June 2025 [S1, unofficial].
*   **Context:** Standard 3DGS uses single-view mini-batches because its rasterization pipeline makes gathering individual pixels from multiple images difficult. NeRF, in contrast, uses multi-view mini-batches. This work bridges that gap for 3DGS [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
