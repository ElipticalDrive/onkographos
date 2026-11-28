# Anchor-2D-GS / Anchor-3D-GS
Also called: Anchor-GS, modified versions of 2D-GS and 3D-GS

## Overview
Anchor-2D-GS and Anchor-3D-GS are modified variants of 2D-GS [15] and 3D-GS [5] that employ an anchor grid structure to organize Gaussian primitives more efficiently, reducing redundancy and storage overhead [S1]. They achieve competitive rendering quality with fewer file storage than original methods, demonstrating that the anchor design organizes primitives more efficiently [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse Structure-from-Motion (SfM) points.
*   **Output:** A set of 2D or 3D Gaussians organized within an anchor grid structure.

**Data Structures:**
The method voxelize the input SfM points to create anchors. Each anchor is assigned 2D or 3D Gaussians [S1]. The specific layout of the Gaussian attributes (position, covariance, opacity, color) follows the standard definitions of the parent method (2D-GS or 3D-GS), but their spatial distribution is constrained by the anchor grid [S1].

**Parameters:**
*   **Voxel size for the anchor grid:** 0.001 for all scenes in the modified anchor versions [S1].
*   **Training Iterations:** 40k iterations for standard scenes, with densification concluding at 20k iterations [S1]. For large-scale scenes, training is set to 100k iterations, with densification ending at 50k iterations [S1].
*   **Densification Interval ($T$):** 100 [S1].
*   **Visibility Threshold ($\tau_v$):** 0.7 for small-scale scenes [S1].

**Loss Functions:**
The method maintains the same densification strategy as Scaffold-GS [S1]. The specific loss functions depend on the base variant:
*   **For Anchor-3D-GS:** Standard L1 and SSIM loss, with weights set to 0.8 and 0.2, respectively [S1].
*   **For Anchor-2D-GS:** Retains the distortion loss $L_d = \sum_{i,j} \omega_i \omega_j |z_i - z_j|$ and normal loss $L_n = \sum_i \omega_i (1 - \mathbf{n}_i^T \mathbf{N})$, with weights set to 0.01 and 0.05, respectively [S1].

**Steps:**
1.  **Voxelization:** Voxelize the input SfM points to generate the anchor grid [S1].
2.  **Assignment:** Assign 2D or 3D Gaussians to each anchor [S1].
3.  **Training:** Optimize the Gaussian parameters using the specified loss functions and densification strategy [S1].

**Edge Cases and Pitfalls:**
*   The method assumes the input consists of SfM points.
*   Not all layers are fully densified during progressive training processes, which may require extending densification by an additional 10k iterations [S1].

## Subtopics

### Anchor Gaussian Handling
Anchor Gaussians are hypothesized to constrain and regularize parameters of coarser-level Gaussians, typically having larger scales than regular Gaussians [S2]. They do not have actual opacity for rendering; for visualization, their opacity is manually set to $\text{sigmoid}(1)$ [S2]. The average scale of anchors is $e^{-5.08}$ across 100 randomly sampled scenes, while the average scale of Gaussians is $e^{-7.12}$ [S2]. This size difference suggests that Gaussians adjust their scales to represent sharp details in real-world images, whereas anchor Gaussians serve a regularization role [S2].

## Operating characteristics
*   **Storage:** Achieves fewer file storage than original 2D-GS and 3D-GS methods [S1].
*   **Quality:** Competitive rendering quality compared to original methods [S1].
*   **Efficiency:** The anchor design creates a more compact representation by reducing redundancy [S1].
*   **Comparison:** While Anchor-2D/3D-GS reduce storage, Octree-GS delivers better visual performance and fewer Gaussian primitives due to its explicit, multi-level anchor design [S1].

## Background
The method was introduced in "Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians" [S1]. It represents a modification of existing 2D-GS [15] and 3D-GS [5] techniques to incorporate an anchor-based organization, building on the concepts explored in Scaffold-GS [3] regarding anchor-based representations.

## Sources
[S1] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S2] GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats, https://doi.org/10.52202/079017-2171, fetched 2026-10-29
