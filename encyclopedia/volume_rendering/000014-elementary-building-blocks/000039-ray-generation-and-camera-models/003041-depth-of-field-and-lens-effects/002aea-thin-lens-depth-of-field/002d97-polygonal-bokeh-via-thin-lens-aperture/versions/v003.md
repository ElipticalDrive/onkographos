# Polygonal Bokeh via Thin Lens Aperture

Also called: Bokeh Rendering, Defocus Blur Rendering, N-Blade Aperture Sampling, Polygonal Bokeh

## Overview
Polygonal Bokeh via Thin Lens Aperture renders depth-of-field effects where out-of-focus highlights appear as regular polygons defined by $N$ camera blades, rather than circles [S1, S2, unofficial]. It differs from its parent [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md) by replacing concentric disk sampling with area-uniform sampling of a rotated $N$-sided polygon [S2, unofficial]. While [S3] describes a general "bokeh rendering" technique using scatter operations, it does not explicitly implement the polygonal aperture shape, focusing instead on the blur radius calculation and kernel weighting [S3].

## How it works
**Inputs:**
- Image resolution.
- Depth map (or disparity map) $D_{depth}$ [S3].
- Camera parameters: aperture radius $a$, focal length $f$, focused disparity $D_f$ (or focused depth $D_{focus}$) [S3].
- Number of blades $N$ [S1, unofficial].
- Rotation angle for the aperture polygon [S2, unofficial].
- Smoothness parameter $\beta$ for the kernel weight function [S3].
- Gamma value $\gamma$ [S3].

**Outputs:**
- Rendered image with polygonal bokeh [S1, S2, unofficial][S3].

**Data Structures:**
- **Disparity/Depth Map:** A 2D array where each element stores the depth $D_{depth}$ or disparity $D$ for the corresponding pixel [S3].
- **Aperture Polygon:** Defined by $N$ vertices on a circle of radius $r_{aperture}$. The vertices are rotated by a given angle [S2, unofficial].

**Steps:**

1.  **Calculate Blur Radius:**
    For each pixel, calculate the Circle of Confusion (CoC) radius $r_x$ using the thin lens formula [S3]:
    $$ r_x = a f \left| \frac{1}{D_{focus}} - \frac{1}{D_{depth}} \right| = a f |D_f - D| $$
    Where:
    - $a$: Camera aperture radius [S3].
    - $f$: Focal length [S3].
    - $D_{depth}$: Depth of the pixel [S3].
    - $D_{focus}$: Depth of the in-focus region [S3].
    - $D_f$: Focused disparity [S3].
    - $D$: Disparity of the pixel [S3].
    *Note: [S3] uses disparity to replace depth, assuming a fixed focal length [S3].*

2.  **Linearize Radiance:**
    Transform the input RGB image $I$ from gamma-corrected space to linear radiance $I_g$ [S3]:
    $$ I_g = I^\gamma $$
    Where $\gamma$ is the gamma intensity [S3].

3.  **Define Aperture Polygon:**
    Generate a regular $N$-sided polygon centered at the origin. The vertices are sampled uniformly on the perimeter or area to simulate the aperture shape [S2, unofficial]. The polygon is rotated by the specified angle to simulate different blade orientations [S2, unofficial].

4.  **Scatter Operation (Kernel Weighting):**
    For each pixel $x$ in the linear radiance image $I_g$, treat it as a light source emitting a weighted kernel $K_x$ centered at $x$ [S3].
    For each offset $\epsilon$ within the kernel support (determined by $r_x$), calculate the weight $K_x(\epsilon)$ [S3]:
    $$ K_x(\epsilon) = \frac{H(r_x - D_\epsilon)}{r_x^2} $$
    Where:
    - $D_\epsilon$: Distance between the center pixel $x$ and the offset pixel $x + \epsilon$ [S3].
    - $H(x)$: A smooth step function replacing the hard cutoff $H(x) = 0$ if $x < 0$, $1$ if $x \ge 0$ [S3]. The smooth version is:
      $$ H(x) = \frac{1}{2} + \frac{1}{2} \tanh(\beta \cdot x) $$
      Where $\beta$ controls the smoothness of the transition [S3].

5.  **Accumulate Bokeh:**
    Compute the final bokeh image $C_b$ by scattering the linear radiance from each pixel $x$ to its neighbors based on the kernel weights [S3]:
    $$ C_b(x) = \left( \sum_{\epsilon} I_g(x + \epsilon) K_{x+\epsilon}(-\epsilon) \right)^\gamma $$
    Where $\epsilon$ are the offsets of pixel $x$, and $x + \epsilon$ are the pixels scattering radiance to $x$ [S3].
    *Note: [S3] applies gamma correction again at the end. The term $K_{x+\epsilon}(-\epsilon)$ implies the weight calculated from the perspective of the neighbor pixel $x+\epsilon$ scattering to $x$.*

6.  **Apply Polygonal Mask (Implicit in [S2, unofficial]):**
    While [S3] describes a general scatter bokeh, [S2, unofficial] specifies that the aperture is an $N$-blade polygon. In a physically accurate implementation, the kernel $K_x$ should be masked by the polygon shape. If the scatter operation in Step 4 uses a circular support, the polygonal shape is achieved by sampling the aperture points uniformly within the polygon and using them as offsets, or by weighting the contribution based on whether the offset lies within the rotated polygon [S2, unofficial]. The provided formula in [S3] uses a circular distance $D_\epsilon$, which suggests a circular blur unless modified by the polygon sampling strategy described in [S2, unofficial].

**Parameters:**
- $N$: Number of camera blades, defining the polygon shape [S1, unofficial].
- $\beta$: Smoothness parameter for the kernel weight [S3].
- $\gamma$: Gamma value [S3].

**Edge Cases and Pitfalls:**
- **Gather vs. Scatter:** [S3] notes that gather operations are less physically accurate than scatter operations for bokeh rendering [S3].
- **Precision:** The smooth step function $H(x)$ requires careful handling of $\beta$ to avoid numerical instability or overly sharp transitions [S3].
- **Boundary Conditions:** Pixels near the image edge may have incomplete kernel support, leading to artifacts if not handled [G].

## Operating characteristics
- **Time Complexity:** $O(N_{pixels} \cdot K_{size})$, where $K_{size}$ is the number of samples in the kernel, proportional to $r_x^2$ [S3].
- **Memory Complexity:** $O(N_{pixels})$ for storing the depth/disparity map and intermediate images [S3].
- **Strengths:** Provides realistic polygonal bokeh shapes (hexagonal, octagonal) matching camera lens aesthetics [S1, S2, unofficial]. Scatter operation is physically more accurate than gather [S3].
- **Limitations:** Computationally expensive due to per-pixel kernel evaluation [S3]. Requires accurate depth/disparity map [S3].

## Background
The concept of polygonal bokeh is inherent to camera lenses with $N$ aperture blades [S1, unofficial]. The thin lens model for depth of field is a standard approximation in computer graphics [S2, unofficial][S3]. The specific scatter-based bokeh rendering technique described in [S3] is proposed for neural radiance fields and defocus map estimation [S3]. The implementation details in [S1, unofficial] and [S2, unofficial] are from unofficial sources (GitHub repositories) [S1, S2, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S3] Dual-Camera All-in-Focus Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3537178, fetched 2026-10-30
