# Efficient Multi-View Training for 3D Gaussian Splatting

Also called: Multi-View Mini-Batch Training for 3DGS, Thread-Efficient Partial Rendering for Multi-View 3DGS

## Overview
This method enables stable optimization of 3D Gaussian Splatting (3DGS) by training on multiple views per iteration, reducing the variance of mini-batch gradients compared to the standard single-view approach [S1, unofficial]. It overcomes the inherent overhead of the tile-based rasterizer through thread-efficient partial rendering and resolves the incompatibility of standard Adaptive Density Control (ADC) with multi-view data by introducing a 3D-space compatible densification mechanism [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with corresponding camera poses; current 3D Gaussian parameters (position $\mu$, scale $S$, rotation $R$, opacity $o$, SH coefficients $c$); learning rate $\eta_t$.
*   **Outputs:** Updated Gaussian parameters; rendered images.

**Data Structures:**
The rasterizer partitions the image plane into tiles. A single thread block is allocated to a single tile. Threads within a block cooperate to preprocess Gaussians and render pixels for that tile. A thread is freed only when all peer threads in the same block have completed their work [S1, unofficial].

**Method Steps:**

1.  **Mini-batch Selection:** Randomly sample a subset of pixels from $M$ different training images to form a mini-batch $B_k$. Unlike standard 3DGS which processes one image entirely, this method aggregates pixels from multiple views [S1, unofficial].

2.  **Thread-Efficient Partial Rendering:**
    *   Naive multi-view rendering incurs significant idle threads because the tile-based architecture allocates one block per tile; rendering multiple images creates synchronization barriers where threads wait for peers in other images' tiles [S1, unofficial].
    *   The method implements a partial rendering scheme that maps pixels from multiple source images to a unified rendering target or processes them in a way that minimizes idle time. Specifically, it avoids the overhead of rendering every pixel of all images in a single iteration by only processing the sampled pixels efficiently [S1, unofficial].
    *   *Note:* The exact algorithm for mapping multi-view pixels to the rasterizer's thread blocks to eliminate idle threads is not fully detailed in the provided text, but the goal is to prevent the synchronization stalls seen in naive implementations [S1, unofficial].

3.  **Forward Pass:**
    For each pixel $p$ in the mini-batch, compute the rendered color $\hat{C}(p)$ by alpha-blending sorted Gaussians $\{G_j\}_{j=1}^M$ projected to 2D [S1, unofficial]:
    $$ \hat{C}(p) = \sum_{i=1}^{M} c_i T_i o_i G_i(p) $$
    where the transparency $T_i$ is:
    $$ T_i = \prod_{k=1}^{i-1} (1 - o_k G_k(p)) $$
    and the 2D Gaussian contribution $G_i(p)$ is:
    $$ G_i(p) = \exp\left(-\frac{1}{2}(p - \mu'_i)^T (\Sigma'_i)^{-1} (p - \mu'_i)\right) $$
    Here, $\mu'$ and $\Sigma'$ are the projected 2D mean and covariance derived from the 3D mean $\mu$, scale $S$, rotation $R$, color $c$, and opacity $o$ [S1, unofficial].

4.  **Loss Computation:**
    Compute the loss between the rendered color $\hat{C}(p)$ and the ground truth color $C(p)$. The method proposes a new D-SSIM loss that can be computed in 3D space to be compatible with the new rendering mechanism, replacing the standard 2D D-SSIM loss which assumes single-view plane alignment [S1, unofficial].

5.  **Backward Pass and Gradient Aggregation:**
    Compute gradients $\nabla \phi_i$ for each pixel in the mini-batch. The mini-batch gradient is the average:
    $$ \nabla \bar{\phi}_k (w_t) = \frac{1}{|B_k|} \sum_{i \in B_k} \nabla \phi_i (w_t) $$
    where $w_t$ are the weights at iteration $t$ [S1, unofficial].

6.  **Adaptive Density Control (ADC) Adjustment:**
    Standard ADC assumes all 2D positional gradients reside within a single image plane. Multi-view training risks performing arithmetic between vectors in different spaces (different image planes), leading to suboptimal densification [S1, unofficial].
    *   The method modifies the densification process to operate in 3D space or aligns gradients appropriately before arithmetic operations, ensuring that splitting and cloning decisions are valid across multiple views [S1, unofficial].

7.  **Parameter Update:**
    Update the weights using the learning rate $\eta_t$:
    $$ w_{t+1} = w_t - \eta_t \nabla \bar{\phi}_k (w_t) $$
    [S1, unofficial].

**Parameters:**
*   $\eta_t$: Learning rate. Typical values are not specified in the notes, but standard 3DGS uses adaptive optimizers like Adam [S1, unofficial].
*   $K$: Number of sampled mini-batches used for estimating variance via Monte Carlo sampling during analysis. Not a training hyperparameter but used for evaluation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Idle Threads:** Naive multi-view rendering causes significant overhead due to thread synchronization in the tile-based rasterizer. This is mitigated by the thread-efficient partial rendering scheme [S1, unofficial].
*   **Gradient Space Mismatch:** Standard ADC fails in multi-view settings because it assumes gradients are coplanar. The method must ensure gradients are handled in a consistent space (e.g., 3D) to avoid invalid densification [S1, unofficial].

## Operating characteristics
*   **Variance Reduction:** Multi-view mini-batch training yields a smaller variance of mini-batch gradients compared to single-view training, leading to more stable optimization. This trend is observed for both NeRF and 3DGS [S1, unofficial].
*   **Overhead:** Standard 3DGS rasterization makes gathering individual pixels from multiple images difficult. Naive multi-view rendering requires rendering every pixel of all images in a single iteration, incurring considerable overhead. The proposed method reduces this overhead through thread-efficient partial rendering [S1, unofficial].
*   **Performance:** The method enables multi-view training which was previously difficult for 3DGS due to its rasterization pipeline, unlike NeRF which naturally supports multi-view mini-batches [S1, unofficial].

## Background
*   **Origin:** Proposed by Minhyuk Choi, Injae Kim, and Hyunwoo J. Kim from Korea University and KAIST [S1, unofficial].
*   **History:** Published as arXiv:2506.12727v2 on June 17, 2025 [S1, unofficial].
*   **Context:** Standard 3DGS training uses single-view mini-batches because its tile-based rasterizer is optimized for single-image rendering. NeRF uses multi-view mini-batches. This work bridges that gap for 3DGS by addressing rasterization overhead and ADC incompatibility [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
