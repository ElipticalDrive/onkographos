# Volume Ambient Occlusion

Also called: AO Field, Occlusion Field

## Overview
Volume Ambient Occlusion computes a scalar occlusion field from a density field, quantifying how much of the surrounding environment each voxel can "see" [S1]. It is primarily used to accelerate cloud rendering by approximating multiple scattering effects through a remapped occlusion field, offering a balance between efficiency and visual quality despite not being physically accurate [S1]. Unlike [Positional & Normal-Based Voxel Shadowing](6206-positional-normal-based-voxel-shadowing.md), which calculates directional shading based on face normals and block types, this method computes a holistic, non-directional occlusion value per voxel [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A volume or VDB density field [S1].
*   **Output:** The input density field with an additional scalar field named `occlusion` [S1]. The `occlusion` field stores values where 0 indicates no occlusion and 1 indicates full occlusion [S1].

**Data Structures**
*   **Density Field:** The source field containing density values.
*   **Occlusion Field:** A scalar field with the same spatial domain as the density field (or a downsampled version based on `Resolution Ratio`).
*   **VDB Structure:** If the input is a VDB, the occlusion field is stored within the VDB structure. The `Expand Voxels` parameter applies here to pad the active area [S1].

**Parameters**
*   **Resolution Ratio:** Defines the occlusion field resolution as a ratio of the density field resolution. A value of 0.5 yields half resolution. Smaller values are more efficient but less accurate [S1].
*   **Expand Voxels:** Expands the active area of the occlusion field by a specified number of voxels to prevent sampling outside the computed area during interpolation at render time. This parameter is only considered for VDBs; dense volumes are not affected [S1].
*   **Density Scale:** A multiplier applied to the density field for the occlusion computation only. The original density field remains untouched. A value of 0.1 makes the density appear ten times more transparent to the computation [S1].
*   **Transmission Cutoff:** The transmittance threshold below which a ray is considered fully occluded, terminating the marching process. A value of 0.005 means an occlusion value of 0.995 is treated as fully occluded. Larger values improve efficiency but reduce accuracy [S1].

**Algorithm: Volume Marching**
The occlusion value for each voxel is computed by marching rays in multiple directions through the density field.

1.  **Initialize:** For each voxel $v$ in the target grid:
    *   Let $d_{scale}$ be the `Density Scale`.
    *   Let $t_{cutoff}$ be the `Transmission Cutoff`.
    *   Initialize total occlusion $O = 0$.
    *   Define a set of ray directions $\mathcal{D}$ (typically a uniform distribution over the sphere, though the specific directions are not detailed in the source [G]).

2.  **Marching:** For each direction $\vec{d} \in \mathcal{D}$:
    *   Start a ray at voxel $v$ in direction $\vec{d}$.
    *   Initialize transmittance $T = 1.0$.
    *   March along the ray through the density field.
    *   At each step, sample the density $\rho$ from the input field.
    *   Apply the scale: $\rho' = \rho \times d_{scale}$.
    *   Update transmittance: $T = T \times e^{-\rho' \Delta s}$, where $\Delta s$ is the step size [G].
    *   If $T < t_{cutoff}$, break the loop (ray is fully occluded).

3.  **Accumulate:**
    *   The occlusion contribution for this direction is $1 - T$.
    *   Accumulate: $O = O + (1 - T)$.

4.  **Normalize:**
    *   The final occlusion value is the average over all directions:
        $$ \text{occlusion}(v) = \frac{1}{|\mathcal{D}|} \sum_{\vec{d} \in \mathcal{D}} (1 - T_{\vec{d}}) $$
    *   Clamp the result to $[0, 1]$ [G].

**Edge Cases and Pitfalls**
*   **VDB Padding:** When using VDBs, failing to set `Expand Voxels` may cause the renderer to sample undefined values at the boundaries during interpolation, leading to artifacts [S1].
*   **Density Scale Interaction:** A high `Density Scale` will cause rays to terminate quickly, potentially underestimating occlusion in sparse regions if `Transmission Cutoff` is not adjusted accordingly [S1].
*   **Resolution Trade-off:** Lower `Resolution Ratio` speeds up computation but reduces the accuracy of the occlusion field, which may result in blocky or inaccurate shading [S1].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of voxels in the occlusion field multiplied by the number of ray directions and the average ray length [S1].
*   **Memory Complexity:** Requires storage for the input density field and the output occlusion field. For VDBs, the `Expand Voxels` parameter increases the memory footprint of the active area [S1].
*   **Performance:** Smaller `Resolution Ratio` and larger `Transmission Cutoff` values improve efficiency but reduce accuracy [S1].
*   **Strengths:** Significantly accelerates cloud rendering by approximating multiple scattering effects [S1].
*   **Limitations:** Not physically accurate; relies on post-processing remapping for visual effect [S1].

## Background
Volume Ambient Occlusion was initially created to accelerate cloud rendering [S1]. It allows for the approximation of multiple scattering by remapping the occlusion field at render time, mapping dark values to cloud edges and bright values to the cloud interior [S1]. This technique was introduced in Houdini version 20.0 [S1].

## Sources
[S1] Volume Ambient Occlusion - SideFX, https://www.sidefx.com/docs/houdini/nodes/sop/volumeambientocclusion.html, fetched 2026-10-27
