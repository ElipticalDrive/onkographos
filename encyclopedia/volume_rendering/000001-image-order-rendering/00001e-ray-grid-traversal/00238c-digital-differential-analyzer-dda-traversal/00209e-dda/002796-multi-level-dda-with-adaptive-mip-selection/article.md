# Multi-level DDA with Adaptive Mip Selection

Also called: DDA-based volume sampling routine, Multi-level traversal scheme

## Overview
This method is a ray traversal and sampling algorithm for volume rendering that combines Digital Differential Analyzer (DDA) stepping with adaptive mip-level selection to optimize the trade-off between step size and sampling accuracy [S1, book]. It solves the problem of inefficient traversal in sparse or heterogeneous volumes by using a majorant texture at multiple resolutions to skip empty space, while dynamically adjusting the granularity of the search to locate collision points precisely [S1, book]. It sets itself apart from standard DDA approaches by avoiding execution divergence on GPUs through a branchless, iterative mip-level adjustment heuristic, and from pure delta tracking by providing deterministic spatial bounds for step sizes [S1, book].

## How it works
The algorithm performs a stochastic distance sampling loop using a majorant density texture [S1, book]. It maintains a target optical thickness (`tauToGo`) and steps through the volume using a DDA that intersects against "dilated" brick boundaries [S1, book]. The core innovation is the dynamic selection of the mip level for the majorant lookup at every step, allowing the algorithm to start coarse (large steps) and refine (smaller steps) based on proximity to a potential collision, without using conditional branches that would cause divergence on SIMT hardware [S1, book].

### Inputs and Outputs
*   **Inputs:**
    *   `rayOrigin`: The starting position of the ray [S1, book].
    *   `rayDir`: The direction of the ray [S1, book].
    *   `majorantTexture`: A multi-mipmap texture storing the maximum density (majorant) for bricks, organized into 4 mip levels (0 to 3) [S1, book].
    *   `densityTexture`: The high-resolution density texture used for final sampling [S1, book].
    *   `random()`: A function returning a uniform random number in $[0, 1)$ [S1, book].
*   **Outputs:**
    *   The algorithm returns the result of the collision handling (e.g., color contribution, new path vertex) or continues the path if a null collision occurs [S1, book].

### Data Structures
*   **Majorant Texture:** A 3D texture array or volume texture organized into mip levels [S1, book]. The source specifies a brick size of $8 \times 2^{\text{mip}}$ voxels [S1, book]. The texture stores the majorant density value for each brick [S1, book].
*   **DDA State:**
    *   `t`: Current distance along the ray [S1, book].
    *   `tauToGo`: Remaining optical thickness to accumulate before a collision [S1, book].
    *   `mip`: Current floating-point mip level index (range $[0.0, 3.0]$) [S1, book].

### Formulas and Parameters
*   **Target Optical Thickness:**
    $$ \tau_{\text{target}} = -\ln(1 - r) $$
    where $r$ is a uniform random number [S1, book]. This defines the distance to the next interaction in optical space [S1, book].
*   **Brick Dimension:**
    $$ \text{dim} = 8 \ll \text{mip} $$
    where $\text{mip}$ is the integer part of the current mip level [S1, book]. The base brick size is 8 voxels [S1, book].
*   **DDA Step Distance (`nextt`):**
    The distance to the next brick boundary intersection, calculated using dilated boundaries [S1, book].
    $$ \text{ofs} = \mathbf{ri} \odot \left( \left( \mathbf{ri} \ge 0 \right) ? (\text{dim} + 0.5) : (-0.5) - \mathbf{ro} \right) $$
    $$ \mathbf{tmax} = \lfloor \mathbf{pos} \odot (1.0 / \text{dim}) \rfloor \odot \text{dim} \odot \mathbf{ri} + \text{ofs} $$
    $$ \text{nextt} = \min(\mathbf{tmax}_x, \min(\mathbf{tmax}_y, \mathbf{tmax}_z)) $$
    Where:
    *   $\mathbf{ri} = 1.0 / \mathbf{rayDir}$ (inverse ray direction) [S1, book].
    *   $\mathbf{ro}$ is the ray origin in the same coordinate space as $\mathbf{pos}$ [S1, book].
    *   $\mathbf{pos}$ is the current ray position [S1, book].
    *   $\odot$ denotes component-wise multiplication [S1, book].
    *   The dilation of $0.5$ voxels ensures the ray oversteps into the brick, avoiding numerical instability at exact boundaries [S1, book].
