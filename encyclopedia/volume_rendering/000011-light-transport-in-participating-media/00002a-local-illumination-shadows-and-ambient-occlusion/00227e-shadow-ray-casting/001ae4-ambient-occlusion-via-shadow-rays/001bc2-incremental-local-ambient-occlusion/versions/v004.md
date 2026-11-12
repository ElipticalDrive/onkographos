# Incremental Local Ambient Occlusion

Also called: LAO, LAO-I

## Overview
Incremental Local Ambient Occlusion (LAO) is a shading model for direct volume rendering that computes ambient lighting effects by integrating attenuated transfer function density over a local spherical neighborhood around each voxel, rather than using a constant ambient factor or global shadowing [S2, unofficial]. It solves the problem of missing local shadowing and context in direct volume rendering while avoiding global shadows that might hide relevant information [S2, unofficial]. Unlike its siblings, which may compute occlusion in a single pass or use different sampling strategies, LAO-I updates the occlusion result incrementally, processing one ray direction per frame and storing the accumulated result in an ambient light texture using OpenGL blend operations, allowing for user interaction during computation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A regular linear volume data set, a transfer function (TF) mapping density to opacity $\alpha$ and color, a set of ray directions distributed over a sphere, and a neighborhood radius $R_b$.
*   **Outputs:** An ambient light intensity texture (3D texture) storing the accumulated LAO value $A(x)$ for each voxel, and the final rendered image combining emissive color and ambient light.

**Data Structures:**
1.  **Packed Volume Texture:** A 3D texture containing the volume data organized into a multiresolution representation.
    *   **Layout:** The volume is divided into blocks of typically $16^3$ voxels [S2, unofficial].
    *   **LOD Levels:** Each block is encoded with levels of detail: $8^3, 4^3, 2^3, 1^3$ voxels [S2, unofficial].
    *   **Constraints:** The minimum LOD for nonempty blocks is often restricted to $4^3$ voxels to allow the reverse mapping index texture to be 64 times smaller than the packed volume [S2, unofficial].
2.  **Index Textures:**
    *   **Forward Mapping:** Maps packed coordinates to volume coordinates.
    *   **Reverse Mapping:** Maps volume coordinates back to packed coordinates (size reduced by restricting min LOD) [S2, unofficial].
3.  **Ambient Light Texture:** A 3D texture with the same dimensions and multiresolution layout as the packed volume texture, storing the accumulated intensity $A(x)$ [S2, unofficial].
4.  **Direction Texture:** Stores ray directions and their associated weights $w_k$ [S2, unofficial].

**Parameters:**
*   $R_b$: Radius of the surrounding LAO sphere. Typical values: 16 or 32 voxels [S2, unofficial].
*   $a$: Small offset to avoid self-occlusion of the voxel at location $x$ [S2, unofficial].
*   $M$: Number of samples for the highest resolution fixed sampling rate. Value: 15 [S2, unofficial].
*   $K$: Number of ray directions to sample. Dynamically configurable. Baseline for quality: 128 directions [S2, unofficial].
*   $w_k$: Weight for direction $k$.

**Formulas:**
The light contribution for direction $k$ at voxel $x$ is given by [S2, unofficial]:
$$ A_k(x) = \int_{a}^{R_b} g_A(s) \cdot e^{-\int_{a}^{s} \alpha(t) dt} ds $$
where $\alpha(t)$ is the extinction coefficient (opacity from TF) and $g_A(s)$ is the light contribution function.

To avoid sharp shadow boundaries, $g_A(s)$ is defined as a volumetric distribution [S2, unofficial]:
$$ g_A(s) = \frac{1}{R_b} \cdot \alpha(s) $$
(If emissive materials are included, $g_A(s) = \frac{1}{R_b} \cdot \alpha(s) + c_E(s)$, where $c_E(s)$ is the emissive color component [S2, unofficial].)

The numerical evaluation using front-to-back compositing is [S2, unofficial]:
$$ A_k(x) = \frac{1}{M} \sum_{m=1}^{M} g_m \prod_{i=1}^{m-1} (1 - d_i) $$
where $g_m = 1$ (due to scaling) and $d_i$ is the opacity at sample position $i$ [S2, unofficial].

The combined local ambient occlusion is [S2, unofficial]:
$$ A(x) = \frac{1}{K} \sum_{k=1}^{K} w_k A_k(x) $$

**Incremental Update:**
The accumulation is updated incrementally using OpenGL blend operations [S2, unofficial]:
$$ A_k = \frac{1}{k} A_k + \left(1 - \frac{1}{k}\right) A_{k-1} $$
where $A_k$ is the accumulated intensity after $k$ ray directions, $A_0 = 0$, and $A_{k-1}$ is the previous accumulation [S2, unofficial].

**Steps:**
1.  **Preprocessing:**
    1.  Divide the regular linear volume into blocks (typically $16^3$ voxels) [S2, unofficial].
    2.  Encode each block with LODs ($8^3, 4^3, 2^3, 1^3$) and organize into a multiresolution representation [S2, unofficial].
    3.  Pack volume data into a single 3D texture [S2, unofficial].
    4.  Create forward and reverse mapping index textures [S2, unofficial].
    5.  Create a 3D texture for emissive color and ambient light [S2, unofficial].
    6.  Create ray directions by subdividing a tetrahedron, icosahedron, or octahedron to achieve the desired number of rays $K$. Store directions and weights $w_k$ in a texture [S2, unofficial].
