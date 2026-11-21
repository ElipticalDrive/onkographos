# Ray-cast Occlusion with Sampling Masks

Also called: Slab-based Occlusion Accumulation, VolumeManager::calcOcclusion, VolumeManager::initSlice, calcOcclusion, initSlice

## Overview
Ray-cast Occlusion with Sampling Masks is an optimized method for calculating volume occlusion values by using pre-generated sampling masks to efficiently update occlusion values along a ray direction, rather than re-sampling the entire sphere for each voxel [S1, unofficial]. It builds on [Ray Occlusion Test](5670-ray-occlusion-test.md) by providing the actual occlusion intensity values required for ray casting, specifically designed to accelerate the computation of local density averages in 3D volumes.

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume data structure (`m_volume`), a sampling radius (`rad` or `radius`), pre-computed sampling masks (`idx_add` and `idx_remove`), and a direction vector (`dir`) with step increment (`stepInc`).
- **Output:** An occlusion value for each voxel, stored in `m_occlusion`.

**Data Structures:**
- `m_volume`: The 3D volume grid. Access via `getVoxel(m_volume, pos)`.
- `idx_add` (`osg::Vec3Array*`): An array of 3D offset vectors representing voxels entering the sampling sphere as the ray progresses.
- `idx_remove` (`osg::Vec3Array*`): An array of 3D offset vectors representing voxels leaving the sampling sphere as the ray progresses.
- `occprev`: An array storing the occlusion value for the previous slice along the primary accumulation axis (e.g., y-slice or x-slice).
- `addprev`, `subtprev`: Accumulators for the net voxel intensity entering and leaving the sampling window along the secondary axes.
- `m_sphereSamples`: The count of voxels actually inside the sampling sphere (not the bounding box).

**Formulas:**
- Total voxels in cubic bounding box: $w = (2 \cdot rad + 1)^3$ [S1, unofficial].
- Net difference in intensity: $diff = \frac{add - subt}{m\_sphereSamples}$ [S1, unofficial].
- Occlusion update: $occ = diff + occprev[y-1]$ (for y-axis progression) or similar for other axes [S1, unofficial].

**Parameters:**
- `rad`: The radius of the spherical sampling kernel. Determines the size of the window used to calculate occlusion [S1, unofficial].
- `radius`: The radius of the sampling sphere [S1, unofficial].
- `dir`: The direction vector of the ray along which occlusion is accumulated [S1, unofficial].
- `stepInc`: The starting coordinate offset for the ray [S1, unofficial].

**Steps:**
The method typically involves two phases: initialization (base case) and incremental updates using masks. The implementation iterates through volume dimensions (z, x, y).

1. **Initialization:**
   - Initialize `occprev[y]` to store occlusion for the previous y-slice.
   - Initialize `addprev` and `subtprev` to 0.
   - For the base case (e.g., $x=0, y=0$), compute the full occlusion for the first voxel by summing all voxels in the sphere centered at `pos`. Store this in `occprev[0]` and set `m_occlusion` for that voxel. This step also sets `m_sphereSamples` [S1, unofficial].