*   **Optical Thickness Traversed:**
    $$ \Delta \tau = \rho_{\text{majorant}} \cdot \Delta t $$
    where $\rho_{\text{majorant}}$ is the sampled majorant density and $\Delta t = \text{nextt} - t$ [S1, book].

### Parameters
*   **Initial Mip Level:** 3.0 (coarsest resolution, largest steps) [S1, book].
*   **Mip Increment:** 0.33 per iteration [S1, book].
*   **Mip Decrement on Collision:** 2.0 [S1, book].
*   **Mip Bounds:** Clamped to $[0.0, 3.0]$ [S1, book].
*   **Brick Dilation:** 0.5 voxels [S1, book].

### Pseudocode
The following Python-like pseudocode implements the core loop described in the source [S1, book]. It assumes helper functions `lookupMajorant`, `lookup`, and `random` are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random as std_random

def random():
    """Returns a uniform random float in [0, 1)."""
    return std_random.random()

def lookup_majorant(pos, mip_level, texture):
    """Samples the majorant density texture at the given mip level."""
    # [G] Standard texture sampling operation.
    return texture.sample(pos, mip_level)

def lookup(pos, texture):
    """Samples the high-resolution density texture."""
    # [G] Standard texture sampling operation.
    return texture.sample(pos, 0)

def handle_real_collision(pos, density, ray_dir):
    """Handles scattering, shading, and throughput update for a real collision."""
    # [G] Renderer-specific shading logic.
    pass

def step_dda_dilated(ray_origin, inv_ray_dir, pos, mip):
    """Computes distance to next brick boundary using dilated boundaries."""
    dim = 8 << mip
    # Component-wise offset calculation for 0.5 voxel dilation
    # ofs = ri * ( ((ri >= 0) ? dim + 0.5 : -0.5) - ro )
    ofs_x = inv_ray_dir.x * (((dim + 0.5) if inv_ray_dir.x >= 0 else -0.5) - ray_origin.x)
    ofs_y = inv_ray_dir.y * (((dim + 0.5) if inv_ray_dir.y >= 0 else -0.5) - ray_origin.y)
    ofs_z = inv_ray_dir.z * (((dim + 0.5) if inv_ray_dir.z >= 0 else -0.5) - ray_origin.z)
    # tmax calculation
    tmax_x = math.floor(pos.x * (1.0 / dim)) * dim * inv_ray_dir.x + ofs_x
    tmax_y = math.floor(pos.y * (1.0 / dim)) * dim * inv_ray_dir.y + ofs_y
    tmax_z = math.floor(pos.z * (1.0 / dim)) * dim * inv_ray_dir.z + ofs_z
    return min(tmax_x, min(tmax_y, tmax_z))

def multi_level_dda_sampling(ray_origin, ray_dir, majorant_texture, density_texture):
    r = random()
    tau_to_go = -math.log(1.0 - r)
    mip = 3.0
    inv_ray_dir = 1.0 / ray_dir
    t = 0.0

    while True:
        pos = ray_origin + t * ray_dir
        mip_int = round(mip)
        majorant = lookup_majorant(pos, mip_int, majorant_texture)
        next_t = step_dda_dilated(ray_origin, inv_ray_dir, pos, mip_int)
        mip = min(3.0, mip + 0.33)
        dt = next_t - t
        dtau = majorant * dt
        t = next_t
        tau_to_go -= dtau

        if tau_to_go > 0:
            continue

        mip = max(0.0, mip - 2.0)
        t += dt * (tau_to_go / dtau)
        collision_pos = ray_origin + t * ray_dir
        density = lookup(collision_pos, density_texture)

        if random() * majorant < density:
            return handle_real_collision(collision_pos, density, ray_dir)
        else:
            r = random()
            tau_to_go = -math.log(1.0 - r)
