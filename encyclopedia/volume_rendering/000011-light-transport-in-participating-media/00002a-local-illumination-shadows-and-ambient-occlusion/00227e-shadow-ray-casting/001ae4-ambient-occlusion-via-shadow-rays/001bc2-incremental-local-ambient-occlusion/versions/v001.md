# Incremental Local Ambient Occlusion

Also called: LAO, LAO-I

## Overview
Incremental Local Ambient Occlusion (LAO) is a shading model for direct volume rendering that enhances ambient lighting by computing incident light from a local spherical neighborhood around each voxel, rather than using a constant ambient factor or global shadows [S2, unofficial]. It solves the problem of missing local shadowing and context in volume rendering while avoiding global shadows that obscure relevant internal structures [S2, unofficial]. Unlike traditional "all-or-nothing" ambient occlusion methods that treat voxels as opaque blockers, LAO incorporates semitransparency by integrating opacity along rays, and it uses an incremental update scheme to allow interactive frame rates during computation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function (TF) mapping density to opacity $\alpha$ and color, and a set of ray directions distributed over a sphere.
*   **Output:** An ambient light intensity texture (3D texture) storing the accumulated occlusion values $A(x)$ for each voxel, which is combined with the emissive color during the final rendering pass.

**Data Structures:**
1.  **Packed Volume Texture:** The regular linear volume is divided into blocks (typically $16^3$ voxels). Each block is encoded with multiple levels of detail (LOD): $8^3, 4^3, 2^3, 1^3$ voxels. These are packed into a single 3D texture.
2.  **Forward Mapping Index Texture:** Maps packed texture coordinates to original volume coordinates.
3.  **Reverse Mapping Index Texture:** Maps original volume coordinates back to packed texture coordinates. To reduce size, the minimum LOD for non-empty blocks is often restricted to $4^3$ voxels, making this texture 64 times smaller than the packed volume [S2, unofficial].
4.  **Ambient Light Texture:** Stores the accumulated intensity $A(x)$. It has the same dimensions and multiresolution layout as the packed volume texture [S2, unofficial].
5.  **Direction Texture:** Stores the ray directions and their associated weights $w_k$ [S2, unofficial].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Divide the volume into blocks of size $16^3$ voxels.
    *   Generate LOD representations ($8^3, 4^3, 2^3, 1^3$) for each block.
    *   Pack the volume data into the Packed Volume Texture.
    *   Create the Forward and Reverse Mapping Index Textures.
    *   Generate ray directions by subdividing a tetrahedron, icosahedron, or octahedron to achieve the desired number of rays $K$. Store these directions and weights $w_k$ in the Direction Texture [S2, unofficial].

2.  **Incremental Update Loop:**
    *   Initialize the Ambient Light Texture to zero ($A_0 = 0$) [S2, unofficial].
    *   For each ray direction $k$ from $1$ to $K$:
        *   Render a quad covering the entire framebuffer, where each pixel maps to one voxel in the mapped 3D slice of the Packed Volume Texture.
        *   In the fragment shader, for each voxel $x$:
            *   Determine the current LOD block containing $x$.
            *   Calculate the number of samples $M$ based on the block's resolution. For the highest resolution, $M=15$ is typical [S2, unofficial].
            *   Compute the light contribution $A_k(x)$ for direction $k$ using the discretized integral:
                $$ A_k(x) = \sum_{m=1}^{M} g_m \prod_{i=1}^{m-1} (1 - d_i) $$
                where $g_m = 1$ (due to scaling), and $d_i$ is the opacity at sample position $i$ derived from the transfer function [S2, unofficial].
            *   The sample positions are determined by the ray direction and the neighborhood radius $R_b$. The integration range is from a small offset $a$ (to avoid self-occlusion) to $R_b$ [S2, unofficial].
            *   Update the accumulated intensity in the Ambient Light Texture using the OpenGL blend operation:
                $$ A_k = \frac{1}{k} A_{k-1} + \left(1 - \frac{1}{k}\right) A_{k-1} $$
                *Note: The source text contains a garbled formula `Ak¼1 kAkþ11 k  Ak1;ð9Þ`. The standard incremental average formula implied by the text description "adds a fraction of the intensity" and the context of averaging $k$ directions is:*
                $$ A_{new} = \frac{1}{k} A_{old} + \frac{k-1}{k} A_{prev\_accumulated} $$
                *However, the text explicitly states $A_k$ is the accumulated intensity after $k$ directions and $A_0=0$. The blend operation adds a fraction. A common incremental mean update is:*
                $$ A_k = \frac{1}{k} A_{k\_current} + \frac{k-1}{k} A_{k-1} $$
                *Where $A_{k\_current}$ is the result of the ray trace for direction $k$ (which is $A_k(x)$ in the notation of Eq 7, but here $A_k$ in Eq 9 refers to the contribution of the k-th ray). Let $C_k$ be the contribution of ray $k$. Then:*
                $$ A_k = \frac{1}{k} C_k + \frac{k-1}{k} A_{k-1} $$
                *The source text's Eq 9 is garbled. Based on Eq 7 $A(x) = \frac{1}{K} \sum w_k A_k(x)$, if weights are uniform, the incremental update is an average. We assume the blend operation implements this averaging [S2, unofficial].*
        *   Interleave rendering with intensity texture updates to allow user interaction [S2, unofficial].

