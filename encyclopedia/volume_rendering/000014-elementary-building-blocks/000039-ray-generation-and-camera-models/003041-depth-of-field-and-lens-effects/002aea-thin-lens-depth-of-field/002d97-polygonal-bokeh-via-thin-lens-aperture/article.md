# Polygonal Bokeh via Thin Lens Aperture

Also called: Bokeh / Depth of Field, Bokeh Rendering, Camera->dof, Defocus Blur Rendering, N-Blade Aperture Sampling, Polygonal Bokeh

## Overview
Polygonal Bokeh via Thin Lens Aperture renders depth-of-field effects where out-of-focus highlights appear as regular polygons defined by $N$ camera blades, rather than circles [S1, unofficial][S2]. It differs from its parent [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md) by replacing concentric disk sampling with area-uniform sampling of a rotated $N$-sided polygon [S3, unofficial]. While [S4] describes a general scatter-based bokeh rendering technique using kernel weighting, it does not explicitly implement the polygonal aperture shape, focusing instead on the blur radius calculation and kernel weighting [S4].

## How it works
**Inputs:**
- Image resolution.
- Depth map or disparity map $D_{depth}$ [S4].
- Camera parameters: aperture radius $a$, focal length $f$, focused disparity $D_f$ (or focused depth $D_{focus}$) [S4].
- Number of blades $N$ [S1, unofficial].
- Rotation angle for the aperture polygon [S3, unofficial].
- Smoothness parameter $\beta$ for the kernel weight function [S4].
- Gamma value $\gamma$ [S4].

**Outputs:**
- Rendered image with polygonal bokeh [S1, unofficial][S2, S4].

**Data Structures:**
- **Disparity/Depth Map:** A 2D array where each element stores the depth $D_{depth}$ or disparity $D$ for the corresponding pixel [S4].
- **Aperture Polygon:** Defined by $N$ vertices on a circle, rotated by the specified angle to simulate blade orientation [S3, unofficial].

**Steps:**

1. **Calculate Blur Radius:**
   For each pixel, calculate the Circle of Confusion (CoC) radius $r_x$ using the thin lens formula [S4]:
   $$ r_x = a f \left| \frac{1}{D_{focus}} - \frac{1}{D_{depth}} \right| = a f |D_f - D| $$
   Where:
   - $a$: Camera aperture radius [S4].
   - $f$: Focal length [S4].
   - $D_{depth}$: Depth of the pixel [S4].
   - $D_{focus}$: Depth of the in-focus region [S4].
   - $D_f$: Focused disparity [S4].
   - $D$: Disparity of the pixel [S4].
   *Note: [S4] uses disparity to replace depth, assuming a fixed focal length [S4].*

2. **Linearize Radiance:**
   Transform the input RGB image $I$ from gamma-corrected space to linear radiance $I_g$ [S4]:
   $$ I_g = I^\gamma $$
   Where $\gamma$ is the gamma intensity [S4].

3. **Generate Aperture Offsets:**
   Sample offsets $\epsilon$ uniformly within a regular $N$-sided polygon centered at the origin [S3, unofficial]. The polygon is rotated by the specified angle [S3, unofficial]. The polygon radius scales with $r_x$ [G].

4. **Compute Kernel Weights:**
   For each offset $\epsilon$ within the kernel support, calculate the weight $K_x(\epsilon)$ [S4]:
   $$ K_x(\epsilon) = \frac{H(r_x - D_\epsilon)}{r_x^2} $$
   Where:
   - $D_\epsilon$: Distance between the center pixel $x$ and the offset pixel $x + \epsilon$ [S4].
   - $H(x)$: A smooth step function replacing the hard cutoff $H(x) = 0$ if $x < 0$, $1$ if $x \ge 0$ [S4]. The smooth version is:
     $$ H(x) = \frac{1}{2} + \frac{1}{2} \tanh(\beta \cdot x) $$
     Where $\beta$ controls the smoothness of the transition [S4].

5. **Accumulate Bokeh:**
   Compute the final bokeh image $C_b$ by scattering the linear radiance from each pixel $x$ to its neighbors based on the kernel weights [S4]:
   $$ C_b(x) = \left( \sum_{\epsilon} I_g(x + \epsilon) K_{x+\epsilon}(-\epsilon) \right)^\gamma $$
   Where $\epsilon$ are the offsets of pixel $x$, and $x + \epsilon$ are the pixels scattering radiance to $x$ [S4].
   *Note: [S4] applies gamma correction again at the end. The term $K_{x+\epsilon}(-\epsilon)$ represents the weight calculated from the perspective of the neighbor pixel $x+\epsilon$ scattering to $x$ [S4].*

6. **Apply Polygonal Mask:**
   The scatter offsets are constrained to the polygon shape, ensuring out-of-focus highlights take the polygonal form [S1, S3, unofficial]. If the scatter operation uses a circular support, the polygonal shape is achieved by sampling the aperture points uniformly within the polygon and using them as offsets [S3, unofficial].

**Parameters:**
- $N$: Number of camera blades, defining the polygon shape [S1, unofficial].
- $\beta$: Smoothness parameter for the kernel weight [S4].
- $\gamma$: Gamma value [S4].

**Edge Cases and Pitfalls:**
- **Gather vs. Scatter:** [S4] notes that gather operations are less physically accurate than scatter operations for bokeh rendering [S4].
- **Precision:** The smooth step function $H(x)$ requires careful handling of $\beta$ to avoid numerical instability or overly sharp transitions [S4].
- **Boundary Conditions:** Pixels near the image edge may have incomplete kernel support, leading to artifacts if not handled [G].
- **Focus Setting:** Focus distance can be fixed, set numerically, or set interactively by clicking a voxel [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides realistic polygonal bokeh shapes (hexagonal, octagonal) matching camera lens aesthetics [S1, unofficial][S2]. Scatter operation is physically more accurate than gather [S4]. Supports all camera projection modes [S2].
- **Limitations:** Computationally expensive due to per-pixel kernel evaluation [S4]. Requires accurate depth/disparity map [S4]. Larger aperture size is required for effective depth of field rendering [S2].

## Background
The concept of polygonal bokeh is inherent to camera lenses with $N$ aperture blades [S1, unofficial]. The thin lens model for depth of field is a standard approximation in computer graphics [S2, S4]. The scatter-based bokeh rendering technique described in [S4] is proposed for defocus map estimation [S4]. Implementation details in [S1, unofficial] and [S2] are from voxel rendering software [S1, unofficial][S2]. [S3, unofficial] implements N-blade polygonal aperture with rotation in a photon engine [S3, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
[S3] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S4] Dual-Camera All-in-Focus Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3537178, fetched 2026-10-30
