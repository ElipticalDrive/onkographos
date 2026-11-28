# FastGS

Also called: FastGS: Training 3D Gaussian Splatting in 100 Seconds

## Overview
FastGS is a general acceleration framework for 3D Gaussian Splatting (3DGS) that significantly reduces training time by regulating the number of Gaussians based on multi-view consistency, dispensing with explicit budgeting mechanisms [S2, unofficial]. It sets itself apart from sibling methods like [Taming-3DGS](4111-physical-parameter-optimization.md) and [Speedy-Splat](6088-non-pinhole-camera-training.md) by evaluating Gaussian importance based on actual multi-view reconstruction quality rather than Gaussian-associated properties or indirect gradient signals [S2, unofficial]. It also incorporates dynamic sparse rasterization and fast gradient checkpointing to optimize the training loop [S1, unofficial].

## How it works
FastGS re-engineers the 3DGS training loop to finish optimization faster. The method combines a multi-view consistent densification and pruning strategy with rasterization and memory optimizations [S1, unofficial][S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Sparse Structure-from-Motion (SfM) point cloud for initialization; multi-view images for training.
*   **Outputs:** Optimized 3D Gaussian primitives (means, covariances, opacities, spherical harmonic coefficients).

### Data Structures
**3D Gaussian Primitive ($G_i$):**
Each primitive is parameterized by:
*   Mean $\mu_i \in \mathbb{R}^3$
*   Rotation $r_i \in \mathbb{R}^4$ (quaternion)
*   Scale $s_i \in \mathbb{R}^3$
*   Opacity $\sigma_i \in \mathbb{R}$
*   Color coefficients $c_i \in \mathbb{R}^{16 \times 3}$ (view-dependent spherical harmonics, degree 3) [S2, unofficial].

The 3D covariance matrix is defined as $\Sigma_i^{3D} = R_i S_i S_i^\top R_i^\top$, where $R_i$ is the rotation matrix derived from $r_i$ and $S_i$ is the diagonal scale matrix [S2, unofficial].

**128-bit Splat Data Structure (Memory Optimization):**
To utilize optimized memory load instructions, splat attributes are restructured into 128-bit aligned vectors [S3]:
1.  **Color:** Padded from a 96-bit 3D float vector (`vec3`) to a 128-bit 4D float vector (`vec4`) [S3].
2.  **Bounding Box:** 4D unsigned integer vector (`uvec4`) using 16-bit unsigned integers, totaling 64 bits [S3].
3.  **Screen Position:** 2D float vector (`vec2`), totaling 64 bits [S3].
The Bounding Box and Screen Position are packed alongside the padded Color to form a single 128-bit structure. This trades additional compute instructions for unpacking data for reductions in memory operation instructions [S3].

### Rendering and Projection
Given a viewing transformation $W$, the covariance matrix in camera coordinates is computed as [S2, unofficial]:
$$ \Sigma'_i^{3D} = J W \Sigma_i^{3D} W^\top J^\top $$
where $J$ is the Jacobian of the affine approximation of the projective transformation [S2, unofficial].

The projected 2D Gaussian contribution to pixel $p$ is:
$$ G'_i(p) = \exp\left(-\frac{1}{2} (p - \mu_i^{2D})^\top \Sigma_i^{2D^{-1}} (p - \mu_i^{2D})\right) $$
where $\mu_i^{2D}$ and $\Sigma_i^{2D}$ are the mean and covariance of the projected 2D Gaussian, obtained by marginalizing $\Sigma'_i^{3D}$ along the viewing direction [S2, unofficial].

The accumulated color of pixel $p$ is computed via $\alpha$-blending for Gaussians $\{G_i\}_{i=1}^N$ sorted by depth:
$$ C(p) = \sum_{i \in \mathcal{N}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $\alpha_i = \sigma_i G'_i(p)$ [S2, unofficial].

### Training Steps
1.  **Initialization:** Initialize 3D Gaussians using SfM point clouds [S2, unofficial].
2.  **Sampling:** Sample training views and generate corresponding per-pixel L1 loss maps between the rendered image and the ground truth [S2, unofficial].
3.  **Multi-View Score Computation:** For each sampled view, compute a multi-view score for each Gaussian. This score is defined as the average number of high-error pixels within the Gaussian's 2D footprint across the sampled views [S2, unofficial]. High-error pixels are identified solely from the per-pixel L1 loss [S2, unofficial].
4.  **Multi-View Consistent Densification (VCD):** Use the multi-view score to guide densification. Gaussians with high scores (indicating they contribute to high-error regions across multiple views) are densified [S2, unofficial].
5.  **Multi-View Consistent Pruning (VCP):** Use the multi-view score to guide pruning. Gaussians with low scores (useless to multi-view rendering quality) are removed [S2, unofficial].
6.  **Rasterization Optimization:** Employ dynamic sparse rasterization that skips tiles with negligible transmittance [S1, unofficial].
7.  **Gradient Checkpointing:** Use a fast gradient checkpointing scheme that recomputes only the necessary intermediate values during backpropagation [S1, unofficial].

### Parameters
*   **Training Time:** Approximately 100 seconds for static scenes (e.g., Tanks & Temples train scene) [S2, unofficial].
*   **High-Error Threshold:** Not given in the sources.
*   **Densification/Pruning Thresholds:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Redundancy:** Existing methods using Gaussian-associated scores (e.g., Taming-3DGS) lead to excessive growth of redundant Gaussians because they do not leverage actual contribution to rendering quality [S2, unofficial].
*   **Quality Degradation:** Existing methods using gradients (e.g., Speedy-Splat) lead to significant degradation in rendering quality because they indirectly enforce multi-view consistency [S2, unofficial].
*   **Memory Efficiency:** Using the original 96-bit 3D float vector for color does not have an optimized load instruction, leading to inefficient memory access [S3].

## Subtopics

### [Accelerated Training for 3DGS](9879-accelerated-training-for-3dgs.md)
A strategy to improve training time efficiency by separating higher spherical harmonic (SH) bands from the 0th band within the rasterizer, reducing the number of updates for higher SH bands compared to diffused colors [9879-accelerated-training-for-3dgs.md]. It sets itself apart from standard 3D Gaussian Splatting by decoupling the update frequency of view-dependent color variations from the base color, and from FastGS by focusing on rasterizer-level optimization rather than gradient checkpointing or dynamic sparse rasterization [9879-accelerated-training-for-3dgs.md].

## Operating characteristics
*   **Time Complexity:** Significantly reduced training time. FastGS achieves a 3.29× acceleration compared to state-of-the-art methods on the Mip-NeRF 360 dataset [S2, unofficial]. It achieves a 15.45× acceleration compared to vanilla 3DGS on the Deep Blending dataset [S2, unofficial].
*   **Memory Complexity:** The 128-bit splat data structure increases memory usage slightly (padding color to 4D) but reduces memory operation instructions [S3].
*   **Measured Performance:**
    *   Static scenes: ~100 seconds for Tanks & Temples train scene [S2, unofficial].
    *   Dynamic scene reconstruction: 2.82× faster [S2, unofficial].
    *   Surface reconstruction: 2.24× faster [S2, unofficial].
*   **Strengths:** General applicability across static, dynamic, surface, sparse-view, large-scale reconstruction, and SLAM tasks [S2, unofficial]. No budgeting mechanism required [S2, unofficial].
*   **Limitations:** Relies on multi-view consistency, which may be less effective in extremely sparse-view scenarios where multi-view overlap is minimal (though it is applied to sparse-view reconstruction [S2, unofficial]).
*   **When to use:** When training time is the primary bottleneck and rendering quality must be maintained comparable to vanilla 3DGS.
*   **When not to use:** When hardware lacks optimized 128-bit load instructions (negating the memory optimization benefit) or when multi-view consistency cannot be reliably established.

## Background
FastGS was proposed to address the failure of dominant 3DGS acceleration methods to properly regulate the number of Gaussians during training, which causes redundant computational time overhead [S2, unofficial]. It introduces a novel, simple, and general acceleration framework that fully considers the importance of each Gaussian based on multi-view consistency [S2, unofficial]. The method was published in December 2025 [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
[S3] Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting, https://doi.org/10.1145/3820019, fetched 2026-11-01
