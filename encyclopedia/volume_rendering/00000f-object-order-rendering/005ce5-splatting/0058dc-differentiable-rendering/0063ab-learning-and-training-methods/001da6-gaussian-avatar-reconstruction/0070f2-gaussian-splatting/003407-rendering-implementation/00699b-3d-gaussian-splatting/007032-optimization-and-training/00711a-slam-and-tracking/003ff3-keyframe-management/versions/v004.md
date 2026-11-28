# Keyframe Management

Also called: Keyframing

## Overview
Keyframe Management maintains a sparse sliding window $W_k$ of keyframes to enable efficient joint optimization of camera poses and 3D Gaussian attributes in online SLAM systems [S1, unofficial]. It solves the problem of computational infeasibility when processing full video streams by discarding redundant frames with low overlap while ensuring sufficient parallax for robust pose estimation [S1, unofficial]. Unlike fixed-interval strategies, this method uses explicit covisibility metrics and geometric displacement to dynamically insert and remove keyframes [S1, unofficial].

## How it works
**Inputs:** Current camera frame $i$, last keyframe $j$, current keyframe window $W_k$, median scene depth $d_{med}$.
**Outputs:** Updated window $W_k$, boolean flag indicating if frame $i$ is registered as a new keyframe.

**Data Structures:**
*   $W_k$: A list of keyframe indices.
*   Visible Gaussians: A set of Gaussian indices marked as visible during rasterization. A Gaussian is visible if it is used in rasterization and the accumulated alpha $\alpha$ along the ray has not reached 0.5 [S1, unofficial].

**Steps:**
1.  **Compute Covisibility:** Calculate the Intersection over Union (IoU) of visible Gaussians between the current frame $i$ and the last keyframe $j$. Let $V_i$ be the set of visible Gaussians in frame $i$ and $V_j$ be the set in frame $j$. The covisibility score $C_{ij}$ is:
    $$ C_{ij} = \frac{|V_i \cap V_j|}{|V_i \cup V_j|} $$
    [S1, unofficial]

2.  **Compute Geometric Displacement:** Calculate the relative translation magnitude $t_{ij} = \|T_i - T_j\|$ between the current frame and the last keyframe [S1, unofficial].

3.  **Keyframe Registration:** Register frame $i$ as a new keyframe if either condition is met:
    *   $C_{ij} < \tau_{cov}$ (covisibility drops below threshold $\tau_{cov}$)
    *   $t_{ij} > \tau_{geo} \cdot d_{med}$ (relative translation is large with respect to median depth $d_{med}$)
    [S1, unofficial]

4.  **Keyframe Removal:** If a new keyframe is added, remove the oldest keyframe $k \in W_k$ if its overlap coefficient with the latest keyframe drops below a threshold. The overlap coefficient is defined similarly to covisibility but measures spatial overlap between keyframes [S1, unofficial].

5.  **Gaussian Insertion:** Upon registering a new keyframe, initialize new Gaussians. If depth is available, back-project depth measurements to initialize means $\mu_W$. In monocular cases, render depth and initialize Gaussians around low-variance depth estimates [S1, unofficial].

**Parameters:**
*   $\tau_{cov}$: Covisibility threshold. Not given in the sources.
*   $\tau_{geo}$: Geometric displacement threshold factor. Not given in the sources.
*   Overlap threshold for removal: Not given in the sources.

**Edge Cases:**
*   **Occlusion:** Handled naturally by the visibility ordering of 3D Gaussians along the camera ray; occluded Gaussians are not counted in $V_i$ [S1, unofficial].
*   **Degenerate Input:** If median depth $d_{med}$ is zero or undefined, the geometric displacement check should be skipped or handled with a fallback threshold [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of visible Gaussians for covisibility calculation. Joint optimization is limited to the small window $W_k$, reducing complexity from $O(N)$ to $O(|W_k|)$ where $N$ is total frames [S1, unofficial].
*   **Memory Complexity:** Proportional to the size of $W_k$ and the number of Gaussians visible in those frames [S1, unofficial].
*   **Strengths:** Prevents catastrophic forgetting; maintains multiview constraints via wide-baseline keyframes; handles occlusions without extra heuristics [S1, unofficial].
*   **Limitations:** Relies on accurate covisibility estimation; performance degrades if covisibility thresholds are poorly tuned [S1, unofficial].
*   **When to use:** Online SLAM systems with limited computational resources [S1, unofficial].
*   **When not to use:** Offline batch processing where all frames can be optimized jointly [S1, unofficial].

## Background
Originated in Gaussian Splatting SLAM (2023) [S1, unofficial]. It adapts keyframe management heuristics from Direct Sparse Odometry (DSO) [S1, unofficial] to the context of 3D Gaussian Splatting, leveraging the inherent visibility ordering of splatted Gaussians.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
