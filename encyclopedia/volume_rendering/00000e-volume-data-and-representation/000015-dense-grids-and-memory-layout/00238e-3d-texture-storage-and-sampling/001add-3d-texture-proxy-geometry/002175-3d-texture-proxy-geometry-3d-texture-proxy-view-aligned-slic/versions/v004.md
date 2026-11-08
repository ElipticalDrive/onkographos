# 3D Texture Proxy Geometry — 3D Texture Proxy, View-Aligned Slices, 3D-Texture Volume Rendering, 3D Texture-Based Volume Rendering

Also called: 3D Texture Proxy, 3D-Texture Volume Rendering, View-Aligned Slices, Voltx Method, Volume Rendering with Shading via Three-Dimensional Texture Maps, Volume Texturing

## Overview
3D Texture Proxy Geometry is a direct volume rendering technique that represents a volumetric dataset as a stack of parallel planar polygons (slabs) textured with a 3D texture map, leveraging hardware-accelerated tri-linear interpolation to resample the volume at arbitrary locations [S3, S5, S6, unofficial]. It solves the performance bottleneck of ray-casting by offloading the integration and interpolation steps to the graphics pipeline's texture mapping hardware, enabling interactive rates for moderate-sized volumes [S1][S3, unofficial]. Unlike shear-warp methods which fix the number of planes to the volume resolution, this method allows the number of proxy planes to be adjusted at runtime independent of the data size, offering a trade-off between image quality and frame rate [S3, S4, unofficial].

## How it works
The method converts a rectilinear volume dataset into a 3D texture map containing pre-computed color and opacity values, then renders the volume by applying this texture to a series of view-aligned proxy planes [S3, S4, unofficial].

### Inputs and Outputs
*   **Input:** A rectilinear volume dataset with dimensions $(n_x, n_y, n_z)$ and voxel spacings $(A_x, A_y, A_z)$.
*   **Output:** A 2D image in the frame buffer.

### Data Structures
1.  **3D Texture Map:** Stores pre-computed color (RGBA) values. One texel corresponds to one voxel. The resolution $(N_x, N_y, N_z)$ is the least powers of two greater than or equal to $(n_x, n_y, n_z)$ [S3, unofficial].
2.  **Lookup Table (LUT):** A software table mapping quantized gradient indices and material classifications to RGBA values. Size is $(m+1) \times g$, where $m$ is the number of materials and $g$ is the number of quantized gradients [S3, unofficial].
3.  **Proxy Geometry:** A stack of $K$ parallel planes (squares) orthogonal to the screen-z axis, spanning a bounding cube of side length $d$ [S3, unofficial].

### Preprocessing
1.  **Classification and Gradient Quantization:** For each voxel, compute the gradient magnitude. Classify the voxel as 'reflecting' or 'ambient' based on a client-supplied gradient-magnitude threshold [S3, S4, unofficial]. Quantize the gradient direction into an index [S3, unofficial].
2.  **Lookup Table Construction:** For each combination of material type and quantized gradient, compute the integrated color and opacity for a slab of thickness $\Delta$ (see Formulas). Fill the LUT [S3, S4, unofficial].
3.  **Texture Map Filling:** For each voxel, use its classification and gradient index to look up the RGBA value in the LUT. Fill the 3D texture map with these values [S3, S4, unofficial]. Large volumes are partitioned into multiple texture maps if texture memory is insufficient [S3, S4, unofficial].

### Rendering Steps
1.  **Bounding Cube Construction:** Construct a bounding cube centered on the volume origin with side length $d = \sqrt{L_x^2 + L_y^2 + L_z^2}$, where $L_x = N_x A_x$, etc. This cube must contain any rotation of the volume [S3, unofficial].
2.  **Plane Generation:** Generate $K$ parallel planes (squares) orthogonal to the view direction. The default number of planes is $d/A_z$ (one sample per voxel along the view axis), typically multiplied by 2–4 for accuracy [S3, S4, unofficial].
3.  **Texture Coordinate Transformation:** Compute texture coordinates $(s, t, r)$ for the corners of the squares.
    *   Initial mapping for original orientation:
        $$ s(x) = \frac{x + \frac{1}{2}n_x A_x}{N_x A_x}, \quad t(y) = \frac{y + \frac{1}{2}n_y A_y}{N_y A_y}, \quad r(z) = \frac{z + \frac{1}{2}n_z A_z}{N_z A_z} $$
    *   For arbitrary viewing angles, apply the inverse rotation matrix $R^{-1}$ to the texture coordinates. The transformation order must be rotation followed by scaling to avoid shearing distortion [S3, S7, unofficial]:
        $$ (s,t,r) = (x,y,z) R^{-1} D^{-1} S T $$
        where $D$ is uniform scale by $d$, $S$ is non-uniform scale by $(d/L_x, d/L_y, d/L_z)$, and $T$ is translation [S3, unofficial].
4.  **Clipping:** Position six programmable world-space clipping planes at the faces of the volume bounding box to clip out regions of the proxy planes that lie outside the volume [S3, S7, unofficial].
5.  **Rasterization and Blending:** Render the squares back-to-front. The hardware interpolates the texture coordinates across the square. Because texture coordinates outside $[0, 1]$ are interpolated (unlike colors which are clamped), in-range values are generated precisely when the pixel is within the volume [S3, unofficial]. Sample the 3D texture using tri-linear interpolation. Blend the resulting color with the frame buffer using back-to-front compositing [S3, S5, unofficial].

