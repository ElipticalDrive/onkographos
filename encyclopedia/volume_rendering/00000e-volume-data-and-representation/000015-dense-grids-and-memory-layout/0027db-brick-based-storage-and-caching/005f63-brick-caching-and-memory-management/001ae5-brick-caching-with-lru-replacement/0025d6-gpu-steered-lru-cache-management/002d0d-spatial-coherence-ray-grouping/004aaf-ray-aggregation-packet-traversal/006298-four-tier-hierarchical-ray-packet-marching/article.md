# Four-Tier Hierarchical Ray Packet Marching


## Overview
Four-Tier Hierarchical Ray Packet Marching (HRM) is a spatial skipping technique that operates on Ray Packets (RPs) encompassing multiple rays to increase parallelism, unlike single-ray methods like HESS [S1]. It solves the problem of inefficient traversal in dense neural radiance fields by integrating an Axis-Aligned Bounding Box (AABB) module that utilizes a Gaussian filter to mitigate noise in occupancy grids, thereby constraining occupancy grid bitmap checks to voxels within the AABB [S1]. This approach compresses traversal paths, leading to a 70.4% reduction in computational load during coarse traversal and a 26.0% reduction in Empty Volume Traversal (EMA) operations compared to standard row-order scanning [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rays grouped into Ray Packets (RPs); an occupancy grid with associated feature vectors; a global RP buffer.
*   **Output:** Pixel values and their coordinates for the reconstructed new-view image; transmittance values for each ray.

**Data Structures:**
*   **Ray Packet (RP):** Contains a group of rays. Each RP is assigned a unique pointer serving as an address in the global RP buffer and an identifier [S1].
*   **Occupancy Grid:** A bitmap indicating voxel occupancy. Noise is mitigated via a Gaussian filter applied to this grid [S1].
*   **Coarse Voxel (CV):** The unit of spatial partitioning for the Tiny MLP Engine (TME). Each CV is assigned an independent fully connected network [S1].

**The Four Tiers:**
The HRM framework comprises four tiers: coarse, fine, leaf, and micro traversal [S1].

1.  **Coarse Traversal:** Uses the AABB module to constrain checks. The Gaussian-filtered occupancy grid is checked only for voxels within the AABB. This significantly reduces the search space [S1].
2.  **Fine Traversal:** Handles the transition from coarse to finer resolution. A critical challenge here is ray divergence, where rays in the same RP propagate into distinct voxels (e.g., three rays enter voxel A while one enters voxel B) [S1].
3.  **Leaf Traversal:** Operates at the voxel level, processing samples within the active voxels identified by previous tiers.
4.  **Micro Traversal:** The final stage, likely operating at the sample level within the leaf voxels, feeding into the Tiny MLP Engine (TME) [S1].

**Ray Divergence Handling (Lag-First Traversal):**
To mitigate throughput degradation caused by rays entering distinct voxels, HRM employs a lag-first traversal strategy [S1].
*   When divergence occurs (e.g., during fine traversal), the framework selectively advances rays that are within the same voxel.
*   Rays propagating into distinct voxels are temporarily deferred [S1].
*   This active synchronization promotes convergence with leading rays, restoring ray-level parallelism and reducing redundant EMA operations [S1].

**Tiny MLP Engine (TME) and Rendering:**
*   Interpolated Feature Vectors (IFVs) are processed by the TME to compute sample color and density [S1].
*   The TME uses a spatial partitioning strategy similar to KiloNeRF, assigning each CV an independent fully connected network [S1].
*   The Volume Rendering Unit (VRU) integrates sample colors along the ray to compute pixel values and transmittance [S1].

**Termination Conditions:**
An RP is considered fully processed when either:
1.  The transmittance of all remaining samples drops below a predefined threshold.
2.  All its rays exit the AABB [S1].

**Execution Flow Control:**
Two feedback signals regulate execution: `retire` and `reschedule` [S1]. The `retire` path is triggered upon RP termination (conditions 1 and 2 above) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_ray_packets(global_rp_buffer, occupancy_grid, aabb):
    # Filter occupancy grid to reduce noise
    filtered_grid = gaussian_filter(occupancy_grid)
    
    # Determine number of RPs based on buffer capacity
    num_rps = global_rp_buffer.capacity
    
    for rp_id in range(num_rps):
        rp = global_rp_buffer.get_rp(rp_id)
        
        # Check if RP is already fully processed
        if is_rp_fully_processed(rp, aabb):
            continue
            
        # 1. Coarse Traversal with AABB constraint
        active_coarse_voxels = find_voxels_in_aabb(rp, aabb, filtered_grid)
        
        # 2. Fine Traversal with Lag-First Divergence Handling
        # Group rays by their current voxel
        voxel_groups = group_rays_by_voxel(rp.rays)
        
        for voxel, rays_in_voxel in voxel_groups.items():
            if voxel in active_coarse_voxels:
                # Advance rays in the same voxel
                process_rays_in_voxel(rays_in_voxel, voxel, rp)
            else:
                # Defer rays in distinct voxels (Lag-First)
                defer_rays(rays_in_voxel, rp)
        
        # 3. Leaf and Micro Traversal
        # Process samples for active rays
        for ray in rp.active_rays:
            samples = generate_samples(ray, aabb)
            for sample in samples:
                # Get Coarse Voxel for this sample
                cv = get_coarse_voxel(sample.position)
                # Use independent MLP for this CV
                color, density = tiny_mlp_engine(cv, sample.position)
                # Integrate into VRU
                vru.integrate(ray, color, density)
        
        # 4. Termination Check
        if is_rp_fully_processed(rp, aabb):
            # Trigger retire signal
            retire(rp)
            # Output pixel values and coordinates
            output_pixels(rp)
        else:
            # Trigger reschedule signal for next iteration
            reschedule(rp)

def is_rp_fully_processed(rp, aabb):
    # Condition 1: Transmittance threshold
    if all(ray.transmittance < threshold for ray in rp.rays):
        return True
    # Condition 2: All rays exit AABB
    if all(ray.position not in aabb for ray in rp.rays):
        return True
    return False
```

## Subtopics

### [Lag-First Traversal](18635-lag-first-traversal.md)
Lag-first traversal is a ray divergence handling technique used within Ray Packet (RP) marching to mitigate throughput degradation caused by rays entering distinct voxels. It selectively advances lagging rays within an RP to promote convergence with leading rays, thereby restoring ray-level parallelism and reducing redundant Empty Volume Traversal (EMA) operations. Unlike standard packet marching which may stall or process divergent rays inefficiently, this method actively synchronizes the group's spatial position.

### Tiny MLP Engine (TME) with Spatial Partitioning
An MLP engine that employs a spatial partitioning strategy similar to KiloNeRF, assigning each Coarse Voxel (CV) an independent fully connected network [S1]. Each Ray Packet (RP) is assigned a unique pointer serving as an address in the global RP buffer and an identifier. The number of RPs under processing is determined by the global RP buffer capacity. An RP is considered fully processed once the transmittance of all remaining samples drops below a predefined threshold or all its rays exit the AABB [S1].

## Operating characteristics
*   **Performance:** HRM achieves a 70.4% reduction in computational load during coarse traversal and a 26.0% reduction in EMA operations compared to row-order scanning [S1].
*   **Parallelism:** By operating on RPs, it increases parallelism compared to single-ray methods like HESS [S1].
*   **Hardware:** Designed for an energy-efficient edge coprocessor for neural rendering [S1].
*   **Limitations:** Requires managing a global RP buffer and handling complex divergence logic (Lag-First) which may add control overhead. The reliance on a Gaussian filter for occupancy grid noise mitigation adds preprocessing cost [S1].

## Background
HRM was proposed in the paper "An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies" [S1]. It addresses the limitations of single-ray traversal methods (like HESS) by leveraging ray packet processing and hierarchical traversal with spatial coherence [S1]. The method integrates concepts from KiloNeRF for spatial partitioning of MLPs [S1].

## Sources
[S1] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
