# Occlusion Caching

Also called: Baking Occlusion

## Overview
Occlusion Caching is a technique that pre-calculates ambient occlusion (AO) data into a set of spatial volumes to accelerate rendering and improve visual fidelity in inverse rendering pipelines [S1]. It differs from runtime ray-marched AO by baking the occlusion integral into spherical harmonic (SH) coefficients stored in discrete volumes, allowing for fast interpolation during the decomposition stage [S1].

## How it works
**Inputs:** Static scene geometry or volume; **Outputs:** A set of occlusion volumes $V^{occl}$ containing SH coefficients representing ambient occlusion.

**Data Structures:**
*   **Occlusion Volumes ($V^{occl}$):** A grid of discrete volumes $v_i^{occl}$.
*   **SH Coefficients ($f_o$):** Spherical Harmonic coefficients representing the principal components of the occlusion field.
*   **Depth Cubemaps ($\hat{D}_p^i$):** 6 depth maps per volume $v_i^{occl}$, corresponding to 6 camera views.
*   **Occlusion Cubemaps ($\hat{O}_p^i$):** 6 cubemaps per volume, derived from depth maps.

**Steps:**

1.  **Baking Stage:**
    1.  For each occlusion volume $v_i^{occl}$, define six cameras with a 90-degree Field of View (FoV). These cameras must be non-overlapping and oriented to cover the hemisphere or full sphere relevant to the volume's local context [S1].
    2.  Perform six render passes to obtain depth cubemaps $\{\hat{D}_p^i\}_{p=1}^6$ [S1].
    3.  Convert the depth cubemaps $\{\hat{D}_p^i\}_{p=1}^6$ into occlusion cubemaps $\{\hat{O}_p^i\}_{p=1}^6$. The conversion typically involves determining if a fragment is occluded based on depth thresholds or visibility [S1].
    4.  Compute the principal components of the occlusion data from the cubemaps and store them as SH coefficients $f_o^i$ for volume $v_i^{occl}$ [S1].

2.  **Decomposition (Recovery) Stage:**
    1.  For a surface point $x$ with normal $n$, identify the eight nearest occlusion volumes $\{v_k\}_{k=1}^8$ from $V^{occl}$ [S1].
    2.  Let $p_k$ be the position vector of volume $v_k$ and $f_k^o$ be its SH coefficients [S1].
    3.  Calculate standard trilinear interpolation weights $\{w_k\}_{k=1}^8$ based on the distance of $x$ to the corners of the voxel grid formed by the eight volumes [S1].
    4.  Apply **Masked-Trilinear Interpolation** to compute modified weights $\tilde{w}_k$:
        $$
        \tilde{w}_k = \begin{cases} 
        0 & \text{if } (p_k - x) \cdot n \le 0 \\
        w_k & \text{if } (p_k - x) \cdot n > 0 
        \end{cases}
        $$
        This mask ensures that only volumes "above" the surface (in the direction of the normal) contribute to the ambient occlusion integral, respecting the upper hemisphere $\Omega$ constraint [S1].
    5.  Normalize the masked weights:
        $$
        \hat{w}_k = \frac{\tilde{w}_k}{\sum_{j=1}^8 \tilde{w}_j}
        $$
        [S1]
    6.  Interpolate the SH coefficients to obtain the final coefficients $f_x^o$ for point $x$:
        $$
        f_x^o = \sum_{k=1}^8 \hat{w}_k f_k^o
        $$
        [S1]
    7.  Evaluate the SH basis functions at the normal direction $n$ using $f_x^o$ to reconstruct the ambient occlusion value [S1].

**Parameters:**
*   Camera FoV: 90 degrees [S1].
*   Number of cameras per volume: 6 [S1].

**Edge Cases:**
*   If all masked weights $\tilde{w}_k$ are zero (e.g., point is deep inside geometry or normal points away from all surrounding volumes), the normalization step fails. [G] Handle by defaulting to a fully occluded value (0) or unoccluded value (1) depending on context.
*   Precision issues in dot product $(p_k - x) \cdot n$ near zero may cause instability. [G] Use a small epsilon threshold for the comparison.

## Subtopics

### Masked-Trilinear Interpolation for AO
A variant of trilinear interpolation used during the recovery stage to enforce hemispherical constraints. Standard trilinear interpolation weights $\{w_k\}$ are modified to $\{\tilde{w}_k\}$ by zeroing out contributions from volumes where the vector from the point to the volume center $(p_k - x)$ has a non-positive dot product with the surface normal $n$. This ensures that ambient occlusion is only calculated from the upper hemisphere relative to the surface normal, preventing artifacts from occlusion data "below" the surface [S1].

## Operating characteristics
*   **Time Complexity:** Baking is $O(N_{vol} \times 6)$ render passes, where $N_{vol}$ is the number of occlusion volumes. Decomposition is $O(1)$ per point (constant 8 neighbors).
*   **Memory Complexity:** $O(N_{vol} \times C)$, where $C$ is the number of SH coefficients stored per volume.
*   **Strengths:** Fast runtime evaluation; decouples occlusion calculation from ray marching; improves visual fidelity by pre-calculating complex occlusion integrals [S1].
*   **Limitations:** Requires static scenes or volumes; baking time can be significant; interpolation artifacts may occur if volume resolution is too low [S1].
*   **When to use:** Inverse rendering of static scenes where AO is a significant component of the appearance and runtime performance is critical [S1].

## Background
Occlusion Caching is introduced in GS-IR: 3D Gaussian Splatting for Inverse Rendering [S1]. It builds on the concept of baking lighting information into spatial data structures to accelerate rendering in inverse rendering pipelines. The method leverages spherical harmonics to compactly represent the angular variation of ambient occlusion [S1].

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