2.  **Incremental LAO Pipeline (Per Frame):**
    1.  Map slices of the volume one by one as the rendering target of a framebuffer object [S2, unofficial].
    2.  Trigger processing by rendering a quad over the entire framebuffer, where each pixel maps to one voxel in the 3D texture [S2, unofficial].
    3.  For the current frame, process one (or more) ray direction(s) from the direction texture [S2, unofficial].
    4.  Compute $A_k(x)$ for the current direction using the multiresolution layout and adaptive or fixed sampling [S2, unofficial].
        *   *Adaptive Sampling:* Adjust step length relative to the block's resolution to avoid missing occluding features in lower-resolution blocks [S2, unofficial].
    5.  Update the ambient light intensity texture using the OpenGL blend operation corresponding to the incremental average formula [S2, unofficial].
    6.  Interleave rendering with intensity texture updates to allow user interaction [S2, unofficial].
3.  **Final Rendering:**
    1.  Once all $K$ directions are processed, use the final ambient light intensity volume in the standard rendering pass [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sharp Shadows:** Using boundary light contribution ($g_A(s) = \delta(s-R_b)$) causes hard shadows if any occluder intersects the sphere. Use volumetric light contribution to smooth shadows [S2, unofficial].
*   **Too Few Directions:** Using too few directions decreases the ambient effect. 128 directions are recommended as a baseline [S2, unofficial].
*   **High-Frequency Details:** LAO alone loses high-frequency surface details. Combine with diffuse shading to explore fine structures [S2, unofficial].
*   **Clip Planes:** Boundaries of active volumes have poorly defined gradients, making gradient-based illumination inappropriate. LAO improves cross-section visibility [S2, unofficial].
*   **Emissive Context:** Improved context from separate absorption TF in LAO is only noticeable when the absorption of the tissue used in the rendering pass is lower than the LAO absorption [S2, unofficial].

## Subtopics

### [Vicinity Shading](1210-vicinity-shading.md)
Vicinity shading is a pre-computed volumetric ambient occlusion technique that enhances volume visualization by calculating an "obscurance" value for each voxel, representing the occlusion information from its immediate neighborhood. It differs from local volumetric shadowing methods (e.g., Hernell et al.) by explicitly testing visibility against the first occluder and ignoring the environment beyond it, rather than integrating attenuation over the entire ray distance. This approach allows for interactive exploration and saliency map generation at a computational cost proportional to the number of voxels times the number of directions, significantly lower than the quadratic complexity of full volumetric shadowing. [Vicinity Shading](1210-vicinity-shading.md)

### [Multiresolution LAO Pipeline](12400-multiresolution-lao-pipeline.md)
The Multiresolution LAO Pipeline is a per-fragment GPU-based method for computing Local Ambient Occlusion (LAO) illumination in multiresolution volume rendering. It accelerates LAO computation by leveraging the same data reduction achieved by the volume's Level-of-Detail (LOD) selection, adapting sampling rates to voxel block resolution. Unlike the parent Incremental LAO method which updates one ray direction per frame, this pipeline processes all incident light directions incrementally over multiple frames using a fragment shader pipeline, allowing for dynamic configuration of ray counts and efficient hardware-accelerated blending. [Multiresolution LAO Pipeline](12400-multiresolution-lao-pipeline.md)

### Adaptive Sampling Density for LAO
A method to improve LAO performance by adjusting sampling density along rays for voxels in lower resolution blocks, using adaptive step lengths based on the sampled block's resolution to avoid missing occluding features [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The performance of LAO computation is slightly superlinear with respect to data reduction [S2, unofficial]. The reduction increases the number of low-resolution blocks, decreasing the number of voxels and allowing longer sample steps, reducing average time per occlusion calculation [S2, unofficial].
*   **Memory Complexity:** Requires storage for the packed volume texture, index textures, ambient light texture, and direction texture. The reverse mapping index texture size is reduced by restricting minimum LOD [S2, unofficial].
*   **Measured Performance:** Tested on a volume with native resolution $512^3$ voxels. LAO computation time per ray is slightly superlinear with respect to data reduction [S2, unofficial].
*   **Strengths:** Allows interactive frame rates; less sensitive to noisy data sets compared to gradient-based methods (e.g., diffuse shading); improves cross-section visibility at clip planes; incorporates semitransparency [S2, unofficial].
*   **Limitations:** High-frequency surface details are lost; requires multiple frames for full convergence if updated incrementally; adaptive sampling decreases parallelism [S2, unofficial].
*   **When to Use:** Medical DVR for diagnosis and surgical planning, specifically for MRI data sets, virtual autopsies, and clip plane rendering [S2, unofficial].
*   **When Not to Use:** When fine high-frequency surface details are critical and noise is low (combine with diffuse shading); when global shadows are required [S2, unofficial].

## Background
Local Ambient Occlusion was introduced by Hernell et al. in 2010 [S2, unofficial]. It builds on earlier local volumetric shadowing methods by Ropinski et al. [2008] and Hernell et al. [2007], which computed local histograms or integrated attenuated transfer function density over a surrounding sphere [S1, unofficial]. The incremental approach allows for progressive refinement during user interaction [S2, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Local Ambient Occlusion in Direct Volume Rendering, https://www.researchgate.net/publication/44596498_Local_Ambient_Occlusion_in_Direct_Volume_Rendering, fetched 2026-10-28 (unofficial)
