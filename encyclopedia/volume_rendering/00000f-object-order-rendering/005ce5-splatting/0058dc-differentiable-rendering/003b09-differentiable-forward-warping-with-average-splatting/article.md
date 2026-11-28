# Differentiable Forward Warping with Average Splatting

Also called: Average Splatting, Forward Warping

## Overview
Differentiable Forward Warping with Average Splatting synthesizes dynamic scenes by warping a canonical volumetric representation (e.g., a voxel grid or radiance field) from a reference time step to a target time step using estimated deformation flows [S1]. It solves the "many-to-one" mapping issue inherent in forward warping—where multiple source voxels map to the same target location, such as when an object shrinks—by fusing these contributions using a weighted average via a trilinear kernel [S1]. Unlike standard forward warping which may leave holes in the "one-to-many" case (expansion), this method specifically addresses the density accumulation conflict, often paired with an inpaint network for missing regions [S1]. It distinguishes itself from sibling methods like [Differentiable Robot Rendering](15698-differentiable-robot-rendering-dr-robot.md) by focusing on temporal deformation of canonical fields rather than static multi-view consistency or specific robotic sensor models [G].

## How it works
**Inputs:**
*   Source voxel grid $\Vgrid{}{S}$ (e.g., canonical radiance field) [S1].
*   Deformation flow field $f_{\text{S}\rightarrow\text{T}}$ mapping source positions to target positions [S1].

**Outputs:**
*   Target voxel grid $\Vgrid{}{T}$ with values fused from the source [S1].

**Data Structures:**
*   $\Vgrid{}{S}$ and $\Vgrid{}{T}$: 3D arrays indexed by integer coordinates $(x, y, z)$ [S1].
*   $f_{\text{S}\rightarrow\text{T}}[\mathbf{q}]$: A vector field where each entry at source index $\mathbf{q}$ is a 3D displacement vector $\in \mathbb{R}^3$ [S1].

**Algorithm:**
The warping function $\Vgrid{}{T} = \text{Fwarp}(\Vgrid{}{S}, f_{\text{S}\rightarrow\text{T}})$ is computed as follows [S1]:
1. Initialize $\Vgrid{}{T}$ with zeros (or appropriate background) [G].
2. For each voxel $\mathbf{q}$ in the source grid $\Vgrid{}{S}$:
   a. Calculate the deformed position: $\mathbf{q}' = \mathbf{q} + f_{\text{S}\rightarrow\text{T}}[\mathbf{q}]$ [S1].
   b. Determine the integer target indices $\mathbf{p}$ that fall within the support of the reconstruction kernel centered at $\mathbf{q}'$ [S1].
   c. For each target voxel $\mathbf{p}$ in the support:
      i. Compute the fractional offset vector $\mathbf{u} = \mathbf{q}' - \mathbf{p}$ [S1].
      ii. Compute the trilinear kernel weight $b[\mathbf{u}]$ [S1].
      iii. Accumulate the weighted source value into the numerator and the weight into the denominator of $\Vgrid{}{T}[\mathbf{p}]$ [S1].
3. Normalize $\Vgrid{}{T}[\mathbf{p}]$ by dividing the accumulated numerator by the accumulated denominator for all $\mathbf{p}$ [S1].

