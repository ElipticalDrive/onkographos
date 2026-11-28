# Octree-based Implicit Surface Representation with SH and SDF Optimization

Also called: Octree-NeRF, Octree-SDF

## Overview
This method represents a 3D scene using an adaptive octree structure where leaf nodes store learnable property values, specifically Spherical Harmonic (SH) coefficients for color and Signed Distance Function (SDF) values for geometry [S1]. It solves the trade-off between the high quality of continuous neural fields and the computational cost of trilinear interpolation by using constant property values at leaf nodes for rendering, achieving over 30x speedup while maintaining visual fidelity through fine-tuning [S1]. Unlike standard grid-based methods that rely on interpolation for continuity, this approach explicitly optimizes the discrete leaf node values to approximate the continuous field, allowing for real-time rendering speeds comparable to or exceeding point-based methods like 3DGS [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Multi-view images of a scene, camera poses, and an initial octree structure (often initialized from a pre-trained neural field like VolSDF).
*   **Outputs:** An optimized octree where each leaf node stores an SDF value $S$ and a set of SH coefficients $\mathbf{c}$ (degree 2, 9 coefficients per node).

**Data Structures**
The scene is represented by an octree. Each node in the octree contains:
*   `is_leaf`: Boolean flag.
*   If `is_leaf` is true:
    *   `sdf`: Float value representing the signed distance to the surface.
    *   `sh_coeffs`: Array of 9 floats (degree 2 SH basis coefficients for RGB).
*   If `is_leaf` is false:
    *   `children`: Array of 8 child nodes.

**Subdivision Strategy**
The octree is built using a fine-to-coarse strategy guided by SDF values [S1]:
1.  Initialize a coarse grid or use a pre-trained network (e.g., VolSDF) to query SDF values at grid points.
2.  Evaluate SDF values at the center of each potential leaf node.
3.  Subdivide a node if the SDF variation within it exceeds a threshold or to reach a target maximum resolution level $l$ [S1].
4.  Continue subdivision until the target resolution is achieved [S1].

**Property Storage and Decoding**
Instead of storing values at all grid points and interpolating, values are stored only at leaf nodes [S1].
*   **SDF Query:** For a query point $\mathbf{x}$, find the containing leaf node. The SDF value $S(\mathbf{x})$ is the constant value stored in that leaf node.
*   **Color Query:** The SH coefficients $\mathbf{c}$ are retrieved from the containing leaf node. The view direction $\mathbf{v}$ is used to compute the color $\mathbf{C}$ using SH basis functions $Y(\mathbf{v})$:
    $$ \mathbf{C}(\mathbf{x}, \mathbf{v}) = \sum_{m=0}^{8} \mathbf{c}_m Y_m(\mathbf{v}) $$
    where $\mathbf{c}_m$ are the SH coefficients stored in the leaf node [S1].

**Rendering**
Volume rendering is performed by sampling points along rays. For each sample point, the SDF determines the density/opacity (via a standard SDF-to-density conversion, e.g., $\sigma = \alpha / \epsilon$ or similar differentiable mapping, though the specific mapping function is not detailed in the source, the SDF is used for geometry) and the SH coefficients determine the color [S1]. The use of constant leaf node values avoids the computational cost of trilinear interpolation [S1].

**Optimization Loop**
The octree properties (SDF and SH coefficients) are optimized using a loss function composed of three terms [S1]:
1.  **Color Loss ($\mathcal{L}_{\text{col}}$):** Mean squared error between rendered pixel colors $\hat{\mathbf{C}}_n$ and ground truth $\mathbf{C}_n$ [S1]:
    $$ \mathcal{L}_{\text{col}} = \frac{1}{N} \sum_{n=1}^N \|\mathbf{C}_n - \hat{\mathbf{C}}_n\|_2^2 $$
    where $N$ is the total number of pixels [S1].

2.  **Total Variation Loss ($\mathcal{L}_{\text{TV}}$):** Applied to SH coefficients to preserve geometric continuity and stability, especially at low resolutions [S1]:
    $$ \mathcal{L}_{\text{TV}} = \frac{1}{|\mathcal{V}|} \sum_{\mathbf{p} \in \mathcal{V}} \sqrt{(D_{\mathbf{p}}^x)^2 + (D_{\mathbf{p}}^x)^2 + (D_{\mathbf{p}}^z)^2} $$
    Note: The source text contains a typo in the formula, repeating $(D_{\mathbf{p}}^x)^2$ instead of $(D_{\mathbf{p}}^y)^2$. The intent is likely the gradient magnitude in 3D space [S1].
    *   $\mathcal{V}$ is the set of grid points.
    *   Only 1% of points from $\mathcal{V}$ are randomly selected to calculate this loss to speed up training [S1].
    *   The SH TV term is turned off when subdivision level $l > 8$ to prevent excessive smoothness and ensure detail recovery [S1].

3.  **SDF Smoothness Loss ($\mathcal{L}_{\text{smooth}}$):** Encourages a smooth surface by regularizing the gradient of the SDF [S1]:
    $$ \mathcal{L}_{\text{smooth}} = \frac{1}{|\mathcal{V}|} \sum_{\mathbf{p} \in \mathcal{V}} \|\mathcal{G}(\nabla S(\mathbf{p})) - \nabla S(\mathbf{p})\|_2^2 $$
    where $\mathcal{G}$ is a Gaussian filter applied to the gradient $\nabla S$ [S1].
    *   The set of points $\mathcal{V}$ is defined as $\{\mathbf{p} \mid -0.1 \le S(\mathbf{p}) \le 0.1\}$ when subdivision level $l \le 8$ [S1].
    *   When $l > 8$, only 10% of points from $\mathcal{V}$ are selected for fast training [S1].

**Total Loss:**
$$ \min_{\Theta} \mathcal{L} = \mathcal{L}_{\text{col}} + \lambda_1 \mathcal{L}_{\text{TV}} + \lambda_2 \mathcal{L}_{\text{smooth}} $$
where $\lambda_1$ and $\lambda_2$ are weights [S1].

**Parameters:**
*   $\lambda_1$ (TV weight): Default $10^{-5}$. Increased to $(10^{-5}, 10^{-3}]$ for bright scenes (metallic/transparent materials or intense light) [S1].
*   $\lambda_2$ (Smoothness weight): $2 \times 10^{-4}$ [S1].
*   SH Degree: 2 [S1].

**Fine-tuning:**
After achieving the target resolution, the property values of leaf nodes are fine-tuned to mitigate quality loss from using constant values instead of interpolation, making rendering quality comparable to trilinear interpolation [S1].

**Pitfalls:**
*   Small $\lambda_1$ results in coarse shapes with holes at low octree resolution, which are difficult to patch as resolution increases [S1].
*   Methods like Voxurf and BakedSDF are sensitive to lighting, exhibiting holes in areas with high specular highlights or intense lighting, whereas this method consistently produces satisfactory geometry [S1].

## Operating characteristics
*   **Speed:** Using constant property values at leaf nodes with fine-tuning achieves a speed increase of over 30 times compared to trilinear interpolation on grid points [S1].
    *   NeRF-Synthetic (Chair, $l=9$, Constant): 288.62 FPS (vs 18.51 FPS for Trilinear $l=9$) [S1].
    *   NeRF-Synthetic (Mic, $l=9$, Constant): 692.03 FPS (vs 18.35 FPS for Trilinear $l=9$) [S1].
    *   DTU (ID 24, $l=10$, Constant): 146.27 FPS (vs 5.59 FPS for Trilinear $l=10$) [S1].
*   **Training Time:**
    *   Level $l=9$: 75k iterations in 40 minutes on NVIDIA Tesla V100 [S1].
    *   Level $l=10$: 100k iterations in 60 minutes on NVIDIA Tesla V100 [S1].
*   **Quality:**
    *   PSNR on NeRF-Synthetic (Chair, $l=10$, Trilinear): 34.62 (VolSDF: 30.75, Plenoxels: 33.87) [S1].
    *   Chamfer Distance (CD) on NeRF-Synthetic (Chair, $l=10$, Trilinear): 1.39 (VolSDF: 1.57, Voxurf: 1.27) [S1].
*   **Strengths:** Real-time rendering capability, consistent geometry even in intense lighting conditions where other methods fail, efficient memory usage via sparse octree.
*   **Limitations:** May struggle to recover specular highlights in intense lighting compared to some neural methods, though geometry remains robust [S1].

## Background
This method was introduced in the paper "Real-time volume rendering with octree-based implicit surface representation" [S1]. It builds on the principles of neural implicit surfaces (like VolSDF) and adaptive octree representations, aiming to bridge the gap between high-quality neural rendering and real-time performance. The integration with VolSDF (VolSDF-SH) demonstrates its extensibility to pre-trained network-based methods [S1].

## Sources
[S1] Real-time volume rendering with octree-based implicit surface representation, https://doi.org/10.1016/j.cagd.2024.102322, fetched 2026-10-27
