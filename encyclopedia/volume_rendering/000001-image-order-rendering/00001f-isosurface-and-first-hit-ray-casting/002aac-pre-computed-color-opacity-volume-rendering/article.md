# Pre-computed Color/Opacity Volume Rendering

Also called: Levoy's method, Pre-computed Classification and Shading, Pre-computed Volume Rendering, Voxel space volume rendering

## Overview
Pre-computed Color/Opacity Volume Rendering is a voxel-space technique that precomputes discrete opacity and color fields from the scalar data field, then uses standard front-to-back ray casting to render pixels [S1, unofficial][S2]. It sets itself apart from its siblings by moving the expensive classification and shading calculations from the ray traversal loop into a one-time (or parameter-change-time) preprocessing step, allowing for fast, view-independent rendering [S1, unofficial][S2]. Unlike [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md) which computes shading per-sample, this method fixes the lighting model during precomputation [S1, unofficial].

## How it works
**Inputs:**
*   Scalar data field $D$ of dimensions $N_x \times N_y \times N_z$.
*   Opacity transfer function $T_o(v)$, mapping data value $v$ to opacity $\alpha \in [0, 1]$.
*   Color transfer function $T_c(v)$, mapping data value $v$ to base object color $\mathbf{C}_{obj}$.
*   Light direction vector $\mathbf{l}$ (fixed for the precomputation).
*   Shading model parameters (e.g., ambient, diffuse coefficients).

**Outputs:**
*   Precomputed Opacity Volume $O$ of dimensions $N_x \times N_y \times N_z$.
*   Precomputed Color Volume $C$ of dimensions $N_x \times N_y \times N_z$.
*   Rendered image pixels.

**Data Structures:**
*   $D[i,j,k]$: Input scalar value at voxel $(i,j,k)$.
*   $O[i,j,k]$: Precomputed opacity at voxel $(i,j,k)$.
*   $C[i,j,k]$: Precomputed RGB color at voxel $(i,j,k)$.
*   $\nabla D[i,j,k]$: Gradient vector at voxel $(i,j,k)$, computed via central differences.

**Steps:**

1.  **Compute Gradients:** Calculate the gradient $\nabla D[i,j,k]$ for every voxel in the dataset using central differences [S1, unofficial]. For interior voxels:
    $$ \nabla D[i,j,k] = \left( \frac{D[i+1,j,k] - D[i-1,j,k]}{2}, \frac{D[i,j+1,k] - D[i,j-1,k]}{2}, \frac{D[i,j,k+1] - D[i,j,k-1]}{2} \right) $$
    Boundary conditions must be handled appropriately (e.g., one-sided differences) [G].

2.  **Compute Opacity Field:** For each voxel $(i,j,k)$, apply the opacity transfer function to the data value:
    $$ O[i,j,k] = T_o(D[i,j,k]) $$
    This step is performed once, or whenever $T_o$ or the sampling distance changes [S1, unofficial].

3.  **Compute Object Color:** For each voxel $(i,j,k)$, apply the color transfer function:
    $$ \mathbf{C}_{obj}[i,j,k] = T_c(D[i,j,k]) $$

4.  **Compute Shading and Final Color Field:** For each voxel $(i,j,k)$, approximate the surface normal $\mathbf{n}$ using the normalized gradient:
    $$ \mathbf{n}[i,j,k] = \frac{\nabla D[i,j,k]}{\|\nabla D[i,j,k]\|} $$
    Calculate the shaded color $\mathbf{C}[i,j,k]$ using a local shading model (typically diffuse-only, as specular requires view-dependent vectors) [S1, unofficial]. For a fixed light direction $\mathbf{l}$ and ambient color $\mathbf{C}_{amb}$:
    $$ \mathbf{C}[i,j,k] = \mathbf{C}_{amb} + k_d (\mathbf{n}[i,j,k] \cdot \mathbf{l}) \mathbf{C}_{obj}[i,j,k] $$
    where $k_d$ is the diffuse coefficient. This results in the precomputed color volume $C$ [S1, unofficial][S2]. Note: Full Phong specular shading is impossible without recomputing for every view because it requires the view vector $\mathbf{v}$ [S1, unofficial].

5.  **Ray Casting:** For each pixel, cast a ray through the volume. Sample the precomputed volumes $O$ and $C$ at the ray's intersection points with the voxel grid using tri-linear interpolation [S2].

6.  **Front-to-Back Composition:** Accumulate the color and opacity along the ray using front-to-back compositing [S1, unofficial]:
    $$ C_{pixel} = \sum_{k=1}^{N} c_k \alpha_k \prod_{j=1}^{k-1} (1 - \alpha_j) $$
    where $c_k$ and $\alpha_k$ are the interpolated color and opacity from volumes $C$ and $O$ at the $k$-th sample along the ray.

**Parameters:**
*   Transfer functions $T_o, T_c$: User-defined.
*   Light direction $\mathbf{l}$: Fixed during precomputation.
*   Shading coefficients ($k_d, \mathbf{C}_{amb}$): Fixed during precomputation.

**Edge Cases and Pitfalls:**
*   **Light/View Changes:** Changing the light source position requires recomputing the entire color field $C$ for the whole dataset [S1, unofficial]. Changing the viewing angle requires recomputing $C$ if specular shading is desired, which is prohibitively expensive [S1, unofficial].
*   **Memory:** Storing gradients and gradient magnitudes for all voxels, plus the two output volumes ($O$ and $C$), increases memory requirements significantly [S1, unofficial][S2].
*   **Interactivity:** Any change in classification or shading parameters requires recomputing $O$ and $C$, making interactive exploration of dynamic datasets difficult for high-resolution data [S2].

## Subtopics

### Voxel Space vs. Image Space
This method operates in voxel space by precomputing attributes, whereas image-space methods (like [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md)) compute attributes on-the-fly during ray traversal. Choose voxel space for static lighting and classification; choose image space for interactive parameter tuning.

## Operating characteristics
*   **Time Complexity:** Precomputation is $O(N_x N_y N_z)$ for gradient, opacity, and color field generation. Rendering is $O(P \cdot S)$, where $P$ is pixels and $S$ is samples per ray, similar to standard ray casting but with faster per-sample processing (no shading/gradient calculation).
*   **Memory Complexity:** Requires storage for the original data, gradients (3 components), opacity volume, and color volume. This is significantly higher than image-space methods which only need the original data [S1, unofficial][S2].
*   **Strengths:** Fast rendering once precomputed; view-independent lighting; simple compositing.
*   **Limitations:** High memory usage; non-interactive for lighting/view changes; diffuse shading only (without expensive recomputation) [S1, unofficial][S2].
*   **When to use:** Static scenes with fixed lighting and classification; offline rendering; low-to-medium resolution data where memory is not a bottleneck.
*   **When not to use:** Interactive visualization requiring dynamic light/view changes; high-resolution datasets with limited memory; scenes requiring view-dependent specular highlights.

## Background
This method is based on the work introduced by Marc Levoy [LEV88] and is widely referred to as Levoy's method [S1, unofficial]. It was one of the early approaches to volume rendering, precomputing opacity and color fields to accelerate the rendering process [S1, unofficial]. It is a foundational technique in voxel-space volume rendering [S1, unofficial].

## Sources
[S1] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
[S2] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