3.  **Final Rendering:**
    *   After processing all $K$ directions, the final ambient light intensity volume is used in the standard volume rendering pass.
    *   The final color is computed by combining the emissive color and the accumulated ambient occlusion [S2, unofficial].

**Formulas:**
*   **Light Contribution Integral:**
    $$ A_k(x) = \int_{a}^{R_b} g_A(s) \cdot e^{-\int_{a}^{s} \alpha(t) dt} ds $$
    where $A_k(x)$ is the light contribution for direction $k$ at voxel $x$, $R_b$ is the radius of the spherical neighborhood, $a$ is a small offset to avoid self-occlusion, $g_A(s)$ is the light contribution function, and $\alpha(t)$ is the extinction coefficient (opacity) [S2, unofficial].
*   **Volumetric Light Contribution Function:**
    $$ g_A(s) = \frac{1}{R_b} \cdot \alpha(s) + c_E(s) $$
    where $c_E(s)$ is an emissive color component derived from a separate transfer function or TF primitive. This formulation avoids sharp shadow boundaries associated with boundary-only light sources [S2, unofficial].
*   **Discretized Light Contribution:**
    $$ A_k(x) = \frac{1}{M} \sum_{m=1}^{M} g_m \prod_{i=1}^{m-1} (1 - d_i) $$
    where $M$ is the number of samples, $g_m$ is the discretized light contribution (equal to 1 due to scaling), and $d_i$ is the opacity at sample position $i$ according to the transfer function [S2, unofficial].
*   **Combined Local Ambient Occlusion:**
    $$ A(x) = \frac{1}{K} \sum_{k=1}^{K} w_k A_k(x) $$
    where $A(x)$ is the combined local ambient occlusion, $K$ is the number of ray directions, and $w_k$ is the weight for direction $k$ [S2, unofficial].

