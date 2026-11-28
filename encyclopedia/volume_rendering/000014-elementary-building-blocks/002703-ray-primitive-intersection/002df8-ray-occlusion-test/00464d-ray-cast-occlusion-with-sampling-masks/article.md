# Ray-cast Occlusion with Sampling Masks

Also called: Slab-based Occlusion Accumulation, VolumeManager::calcOcclusion, VolumeManager::initSlice, calcOcclusion, initSlice

## Overview
Ray-cast Occlusion with Sampling Masks is an optimized method for calculating volume occlusion values by using pre-generated sampling masks to efficiently update occlusion values along a ray direction, rather than re-sampling the entire sphere for each voxel [S1, unofficial]. It builds on [Ray Occlusion Test](5670-ray-occlusion-test.md) by providing the actual occlusion intensity values required for ray casting, specifically designed to accelerate the computation of local density averages in 3D volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume data structure (`m_volume`), a sampling radius (`rad` or `radius`), pre-computed sampling masks (`idx_add` and `idx_remove`), and a direction vector (`dir`) with step increment (`stepInc`) [S1, unofficial].
- **Output:** An occlusion value for each voxel, stored in `m_occlusion` [S1, unofficial].

**Data Structures:**
- `m_volume`: The 3D volume grid, accessed via `getVoxel(m_volume, pos)` [S1, unofficial].
- `idx_add` and `idx_remove`: `osg::Vec3Array*` objects containing offset vectors for voxels entering and leaving the sampling sphere as the ray progresses [S1, unofficial].
- `occprev`: An array storing the occlusion value for the previous slice along the primary accumulation axis (e.g., y-slice or x-slice) [S1, unofficial].
- `addprev`, `subtprev`: Accumulators for the net voxel intensity entering and leaving the sampling window along the secondary axes [S1, unofficial].
- `m_sphereSamples`: The count of voxels actually inside the sampling sphere [S1, unofficial].

**Formulas:**
- Total voxels in cubic bounding box: $w = (2 \cdot rad + 1)^3$ [S1, unofficial].
- Net difference in intensity: $diff = \frac{add - rem}{m\_sphereSamples}$ [S1, unofficial].
- The source specifies dividing by $w$ in intermediate slab-based updates, but the final formula section specifies dividing by $m\_sphereSamples$ (the actual voxel count inside the sphere) [S1, unofficial].

**Parameters:**
- `rad`: The radius of the spherical sampling kernel, determining the window size for occlusion calculation [S1, unofficial].
- `radius`: The radius of the sampling sphere [S1, unofficial].
- `dir`: The direction vector of the ray along which occlusion is accumulated [S1, unofficial].
- `stepInc`: The starting coordinate offset for the ray [S1, unofficial].

