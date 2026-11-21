# Anisotropic Voxelization via Projection

Also called: View-dependent voxelization, Projected anisotropic voxelization

## Overview
Anisotropic Voxelization via Projection addresses the inaccuracy of storing single density values without view direction consideration, which fails to represent correlated media accurately [S1]. It computes view-direction-dependent probabilities of events by projecting 3D voxel block grids onto 2D grids (XY, XZ, YZ planes) to produce anisotropic voxels [S1]. It sets itself apart from isotropic methods by storing directional data per axis via GPU shared memory projection, and from other anisotropic variants by explicitly managing the projection of sample positions to derive axis-specific values rather than simple averaging [S1].

## How it works
**Inputs:**
- A 3D voxel grid at a fine resolution (Level 3).
- A coarser voxel grid structure (Level 2) composed of blocks containing Level 3 voxels.
- View direction vectors for the rendering context.

**Outputs:**
- Anisotropic voxels at Level 2, containing directional density/probability information.

**Data Structures:**
- **Level 3 Voxel:** Contains an identifier and hit status.
- **Level 2 Block:** Contains a collection of Level 3 voxels.
- **Projected Grid:** 2D grid (XY, XZ, or YZ) storing projected sample positions and accumulated values per axis.

**Steps:**
1. **Projection Setup:** For each view direction, define the projection planes: XY, XZ, and YZ. Initialize GPU shared memory buffers for each plane to store projected positions and values per block [S1].
2. **Sample Projection:** Iterate through every sample in the Level 3 voxel grid. Project the 3D position of each sample onto the XY, XZ, and YZ planes [S1].
3. **Block-wise Accumulation:** Store the projected position of every sample by block to obtain a value per axis. This involves mapping the 3D coordinates to 2D indices in the shared memory buffers associated with the specific Level 2 block [S1].
4. **Occupancy Computation ($O$):** Instead of projecting data to the axis for occupancy, analyze the third-level data directly. Compute the occupancy $O$ for a Level 2 block by dividing the number of different identifiers (hit voxels) by the total amount of voxels in the third level of voxelization within that block [S1].
   $$ O = \frac{\sum_{v \in \text{block}_3} \text{hit}(v)}{\text{Resolution}_3} $$
   Where $\text{hit}(v)$ is 1 if the Level 3 voxel $v$ is occupied, and 0 otherwise, and $\text{Resolution}_3$ is the total number of voxels in the Level 3 block [S1].
5. **Anisotropic Value Derivation:** Use the projected positions and values stored per axis to compute the final anisotropic density. The method implies using the distribution of projected samples to define the directional properties, potentially linked to covariance matrices for SGGX (Spherical Gaussian Gaussian) distributions as referenced in related work [S1].
   $$ \lambda_1 = \sigma(\omega_1), \quad \lambda_2 = \sigma(\omega_2), \quad \lambda_3 = \sigma(\omega_3) $$
   Where $\omega_i$ are eigenvectors and $\lambda_i$ are eigenvalues equivalent to projected areas [S1].

**Parameters:**
- **Voxelization Levels:** Level 2 (coarse/anisotropic) and Level 3 (fine/occupancy). Typical values not given in the sources.
- **Projection Planes:** XY, XZ, YZ.

**Edge Cases and Pitfalls:**
- **Precision:** GPU shared memory limits may restrict block sizes.
- **Boundaries:** Projection must handle samples near block boundaries correctly to avoid double-counting or gaps.
- **Degenerate Input:** Empty blocks result in $O=0$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_voxelization(level3_voxels, level2_blocks, view_direction):
    # Initialize shared memory buffers for XY, XZ, YZ projections
    proj_xy = SharedMemoryBuffer()
    proj_xz = SharedMemoryBuffer()
    proj_yz = SharedMemoryBuffer()

    # Step 1 & 2: Project samples and store by block
    for block in level2_blocks:
        for voxel in block.level3_voxels:
            if voxel.is_hit():
                pos = voxel.position
                # Project to 2D planes
                xy_proj = (pos.x, pos.y)
                xz_proj = (pos.x, pos.z)
                yz_proj = (pos.y, pos.z)
                
                # Store projected position and value in shared memory
                # Implementation detail: Map to shared memory indices
                proj_xy.store(block.id, xy_proj, voxel.value)
                proj_xz.store(block.id, xz_proj, voxel.value)
                proj_yz.store(block.id, yz_proj, voxel.value)

    # Step 3: Compute occupancy O for Level 2 blocks
    for block in level2_blocks:
        hit_count = 0
        total_voxels = len(block.level3_voxels)
        
        for voxel in block.level3_voxels:
            if voxel.is_hit():
                hit_count += 1
        
        # Occupancy is percentage of hit voxels in Level 3
        block.occupancy = hit_count / total_voxels

    # Step 4: Derive anisotropic values from projections
    # Note: Specific formula for combining projections into final anisotropic
    # density is not fully detailed in sources, but relies on stored axis values.
    # Reference to SGGX matrix S is mentioned for distribution definition.
    for block in level2_blocks:
        block.anisotropic_data = compute_anisotropic_from_projections(
            proj_xy.get(block.id),
            proj_xz.get(block.id),
            proj_yz.get(block.id)
        )
        
    return level2_blocks
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Level 3 voxels, as each sample is projected once per plane.
- **Memory Complexity:** Proportional to the number of Level 2 blocks times the shared memory buffer size.
- **Performance:** Relies on GPU shared memory for efficient projection, suggesting high performance on GPU architectures.
- **Strengths:** Accurately represents correlated media and thin structures (e.g., cloth) where isotropic averaging fails.
- **Limitations:** Requires finer resolution Level 3 data; computational overhead of multiple projections.
- **When to use:** When view-dependent rendering accuracy is critical for thin or complex geometries.
- **When not to use:** For simple, thick volumes where isotropic approximation is sufficient.

## Background
Originated in the context of fast voxelization for microgeometry rendering, specifically addressing the limitations of single-value density storage [S1]. It builds on concepts from Crassin et al. [28] regarding 3D to 2D projection of voxel grids [S1]. The method is part of a broader effort to improve volumetric rendering accuracy for correlated media [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
