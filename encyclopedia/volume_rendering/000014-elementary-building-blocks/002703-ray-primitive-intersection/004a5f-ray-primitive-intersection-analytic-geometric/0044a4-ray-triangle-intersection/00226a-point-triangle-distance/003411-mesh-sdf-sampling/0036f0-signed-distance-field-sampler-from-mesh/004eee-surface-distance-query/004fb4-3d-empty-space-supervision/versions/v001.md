# 3D Empty Space Supervision

Also called: 3D emptiness loss, object field 3D supervision

## Overview
3D Empty Space Supervision is a regularization technique for neural radiance fields that uses estimated surface distances to identify empty, surface, and behind-surface points along a ray, thereby supervising the non-occupied object code dimensions [S1, unofficial]. It sets itself apart from sibling methods by explicitly deriving "emptiness" and "surfaceness" scores from the volume rendering integration to penalize solid object codes in free space, complementing 2D pixel-level supervision [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $K$: Number of sample points along a query light ray.
    *   $\sigma_i$: Learned density at sample point $i$ ($i=1 \dots K$).
    *   $\delta_k$: Distance between sample point $k$ and $k+1$ (or distance from camera to $k$ if defined differently, but context implies interval length).
    *   $d_k$: Distance from camera center to the $k$-th sample point.
    *   $\Delta d$: Hyperparameter for distance inaccuracy compensation.
    *   $o_k^{H+1}$: The last dimension of the object code vector for the $k$-th point (emptiness indicator).
    *   $o_k^h$: The $h$-th dimension of the object code vector for the $k$-th point ($h=1 \dots H$, solid object indicators).
*   **Outputs:**
    *   Loss values $\ell_{3d\_empty}$ and $\ell_{3d\_obj}$ used to update the object field MLP.

**Data Structures**
*   **Object Code:** A one-hot vector $\boldsymbol{o} \in \mathbb{R}^{H+1}$ associated with each 3D query point. The last dimension ($H+1$) represents "empty" or "non-occupied," while the first $H$ dimensions represent specific solid object classes [S1, unofficial].

**Steps**
1.  **Estimate Surface Distance ($d$):** Compute the approximate distance from the camera center to the surface hit point along the query ray using the volume rendering equation [S1, unofficial].
2.  **Compute Surfaceness Score ($s_k$):** For each sample point $k$, calculate how close it is to the estimated surface distance $d$ [S1, unofficial].
3.  **Compute Emptiness Score ($e_k$):** For each sample point $k$, determine if it lies in empty space relative to the surface, accounting for distance inaccuracy [S1, unofficial].
4.  **Calculate 3D Emptiness Loss ($\ell_{3d\_empty}$):** Supervise the last dimension of the object code ($o^{H+1}_k$) using the emptiness and surfaceness scores [S1, unofficial].
5.  **Calculate 3D Object Loss ($\ell_{3d\_obj}$):** Push the first $H$ dimensions of the object code ($o^h_k$) to zero in empty space [S1, unofficial].

**Formulas**
The approximate distance $d$ from the camera center to the surface hit point is computed as [S1, unofficial]:
$$
d = \sum _{k=1}^K T_k \alpha _k \delta _k
$$
where:
*   $T_k = \exp\left(-\sum _{i=1}^{k-1}\sigma _i\delta _i\right)$ is the transmittance up to sample $k$.
*   $\alpha _k = 1- \exp(-\sigma _k\delta _k)$ is the opacity at sample $k$.
*   $\sigma_i$ is the learned density.
*   $\delta_k$ is the distance between the $(k+1)$-th and $k$-th sample points.

The **surfaceness score** $s_k$ for the $k$-th sample point is [S1, unofficial]:
$$
s_k = \exp\big (-(d_k - d)^2\big )
$$
where $d_k$ is the distance from the camera to the $k$-th sample point.

The **emptiness score** $e_k$ for the $k$-th sample point is [S1, unofficial]:
$$
e_k = (1 - s_k) \cdot \mathbbm {1}(d - \Delta d - d_k > 0)
$$
where $\mathbbm {1}()$ is the indicator function (1 if the condition is true, 0 otherwise), and $\Delta d$ is a hyperparameter. This score masks out sample points behind the surface.

The **3D emptiness loss** $\ell_{3d\_empty}$ supervises the last dimension of the object code $o^{H+1}_k$ [S1, unofficial]:
$$
\ell _{3d\_empty} = -\frac {1}{K}\sum _{k=1}^K \Big ( e_k \log(o^{H+1}_k) + s_k \log(1-o^{H+1}_k) \Big )
$$

The **3D object loss** $\ell_{3d\_obj}$ pushes solid object codes to zero in empty space [S1, unofficial]:
$$
\ell _{3d\_obj} = - \frac {1}{K} \sum _{k=1}^K \Big ( e_k \sum _{h=1}^H \log(1 - o_k^h) \Big )
$$
where $o_k^h$ is the $h$-th dimension of the object code for the $k$-th point.

The total object field loss combines this with 2D supervision $\ell_{2d\_obj}$ [S1, unofficial]:
$$
\ell = \ell _{2d\_obj} + \ell _{3d\_empty} + \ell _{3d\_obj}
$$

**Parameters**
*   $\Delta d$: Hyperparameter to compensate for the inaccuracy of the estimated surface distance $d$. Set to **0.05 meters** [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Behind-Surface Points:** The indicator function in $e_k$ explicitly masks out points behind the surface ($d_k > d - \Delta d$). Without this, points behind the surface might incorrectly contribute to the emptiness loss if they are not close to the surface [S1, unofficial].
*   **Distance Inaccuracy:** The estimated surface distance $d$ is an approximation based on discrete sampling. The hyperparameter $\Delta d$ is critical to handle this inaccuracy; if set too low, valid empty space points behind the estimated surface might be penalized [S1, unofficial].
*   **Gradient Flow:** The loss is applied only to the object field MLP, not the backbone NeRF, ensuring the radiance field geometry remains stable while object codes are regularized [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds $O(K)$ operations per ray for computing scores and losses, where $K$ is the number of samples. This is negligible compared to the forward pass of the neural networks.
*   **Memory Complexity:** Requires storing the object codes for all sampled points during the backward pass if using automatic differentiation, or recomputing them. No additional large data structures.
*   **Strengths:** Provides explicit 3D geometric supervision for object segmentation without requiring 3D ground truth masks. It prevents "ghost" objects in empty space by pushing non-occupied codes to zero.
*   **Limitations:** Relies on the accuracy of the estimated surface distance $d$. If the density field $\sigma$ is noisy or the sampling is sparse, $d$ may be inaccurate, requiring careful tuning of $\Delta d$.
*   **When to use:** When training neural radiance fields for scene decomposition or manipulation where object-level segmentation is desired, but only 2D masks are available.

## Background
3D Empty Space Supervision was introduced in **DM-NeRF** (3D Scene Geometry Decomposition and Manipulation from 2D Images) by **Wang et al.** in **2022** (published at ICLR 2023) [S1, unofficial]. It addresses the issue of supervising non-occupied object codes in 3D space using only 2D instance masks. The method leverages the volume rendering equation to estimate surface distances and derives emptiness/surfaceness scores to regularize the object field, enabling geometry manipulation from 2D images alone [S1, unofficial].

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
