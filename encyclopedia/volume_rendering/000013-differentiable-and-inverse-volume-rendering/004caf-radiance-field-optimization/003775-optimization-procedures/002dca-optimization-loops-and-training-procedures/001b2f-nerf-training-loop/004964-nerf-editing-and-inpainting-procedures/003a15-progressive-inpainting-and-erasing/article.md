# Progressive Inpainting-and-Erasing
Also called: Conflicted Geometry Detection and Erasing, Inpainting-and-Erasing Strategy, PERF, PERF Algorithm

## Overview
Progressive Inpainting-and-Erasing (PERF) is a method for reconstructing a panoramic Neural Radiance Field (NeRF) from a single input panorama by iteratively expanding the training viewpoint coverage. It solves the problem of geometry conflicts that arise when inpainting invisible regions in novel views, which often violate the consistent geometry established by the initial reference view. Unlike methods that rely on multi-view consistency constraints or external semantic priors, PERF explicitly detects and erases geometrically inconsistent regions from the training supervision, ensuring that the learned 3D structure remains coherent across all added views [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single RGB panorama image $I_{\text{rgb}}$.
*   **Output:** A panoramic NeRF representation $f_{\text{nerf}}$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $c$.

**Data Structures:**
*   **NeRF Representation:** The method uses Instant-NGP [S1]. The data structure consists of a hash table of size $2^{19}$ with 16 levels of feature vectors, where each vector has a channel size of 2 [S1].
*   **Reference Views Set ($I_{\text{ref}}$):** A set containing the initial RGBD panorama and subsequently added inpainted RGBD panoramas.
*   **Trajectories ($\{p_k\}$):** A pre-generated set of novel camera poses sampled from the depth map, used to select new viewpoints.

**Formulas:**
1.  **Initial Depth Estimation:** An initial depth map $I_d$ is estimated from $I_{\text{rgb}}$ [S1].
2.  **Invisible Region Mask ($m'$):** For a new view, invisible regions are detected by warping depth from the new view to a reference view. If the warped depth is significantly larger than the reference depth, the point is considered invisible (occluded) in the reference view.
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v'_{ij}) - v_{ij} > \epsilon) $$
    where $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth from the new view, $\epsilon$ is a small threshold, and $\mathbb{I}(\cdot)$ is the indicator function [S1].
3.  **Conflicted Region Mask ($m''$):** Conflicts occur when inpainted regions in a new view contradict the geometry of existing reference views. This is detected by warping depth from the reference view to the new view. If the reference depth is significantly larger than the warped depth, the region is inconsistent.
    $$ m''_{ij} = \mathbb{I}(v_{ij} - \text{warp}(v'_{ij}) > \epsilon) $$
    where $v_{ij}$ is the reference depth and $v'_{ij}$ is the warped depth [S1].
4.  **Final Conflict Mask:** The final mask of conflicted regions is computed by performing a logical AND operation across the individual conflict masks $m''$ from all views in $I_{\text{ref}}$ [S1].

**Parameters:**
*   $\epsilon$: A small value used as a threshold for depth difference in warping operations [S1].
*   Hash table size: $2^{19}$ [S1].
*   Feature levels: 16 [S1].
*   Feature channel size: 2 [S1].
*   Initial training iterations: 10,000 iterations for the single input panorama [S1].
*   New view sampling: 32 new positions with approximately gradually increased distance from the original position [S1].
*   Fine-tuning iterations per new view: 2,000 iterations total [S1].
*   Optimizer: Adam [S1].
*   Learning rate: Initially $10^{-3}$, decayed to $10^{-4}$ with cosine annealing strategy for each learning stage of a newly sampled view [S1].

**Algorithm Steps:**
1.  **Initialization:**
    *   Estimate the initial depth map $I_d$ of the input single panorama $I_{\text{rgb}}$ to obtain an RGBD panorama $I_{\text{rgbd}}$ [S1].
    *   Train a single-view NeRF $f_{\text{nerf}}$ using $I_d$ and $I_{\text{rgb}}$ for 10,000 iterations [S1].
    *   Generate a trajectory set $\{p_k\}$ of novel views based on the depth map $I_d$ [S1].
    *   Initialize the set of reference views $I_{\text{ref}} = \{I_{\text{rgbd}}\}$ [S1].

2.  **Iterative Expansion:**
    *   Iterate $n$ from 1 to `iters` (total number of new views to add):
        *   Randomly sample a new camera view pose $p$ from the trajectory set $\{p_k\}$ [S1].
        *   **Invisible Region Detection:** Compute the invisible regions mask $m'$ for the new view relative to the current reference views using the warping formula (Eq. 4) [S1].
        *   **Collaborative RGBD Inpainting:** Perform inpainting on the invisible regions of the new view. Use Stable Diffusion for RGB inpainting and a depth estimator for depth stitching, guided by the mask $m'$ [S1]. This generates a completed panorama for the new view.
        *   **Conflict Detection and Erasing:**
            *   For each reference view $I_{\text{ref}} \in I_{\text{ref}}$:
                *   Compute the conflicted region mask $m''$ between the reference view and the newly inpainted view using the warping formula (Eq. 6) [S1].
                *   Erase these conflicted regions from the supervision of the NeRF training for this reference view [S1].
            *   Compute the final conflict mask by performing a logic AND operation across all individual masks $m''$ from all reference views [S1].
        *   **Update Reference Set:** Add the newly inpainted and erased view to the reference views set $I_{\text{ref}}$ [S1].
        *   **Fine-tuning:** Fine-tune $f_{\text{nerf}}$ with the updated $I_{\text{ref}}$.
            *   Total iterations: 2,000 [S1].
            *   First 100 iterations: Supervision uses only the new reference view [S1].
            *   Remaining 1,900 iterations: Supervision uses all reference views in $I_{\text{ref}}$ [S1].
            *   During each iteration, randomly sample 2,048 rays with ground-truth or inpainted colors and depth values to supervise the NeRF [S1].

**Edge Cases and Pitfalls:**
*   **Depth Estimation Error:** The method relies on the accuracy of the initial depth estimator and the depth estimator used during inpainting. Errors here can lead to incorrect warping and false conflict detection [S1].
*   **Occlusion Handling:** The warping logic assumes that if a point is visible in the new view but its warped depth is larger than the reference depth, it is occluded in the reference view. This may fail in cases of complex self-occlusion or non-Lambertian surfaces if the depth estimator is unreliable [S1].
*   **Convergence:** The algorithm iterates until a fixed number of views (`iters`) are added. Convergence of the NeRF geometry is assumed to be achieved through the iterative fine-tuning and erasing process [S1].

## Subtopics

### Collaborative RGBD Inpainting
A sub-procedure that fills invisible regions in novel views using Stable Diffusion for color and a depth estimator for geometry, guided by visibility masks. It differs from standard inpainting by explicitly handling the RGBD consistency required for NeRF training. Choose this when inpainting is needed to expand viewpoint coverage from sparse inputs. [Link to specific inpainting method if available, otherwise note as component of PERF].

### Conflicted Geometry Detection
A mechanism using bidirectional depth warping to identify regions where inpainted geometry contradicts existing reference views. It sets apart PERF from methods that simply add new views without consistency checks. Choose this when preventing geometric artifacts from inpainted regions is critical. [Link to specific detection method if available, otherwise note as component of PERF].

## Operating characteristics
*   **Time Complexity:** The method is iterative, with complexity proportional to the number of new views added (`iters`) and the cost of NeRF fine-tuning per view. Each iteration involves depth warping, inpainting (which can be computationally expensive depending on the Stable Diffusion model), and 2,000 iterations of NeRF optimization [S1].
*   **Memory Complexity:** Depends on the Instant-NGP hash table size ($2^{19}$ entries) and the storage of reference views. The hash table uses 16 levels of feature vectors with channel size 2 [S1].
*   **Measured Performance:** Training a scene typically takes about 2 hours on a single Nvidia V100 GPU with 32G memory [S1].
*   **Strengths:** Effectively avoids geometry conflicts between visible and invisible regions, leading to consistent 3D geometry. It allows for the reconstruction of panoramic NeRFs from a single input.
*   **Limitations:** Relies on the quality of depth estimation and inpainting. The iterative process can be slow. It is designed for panoramic views, which may limit applicability to standard perspective views without modification [S1].
*   **When to use:** When reconstructing a NeRF from a single panorama and ensuring geometric consistency across expanded viewpoints is a priority.
*   **When not to use:** When computational resources are limited and faster, non-iterative methods are preferred, or when the input is not a panorama.

## Background
Progressive Inpainting-and-Erasing (PERF) was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. The method addresses the challenge of learning a NeRF from a single panoramic image by progressively expanding the training viewpoint coverage. It was developed to handle the geometry conflicts that arise when inpainting invisible regions in novel views, which often violate the consistent geometry established by the initial reference view. The method uses Instant-NGP for efficient NeRF representation and Stable Diffusion for inpainting [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