**Formulas:**
The value of a target voxel $\mathbf{p}$ is given by [S1]:
$$
\Vgrid{}{T}[\mathbf{p}] = \frac {\sum _{\forall \mathbf {q}\in \Vgrid {}{S}}b[\mathbf {u}]\cdot \Vgrid {}{S}[\mathbf {q}]}{\sum _{\forall \mathbf {q}\in \Vgrid {}{S}}^{}b[\mathbf {u}]}
$$
where the trilinear kernel weight $b[\mathbf{u}]$ is defined as [S1]:
$$
b[\mathbf {u}] = \prod _{i \in \{x,y,z\}} \text {max}(0, 1\!-\!\left |\mathbf {u}_i \right |)
$$
and the offset vector $\mathbf{u}$ is [S1]:
$$
\mathbf {u} = (\mathbf {q}+f_{\text {S}\rightarrow \text {T}}[\mathbf {q}]) - \mathbf {p}
$$
Here, $\mathbf{q}$ is the integer coordinate of a source voxel, $f_{\text{S}\rightarrow\text{T}}[\mathbf{q}]$ is the deformation flow vector at $\mathbf{q}$, $\mathbf{p}$ is the integer coordinate of a target voxel, and $\mathbf{u}_i$ is the $i$-th component of $\mathbf{u}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def forward_warp_average_splatting(source_grid, flow_field, target_shape):
    """
    Warps source_grid to target_grid using average splatting.
    
    Args:
        source_grid: 3D list of values indexed as [d][h][w].
        flow_field: 3D list of 3-tuples (dx, dy, dz) indexed as [d][h][w].
        target_shape: Tuple (D, H, W) defining the output grid dimensions.
        
    Returns:
        target_grid: 3D list with warped values.
    """
    # Initialize accumulators
    numerator = [[[0.0 for _ in range(target_shape[2])] for _ in range(target_shape[1])] for _ in range(target_shape[0])]
    denominator = [[[0.0 for _ in range(target_shape[2])] for _ in range(target_shape[1])] for _ in range(target_shape[0])]
    
    # Iterate over all source voxels
    for q_d in range(len(source_grid)):
        for q_h in range(len(source_grid[0])):
            for q_w in range(len(source_grid[0][0])):
                val = source_grid[q_d][q_h][q_w]
                flow = flow_field[q_d][q_h][q_w]
                
                # Calculate deformed position
                deformed_pos = (q_d + flow[0], q_h + flow[1], q_w + flow[2])
                
                # Determine bounding box of the kernel support
                # Trilinear kernel is non-zero when |u_i| < 1
                p_min = [int(math.floor(deformed_pos[i] - 1.0)) for i in range(3)]
                p_max = [int(math.floor(deformed_pos[i] + 1.0)) for i in range(3)]
                
                # Clip to target grid boundaries
                for i in range(3):
                    p_min[i] = max(0, min(p_min[i], target_shape[i] - 1))
                    p_max[i] = max(0, min(p_max[i], target_shape[i] - 1))
                
                # Iterate over target voxels in the support
                for p_d in range(p_min[0], p_max[0] + 1):
                    for p_h in range(p_min[1], p_max[1] + 1):
                        for p_w in range(p_min[2], p_max[2] + 1):
                            # Calculate fractional offset u
                            u = (deformed_pos[0] - p_d, deformed_pos[1] - p_h, deformed_pos[2] - p_w)
                            
                            # Calculate trilinear weight b[u]
                            weight = 1.0
                            for i in range(3):
                                weight *= max(0.0, 1.0 - abs(u[i]))
                            
                            # Accumulate
                            numerator[p_d][p_h][p_w] += weight * val
                            denominator[p_d][p_h][p_w] += weight
                            
    # Normalize
    target_grid = [[[0.0 for _ in range(target_shape[2])] for _ in range(target_shape[1])] for _ in range(target_shape[0])]
    for d in range(target_shape[0]):
        for h in range(target_shape[1]):
            for w in range(target_shape[2]):
                denom = denominator[d][h][w]
                target_grid[d][h][w] = numerator[d][h][w] / denom if denom != 0.0 else 0.0
                    
    return target_grid
```

**Parameters:**
*   Kernel support: Trilinear (support size 2x2x2 in terms of integer grid steps, covering the immediate 8 neighbors) [S1].
*   Flow estimation: The flow $f_{\text{S}\rightarrow\text{T}}$ is typically estimated to be smooth and continuous along the timeline [S1].

**Edge Cases and Pitfalls:**
*   **Many-to-One:** When multiple source voxels map to the same target voxel (e.g., object shrinking), average splatting correctly fuses them [S1].
*   **One-to-Many:** When one source voxel maps to multiple target voxels (e.g., object expanding), holes may appear in the target grid; the source notes indicate an inpaint network is used to handle this [S1].
*   **Boundary Conditions:** Source voxels near the boundary may warp outside the target grid bounds; the implementation must clip coordinates to valid grid indices [G].

## Subtopics

### Forward Warping with Subpixel Offsets
A variant used in multi-view warping contexts (e.g., Depth Peeling) to handle warping seams caused by source texel enlargement in the novel view [S2]. It uses four subpixel samples (subpixel offsets) to avoid seams, as no simpler solution is known [S2]. It also extends the field of view for Depth Peeling by scaling the screen dimension by a constant factor (1.2) to handle corrupt periphery due to large view displacements and lack of source pixels at image boundaries [S2].

## Operating characteristics
*   **Complexity:** The complexity is proportional to the number of source voxels multiplied by the kernel support size (constant, 8 for trilinear). $O(N \cdot K)$ where $N$ is the number of source voxels and $K$ is the kernel footprint [G].
*   **Performance:** The method is differentiable, allowing gradients to flow back through the warping operation to the flow field and source grid, enabling end-to-end training [S1].
*   **Strengths:** Handles the many-to-one problem naturally without leaving artifacts or requiring complex conflict resolution logic beyond weighting. Ensures smooth flows by assuming continuous deformation [S1].
*   **Limitations:** Does not inherently solve the one-to-many problem (holes); requires additional inpainting or sampling strategies [S1]. Performance may degrade if the deformation field is highly discontinuous or if the kernel support is large [G].

## Background
The method was introduced in the context of novel view synthesis for dynamic scenes, specifically in the paper "Forward Flow for Novel View Synthesis of Dynamic Scenes" [S1]. It builds on the concept of forward warping, which estimates deformation from a canonical time to another time step [S1]. The specific "average splatting" technique was proposed to address the many-to-one mapping issue in forward warping of voxel grids [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Potentially Visible Hidden-Volume Rendering for Multi-View Warping, https://doi.org/10.1145/3592108, fetched 2026-11-08