```

### Edge Cases and Pitfalls
*   **Numerical Instability at Boundaries:** Standard DDA can fail or produce jitter when a ray intersection falls exactly on a brick boundary [S1, book]. The algorithm mitigates this by dilating brick boundaries by 0.5 voxels in the direction of travel [S1, book]. This ensures the ray always "oversteps" into the brick, avoiding exact boundary hits [S1, book].
*   **Execution Divergence:** Implementing separate loops for each mip level causes significant divergence on GPUs (SIMT) [S1, book]. The adaptive mip selection heuristic (incrementing/decrementing a float variable) allows all threads to execute the same loop body, minimizing divergence [S1, book].
*   **Null Collisions:** If a collision is detected in the majorant but not in the actual density (null collision), the traversal must restart with a new random target [S1, book]. This can lead to inefficiency if the majorant is a poor approximation of the density [S1, book]. The multi-level approach helps by allowing finer steps when close to a collision, improving the majorant's accuracy [S1, book].
*   **Mip Level Bounds:** The mip level is clamped between 0.0 and 3.0 [S1, book]. Starting at 3.0 ensures large initial steps [S1, book]. Decrementing by 2.0 on collision ensures a significant refinement to locate the exact point [S1, book].

## Operating characteristics
*   **Time Complexity:** The number of steps is reduced compared to a regular DDA with global majorants [S1, book]. The source reports an average of 3 steps per path on the Moana cloud scene, compared to 5 steps for a regular DDA [S1, book]. The adaptive mip selection reduces texture lookups in empty space regions [S1, book].
*   **Memory Complexity:** The method requires storing a multi-mipmap majorant texture [S1, book]. The source uses 4 mip levels [S1, book]. The memory usage for the Moana cloud scene was 86.7 MB (using BC4 compression) [S1, book].
*   **Measured Performance:** On an NVIDIA RTX 3090, rendering one sample per pixel at 1920x1080 with 3 bounces, the method achieved 3.9 ms for the Moana cloud scene [S1, book].
*   **Strengths:**
    *   Reduces the number of volume lookups and DDA steps, especially in empty space [S1, book].
    *   Branchless mip selection avoids GPU execution divergence [S1, book].
    *   Dilated boundaries improve numerical stability [S1, book].
*   **Limitations:**
    *   Requires a multi-mipmap majorant texture, increasing memory usage [S1, book].
    *   Performance depends on the quality of the majorant; a poor majorant leads to more null collisions and restarts [S1, book].
    *   The heuristic for mip selection (0.33 increment, 2.0 decrement) is empirical and may not be optimal for all scenes [S1, book].

## Background
The method was introduced in the book *Efficient Unbiased Volume Path Tracing on the GPU* [S1, book]. It builds upon the concept of delta tracking and ratio tracking for volume rendering, adapting them to a DDA-based traversal framework [S1, book]. The key contribution is the integration of adaptive mip-level selection into the DDA loop to balance step size and accuracy without branching [S1, book]. The technique was evaluated against a baseline path tracer using NanoVDB and delta tracking, demonstrating significant speedups (up to 2.58x) and memory savings (using BC4 compression) on complex cloud scenes [S1, book].

## Sources
[S1] Efficient Unbiased Volume Path Tracing on the GPU, https://doi.org/10.1007/978-1-4842-7185-8_43, fetched 2026-10-24 (book)