**Steps:**
The method typically involves initialization and incremental updates using masks [S1, unofficial]. The implementation iterates through volume dimensions $z$ from $0$ to `m_volume->r()`, $x$ from $0$ to `m_volume->s()`, and $y$ from $0$ to `m_volume->t()` [S1, unofficial].
1. **Initialization:** Initialize `occprev[y]` to store occlusion for the previous y-slice [S1, unofficial]. Initialize `addprev` and `subtprev` to $0$ [S1, unofficial]. For the base case ($x=0, y=0$), compute the full occlusion for the first voxel by summing all voxels in the sphere centered at `pos` [S1, unofficial]. Store this in `occprev[0]` and set `m_occlusion` for that voxel [S1, unofficial]. This step also sets `m_sphereSamples` [S1, unofficial].
2. **First Column Update ($x=0, y>0$):** Calculate the difference between voxels entering the sphere at the top ($y+rad$) and leaving at the bottom ($y-rad$) along the y-axis [S1, unofficial]. Sum these into temporary accumulators `add` and `subt` [S1, unofficial]. Compute $diff = \frac{add - subt}{w}$ [S1, unofficial]. Update occlusion: $occ = diff + occprev[y-1]$ [S1, unofficial]. Store result in `occprev[y]` and set `m_occlusion` voxel [S1, unofficial].
3. **First Row Reset ($y=0, x>0$):** Reset `addprev` and `subtprev` to $0$ [S1, unofficial]. Calculate the difference between voxels entering at the right ($x+rad$) and leaving at the left ($x-rad$) along the x-axis for the current z-slice [S1, unofficial]. This prepares the accumulators for the main loop [S1, unofficial].
4. **Main Loop Update ($x>0, y>0$):** Update `addprev` by adding voxels entering at $x+rad$ and subtracting voxels leaving at $x-rad$ for each $z$ in $[-rad, rad]$ [S1, unofficial]. Update `subtprev` similarly [S1, unofficial]. This effectively slides the sampling window along the x-axis [S1, unofficial]. Compute final occlusion for current voxel: $diff = \frac{addprev - subtprev}{w}$ [S1, unofficial]. $occ = occprev[y] + diff$ [S1, unofficial]. Set `m_occlusion` voxel and update `occprev[y]` for the next y iteration [S1, unofficial].
5. **Alternative Mask-Based Update:** For each voxel at `pos` (iterating $y$ from $0$ to $t()$, $x$ from $0$ to $s()$): Base case ($x=0$): Compute initial occlusion using `getOcclusion(pos, radius)` [S1, unofficial]. Set `m_occlusion` and store in `occold` [S1, unofficial]. Update case ($x>0$): Define `prev` position as $(x-1, y, stepInc)$ [S1, unofficial]. Initialize `add=0`, `rem=0` [S1, unofficial]. Subtract old voxels: Iterate through `rem_mask` (equivalent to `idx_remove`) [S1, unofficial]. For each offset, calculate `tmp = prev + offset` [S1, unofficial]. Add `getVoxel(m_volume, tmp)` to `rem` [S1, unofficial]. Add new voxels: Iterate through `add_mask` (equivalent to `idx_add`) [S1, unofficial]. For each offset, calculate `tmp = pos + offset` [S1, unofficial]. Add `getVoxel(m_volume, tmp)` to `add` [S1, unofficial]. Compute $diff = \frac{add - rem}{m\_sphereSamples}$ [S1, unofficial]. Update occlusion based on previous value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_occlusion_masks(volume, radius, idx_add, idx_remove, step_inc):
    occprev = [0.0] * volume.t()
    addprev = 0.0
    subtprev = 0.0
    m_sphere_samples = 0
    occold = 0.0

    for z in range(volume.r()):
        for x in range(volume.s()):
            for y in range(volume.t()):
                pos = (x, y, z)

                # Base case: first voxel
                if x == 0 and y == 0:
                    occold = get_occlusion(pos, radius, volume)
                    occprev[0] = occold
                    volume.set_occlusion(pos, occold)
                    continue

                # Alternative mask-based update (simplified view from source)
                if x == 0:
                    occold = get_occlusion(pos, radius, volume)
                    volume.set_occlusion(pos, occold)
                    continue

                # Update case (x > 0)
                prev_pos = (x - 1, y, step_inc)
                add_val = 0.0
                rem_val = 0.0

                # Subtract old voxels
                for offset in idx_remove:
                    tmp = (prev_pos[0] + offset.x, prev_pos[1] + offset.y, prev_pos[2] + offset.z)
                    rem_val += volume.get_voxel(tmp)

                # Add new voxels
                for offset in idx_add:
                    tmp = (pos[0] + offset.x, pos[1] + offset.y, pos[2] + offset.z)
                    add_val += volume.get_voxel(tmp)

                # Compute difference and update occlusion
                diff = (add_val - rem_val) / m_sphere_samples
                occ = occold + diff
                volume.set_occlusion(pos, occ)
                occold = occ
```

**Pitfalls:**
- **Boundary Conditions:** Specific `if/else` branches are required for $x=0$ and $y=0$ to handle the initialization of the sliding window accumulators correctly, preventing out-of-bounds access or incorrect accumulation logic [S1, unofficial].
- **Normalization:** `m_sphereSamples` is set by `getOcclusion` and represents the number of voxels in the sampling sphere, used as the divisor for the difference calculation [S1, unofficial].

## Subtopics

### Full Sphere Occlusion Calculation
A method to calculate occlusion values for a volume, serving as the base case for the incremental mask-based method [S1, unofficial]. It computes the average intensity of all voxels within a sphere of radius `rad` centered at `pos` [S1, unofficial].
- **Definition:** Computes `res` by iterating through a cube bounding the sphere ($x, y, z$ from $-rad$ to $rad$), incrementing `m_sphereSamples` for each valid voxel inside the sphere, and summing their values [S1, unofficial]. Returns `res / m_sphereSamples` [S1, unofficial].
- **Parameters:** `radius` (sphere radius), `dir` (direction vector for sampling in `calcOcclusion`), `stepInc` (step increment, default $1$) [S1, unofficial].
- **Usage:** Implemented in `VolumeManager.cpp`, used by the `VolumeManager` singleton to provide occlusion data for the ray casting renderer and to initialize values for the faster mask-based method [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method originates from the `volume-occlusion-editor` project by MPanknin [S1, unofficial]. It is designed to support ray casting renderers by providing efficient local density estimates [S1, unofficial].

## Sources
[S1] MPanknin/volume-occlusion-editor, https://github.com/MPanknin/volume-occlusion-editor, fetched 2026-10-26 (unofficial)
