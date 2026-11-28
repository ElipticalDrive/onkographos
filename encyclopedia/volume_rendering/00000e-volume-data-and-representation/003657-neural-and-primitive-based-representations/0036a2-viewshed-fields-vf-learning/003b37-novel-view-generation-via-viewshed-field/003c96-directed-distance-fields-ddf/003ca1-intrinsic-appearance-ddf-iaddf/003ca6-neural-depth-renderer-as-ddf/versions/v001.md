# Neural Depth Renderer as DDF

Also called: Neural Depth Rendering as Directed Distance Field

## Overview
This method treats a neural depth renderer as a conditional Directed Distance Field (DDF), mapping camera parameters and scene shapes to depth and visibility images [S1]. It differs from standard neural radiance fields by explicitly structuring the output as a DDF, thereby inheriting geometric properties such as the relationship between camera position derivatives and surface normals [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Continuous camera parameters $\Pi \in \Lambda_\Pi$ (intrinsics and extrinsics) and a shape representation $\Lambda_S$.
*   **Outputs:** An $m$-pixel image $I_{d,\xi}$ where each pixel value is a tuple $(d(p, v), \xi(p, v))$. Here, $d$ is the depth and $\xi$ is the visibility (typically in $[0, 1]$) [S1].

**Data Structures:**
The renderer $f$ is defined as a function $f : \Lambda_\Pi \times \Lambda_S \rightarrow I_{d,\xi}$ [S1]. The image domain is $R_+ \times [0, 1]$ [S1].

**Steps:**
1.  For each pixel in the output image, determine the camera center of projection $p$ and the ray direction $v$ based on the camera parameters $\Pi$ [S1].
    *   $p$ is determined by the camera position (extrinsics) [S1].
    *   $v$ is determined by the ray direction corresponding to the pixel coordinates, originating from $p$ [S1].
2.  Evaluate the neural renderer $f$ using $p$, $v$, and the shape $\Lambda_S$ to obtain the depth $d(p, v)$ and visibility $\xi(p, v)$ [S1].
3.  Construct the image $I_{d,\xi}$ from these pixel values [S1].

**Formulas:**
The core mapping is:
$$ f(\Pi, \Lambda_S) \rightarrow I_{d,\xi} $$
where each pixel $i$ contains:
$$ I_{d,\xi}[i] = (d(p_i, v_i), \xi(p_i, v_i)) $$
with $p_i$ and $v_i$ being functions of $\Pi$ [S1].

**Properties:**
Because the depth is treated as a function of $p$ and $v$, DDF properties apply:
*   **Property II:** The derivative of the depth image with respect to camera position, $\partial_{C_p} I_d$, relates to the surface normals image [S1].
*   **Property III:** The derivative with respect to viewpoint, $\partial_{C_v} I_d$, is constrained [S1].

**Edge Cases and Pitfalls:**
*   The method assumes the neural renderer can be evaluated as a function of $(p, v)$ regardless of its internal architecture [S1].
*   Precision in computing $p$ and $v$ from $\Pi$ is critical for the validity of the derivative properties [S1].

## Subtopics

### [CPDDF-based SI3DR](9291-cpddf-based-si3dr.md)
Predicts camera and latent shape from an image, using a Conditional PDDF for rendering. Chosen when explicit camera prediction from single images is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables differentiable rendering of geometric quantities by leveraging DDF properties [S1].
*   **Limitations:** Relies on the neural renderer's ability to output valid depth/visibility tuples [S1].

## Background
This perspective was proposed to improve the framework for differentiable neural rendering of geometric quantities [S1]. It builds on the theoretical foundation of Probabilistic Directed Distance Fields [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