**Parameters:**
*   $R_b$: Radius of the surrounding LAO sphere. Typical values: 16 voxels, 32 voxels [S2, unofficial].
*   $M$: Number of samples for the highest resolution fixed sampling rate. Value: 15 [S2, unofficial].
*   $K$: Number of ray directions. Baseline for quality: 128 directions. Too few directions decrease the ambient effect [S2, unofficial].
*   Block Size: Typically $16^3$ voxels [S2, unofficial].
*   LOD Levels: $8^3, 4^3, 2^3, 1^3$ voxels per block [S2, unofficial].
*   Minimum LOD: Often restricted to $4^3$ voxels for non-empty blocks to reduce reverse index texture size [S2, unofficial].
*   Offset $a$: Small offset to avoid self-occlusion [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sharp Shadow Boundaries:** Using boundary light contribution ($g_A(s) = \delta(s-R_b)$) causes hard shadows if any occluder intersects the sphere. Solution: Use volumetric light contribution ($g_A(s) = \frac{1}{R_b}\alpha(s)$) to smooth shadows [S2, unofficial].
*   **Insufficient Directions:** Using too few ray directions decreases the ambient effect. 128 directions are recommended as a baseline [S2, unofficial].
*   **Loss of High-Frequency Details:** LAO alone may lose high-frequency surface details. Combining LAO with diffuse shading can help explore fine structures [S2, unofficial].
*   **Clip Plane Boundaries:** Boundaries of active volumes (clip planes) have poorly defined gradients, making gradient-based illumination inappropriate. LAO improves cross-section visibility in these areas [S2, unofficial].
*   **Absorption TF:** Improved context from separate absorption TF in LAO is only noticeable when the absorption of the tissue used in the rendering pass is lower than the LAO absorption [S2, unofficial].
*   **Interblock Interpolation:** Not considered in LAO computations to avoid excessive texture lookups, leading to minor artifacts at block boundaries [S2, unofficial].

## Subtopics

### Adaptive Sampling Density for LAO
A method to improve LAO performance by adjusting sampling density along rays for voxels in lower resolution blocks. It uses adaptive step lengths based on the sampled block's resolution to avoid missing occluding features. For a ray originating in a block of scale $\epsilon$, the step length is adjusted with $\epsilon_{max}/\epsilon$ for all samples along that ray, where $\epsilon_{max}$ is the maximum resolution. This reduces artifacts but increases execution time due to processing different numbers of samples for neighboring voxels and decreased parallelism [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The performance gain from multiresolution is significant. Data reduction via LOD selection directly implies performance gain because processing is driven by the multiresolution layout. The number of voxels is decreased, and low-resolution blocks permit longer sample steps, reducing the average time per occlusion calculation. The LAO computation time per ray is slightly superlinear with respect to data reduction [S2, unofficial].
*   **Memory Complexity:** Requires storage for the Packed Volume Texture, Forward/Reverse Mapping Index Textures, and the Ambient Light Texture. The reverse mapping index texture is optimized to be 64 times smaller than the packed volume by restricting minimum LOD [S2, unofficial].
*   **Measured Performance:** Tested on a volume with native resolution $512^3$ voxels. Performance is measured in frames per second (FPS) for LAO map creation. Adaptive sampling has slightly reduced performance compared to fixed sampling due to higher quality and decreased parallelism [S2, unofficial].
*   **Strengths:** Interactive frame rates; incorporates semitransparency; less sensitive to noisy data sets compared to gradient-based methods (e.g., diffuse shading); improves visibility at clip planes; allows user interaction during computation [S2, unofficial].
*   **Limitations:** Loses high-frequency surface details if used alone; requires recomputation when transfer functions change; does not support color bleeding; interblock interpolation is not supported [S1, unofficial][S2, unofficial].
*   **When to Use:** Medical DVR for diagnosis and surgical planning, specifically for MRI data sets, virtual autopsies, and clip plane rendering. Useful when local shape and density context is more important than global shadows or fine high-frequency details [S2, unofficial].
*   **When Not to Use:** When color bleeding is required; when fine high-frequency surface details are the primary focus and noise levels are low (diffuse shading may be better); when global shadows are necessary [S1, unofficial][S2, unofficial].

## Background
Local Ambient Occlusion was proposed by Hernell et al. in 2010 [S2, unofficial]. It builds on earlier ambient occlusion techniques such as Landis [Landis 2002] and Zhukov [Zhukov 1998], and volumetric versions like Vicinity Shading by Stewart [Stewart 2003] and Ropinski et al. [Ropinski 2008]. The incremental approach allows for interactive refinement, addressing the computational cost of global shadowing in participating media [S2, unofficial]. It is primarily used in medical imaging for MRI data sets [S2, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Local Ambient Occlusion in Direct Volume Rendering, https://www.researchgate.net/publication/44596498_Local_Ambient_Occlusion_in_Direct_Volume_Rendering, fetched 2026-10-28 (unofficial)