### Formulas
*   **Slab Thickness:** $\Delta = \frac{\text{Total distance covered by stack}}{K}$ [S3, S4, unofficial].
*   **Color and Opacity Integration (per slab):**
    $$ C_p = E(p) \frac{1 - e^{-\alpha \Delta}}{\alpha}, \quad A = 1 - e^{-\alpha \Delta} $$
    where $E(p)$ is color emission per unit distance for primary $p$, $\alpha$ is opacity per unit distance, and $\alpha_i$ is opacity per unit distance (Note: Sources [S3, unofficial] and [S4, unofficial] use slightly different notation; [S3, unofficial] uses $\alpha_i$ for opacity and $\alpha$ for emission in the numerator fraction, while [S4, unofficial] uses $\bar{\rho}$ for opacity. The standard volume rendering equation form is used here) [S3, S4, unofficial].
*   **Back-to-Front Blending:**
    $$ C_{dst} = (1 - \alpha_{src}) C_{dst} + \alpha_{src} C_{src} $$
    where $C_{dst}, \alpha_{dst}$ are the color and opacity in the frame buffer, and $C_{src}, \alpha_{src}$ are the incoming fragment's color and opacity [S5, unofficial].
*   **Bounding Cube Diagonal:**
    $$ d = \sqrt{L_x^2 + L_y^2 + L_z^2} $$
    [S3, unofficial]

### Parameters
*   **Number of Planes ($K$):** Default $d/A_z$. Typical value: $2 \times$ to $4 \times$ default [S3, S4, unofficial].
*   **Gradient Threshold:** Client-supplied value to distinguish reflecting from ambient materials [S3, unofficial].
*   **Texture Resolution:** Powers of two $(N_x, N_y, N_z) \ge (n_x, n_y, n_z)$ [S3, unofficial].

### Edge Cases and Pitfalls
*   **Perspective Artifacts:** In perspective projection, the effective slab thickness increases with off-axis angle, causing up to 14% error in large windows. Compensation requires spherical shells, which are expensive [S3, S4, S6, unofficial].
*   **Nonlinear Mapping Inaccuracy:** Hardware interpolates pre-computed color, not raw data. Nonlinear transfer functions cause inaccuracies, especially in close-up views [S3, S7, unofficial].
*   **Aliasing:** Discontinuous transfer functions (single material per voxel) can cause aliasing artifacts [S3, S4, unofficial].
*   **Geometry Cost:** High geometric overhead (generating, transforming, rasterizing many primitives) becomes a bottleneck on hardware optimized for fragment processing [S2, S6, unofficial].
*   **Memory Limits:** Large volumes require partitioning into multiple texture maps, which cannot be transferred to GPU memory in parallel [S3, S4, unofficial].

## Subtopics

### [Discontinuous Displacement Mapping for Volume Graphics](10637-discontinuous-displacement-mapping-for-volume-graphics.md)
Discontinuous Displacement Mapping is a variant that deforms a proxy scene geometry (typically a bounding box) using a 3D displacement texture to sample an original object texture, handling discontinuities such as cuts and fractures via an alpha mask to prevent interpolation artifacts. It differs from standard displacement mapping by explicitly modeling discontinuities through a separate alpha channel and transforming normals at cut boundaries to maintain correct lighting, allowing for effects like fracturing, slicing, and deforming in real-time.

## Operating characteristics
*   **Time Complexity:** Preprocessing is proportional to volume size $V$ (texture filling) and independent of volume size for the LUT (size depends on gradient quantization levels and material count) [S3, unofficial]. Rendering time is proportional to the number of proxy planes $K$ and image resolution [S3, unofficial].
*   **Memory Complexity:** Requires $O(V)$ for the 3D texture map and $O(g \cdot m)$ for the LUT, where $g$ is the number of quantized gradients and $m$ is the number of materials [S3, unofficial]. Gradient storage increases memory by a factor of 2–3 if gradients are stored separately [S6, unofficial].
*   **Performance:** On SGI Reality Engine II, rendering 600x600 images with 1000 planes took several seconds with directional lighting, and ~10 times faster without lighting [S3, S4, S7, unofficial]. Performance scales poorly with multi-processor setups due to serial graphics pipeline bottlenecks [S3, S7, unofficial].
*   **Strengths:** Interactive rates for moderate volumes; supports arbitrary viewing angles and perspective projection without performance penalty (unlike shear-warp); simple implementation leveraging standard 3D texture hardware [S3, S4, unofficial].
*   **Limitations:** High geometric overhead; inaccurate with nonlinear transfer functions in close-ups; perspective artifacts; limited by texture memory size [S2, S3, S4, S6, unofficial].
*   **When to Use:** Orthographic or perspective viewing of moderate-sized volumes ($256^3$ or less) where interactive frame rates are needed and hardware supports 3D textures [S2, S3, unofficial].
*   **When Not to Use:** Large volumes exceeding texture memory; scenes requiring high-fidelity nonlinear shading; navigation inside the volume where perspective sampling artifacts are unacceptable [S2, S3, S6, unofficial].

## Background
The method was introduced by Cullip and Neumann in 1993 [CN94] and improved by Cabral et al. in 1994 to exploit hardware 3D texture interpolation [S2, unofficial]. The specific implementation "Voltx" was described by Van Gelder and Kim in 1997, demonstrating direct volume rendering with shading via 3D textures on Silicon Graphics hardware [S3, S4, unofficial]. It is used in early hardware-accelerated volume rendering systems and has been revisited for modern GPUs, including applications in CT Angiography [S6, unofficial].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S4] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S5] [PDF] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
[S6] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S7] o,,ect Volume Rendering with Shading via Three- ..., https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
[S8] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