2. **First Column Update ($x=0, y>0$):**
   - Calculate the difference between voxels entering the sphere at the top ($y+rad$) and leaving at the bottom ($y-rad$) along the y-axis.
   - Sum these into temporary accumulators `add` and `subt`.
   - Compute $diff = \frac{add - subt}{w}$ (Note: sources cite $w$ here, but logically it should be normalized by sample count or consistent with the base case; the source explicitly states $w$ in the formula section but uses $m\_sphereSamples$ in the diff formula. We follow the explicit formula $diff = (add - subt) / w$ as written in the source's formula section, but note the potential ambiguity with $m\_sphereSamples$) [S1, unofficial].
   - Update occlusion: $occ = diff + occprev[y-1]$.
   - Store result in `occprev[y]` and set `m_occlusion` voxel [S1, unofficial].

3. **First Row Reset ($y=0, x>0$):**
   - Reset `addprev` and `subtprev` to 0.
   - Calculate the difference between voxels entering at the right ($x+rad$) and leaving at the left ($x-rad$) along the x-axis for the current z-slice.
   - This prepares the accumulators for the main loop [S1, unofficial].

4. **Main Loop Update ($x>0, y>0$):**
   - Update `addprev` by adding voxels entering at $x+rad$ and subtracting voxels leaving at $x-rad$ for each $z$ in $[-rad, rad]$.
   - Update `subtprev` similarly.
   - This effectively slides the sampling window along the x-axis [S1, unofficial].
   - Compute final occlusion for current voxel: $diff = \frac{addprev - subtprev}{w}$ (or $m\_sphereSamples$ depending on normalization choice, source cites both contexts) [S1, unofficial].
   - $occ = occprev[y] + diff$.
   - Set `m_occlusion` voxel and update `occprev[y]` for the next y iteration [S1, unofficial].

5. **Alternative Mask-Based Update (Simplified View):**
   - For each voxel at `pos` (iterating y from 0 to $t$, x from 0 to $s$):
   - Base case ($x=0$): Compute initial occlusion using `getOcclusion(pos, radius)`. Set `m_occlusion` and store in `occold` [S1, unofficial].
   - Update case ($x>0$):
     - Define `prev` position as $(x-1, y, stepInc)$.
     - Initialize `add=0`, `rem=0`.
     - Subtract old voxels: Iterate through `rem_mask` (equivalent to `idx_remove`). For each offset, calculate `tmp = prev + offset`. Add `getVoxel(m_volume, tmp)` to `rem` [S1, unofficial].
     - Add new voxels: Iterate through `add_mask` (equivalent to `idx_add`). For each offset, calculate `tmp = pos + offset`. Add `getVoxel(m_volume, tmp)` to `add` [S1, unofficial].
     - Compute $diff = \frac{add - rem}{m\_sphereSamples}$ [S1, unofficial].
     - Update occlusion based on previous value [S1, unofficial].

**Pitfalls:**
- **Boundary Conditions:** Specific `if/else` branches are required for $x=0$ and $y=0$ to handle the initialization of the sliding window accumulators correctly, preventing out-of-bounds access or incorrect accumulation logic [S1, unofficial].
- **Normalization:** `m_sphereSamples` is set by `getOcclusion` and represents the number of voxels in the sampling sphere. It is used as the divisor for the difference calculation. Using $w$ (bounding box volume) vs $m\_sphereSamples$ (actual sphere volume) must be consistent with the base case calculation [S1, unofficial].

## Subtopics

### Full Sphere Occlusion Calculation
A method to calculate occlusion values for a volume, serving as the base case for the incremental mask-based method. It computes the average intensity of all voxels within a sphere of radius `rad` centered at `pos` [S1, unofficial].
- **Definition:** Computes `res` by iterating through a cube bounding the sphere ($x, y, z$ from $-rad$ to $rad$), incrementing `m_sphereSamples` for each valid voxel inside the sphere, and summing their values. Returns `res / m_sphereSamples` [S1, unofficial].
- **Parameters:** `radius` (sphere radius), `dir` (direction vector for sampling in `calcOcclusion`), `stepInc` (step increment, default 1) [S1, unofficial].
- **Usage:** Implemented in `VolumeManager.cpp`, used by the `VolumeManager` singleton to provide occlusion data for the ray casting renderer and to initialize values for the faster mask-based method [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The naive approach (Full Sphere) is $O(rad^3)$ per voxel. The Mask-based method reduces this to $O(rad^2)$ per voxel update (sliding along one axis) or $O(1)$ per voxel if masks are pre-computed and applied, as it only processes the boundary voxels entering/leaving the sphere [S1, unofficial].
- **Memory Complexity:** Requires storage for `idx_add` and `idx_remove` masks, which scale with the surface area of the sampling sphere ($O(rad^2)$). Additionally, `occprev` arrays require $O(volume\_size)$ memory [S1, unofficial].
- **Strengths:** Significantly faster than re-computing the full sphere sum for every voxel by reusing previous calculations and only updating the difference [S1, unofficial].
- **Limitations:** Requires pre-computation of masks. Boundary conditions are complex to implement correctly [S1, unofficial].
- **When to use:** When occlusion values are needed for all voxels in a volume for ray casting, and performance is critical [S1, unofficial].

## Background
The method originates from the `volume-occlusion-editor` project by MPanknin [S1, unofficial]. It is implemented within the OpenSceneGraph (OSG) ecosystem, utilizing `osg::Vec3Array` for mask storage and `VolumeManager` for singleton management of occlusion data [S1, unofficial]. The approach is designed to support ray casting renderers by providing efficient local density estimates [S1, unofficial].

## Sources
[S1] MPanknin/volume-occlusion-editor, https://github.com/MPanknin/volume-occlusion-editor, fetched 2026-10-26 (unofficial)
