# Four-Tier Hierarchical Ray Packet Marching


## Overview
Four-Tier Hierarchical Ray Packet Marching (HRM) is a spatial skipping technique that operates on Ray Packets (RPs) encompassing multiple rays to increase parallelism, unlike single-ray methods like HESS [S1]. It solves the problem of inefficient traversal and redundant computation in neural volume rendering by integrating an Axis-Aligned Bounding Box (AABB) module with a four-tier traversal hierarchy [S1]. What sets it apart from its siblings is its specific handling of ray divergence within packets via a "lag-first" scheduler and its use of Gaussian-filtered occupancy grids to constrain bitmap checks, achieving significant reductions in computational load [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rays grouped into Ray Packets (RPs); an occupancy grid (bitmap); an Axis-Aligned Bounding Box (AABB) defining the region of interest; a Tiny MLP Engine (TME) with spatial partitioning.
*   **Output:** Pixel values and their coordinates for the reconstructed image; transmittance values for RP termination.

**Data Structures:**
*   **Ray Packet (RP):** Contains multiple rays. Each RP is assigned a unique pointer serving as an address in the global RP buffer and an identifier [S1].
*   **Occupancy Grid:** A bitmap indicating voxel occupancy. Noise is mitigated using a Gaussian filter [S1].
*   **Coarse Voxel (CV):** The grid is partitioned into Coarse Voxels. Each CV is assigned an independent fully connected network in the TME [S1].
*   **Global RP Buffer:** Stores active RPs. The number of RPs under processing is determined by the capacity of this buffer [S1].

**Steps:**
1.  **AABB Construction and Filtering:** Construct an AABB for the current view or packet group. Apply a Gaussian filter to the occupancy grid to mitigate noise. This constrains subsequent occupancy grid bitmap checks to voxels strictly within the AABB [S1].
2.  **Coarse Traversal:** Traverse the hierarchy at the coarsest level. Use the filtered occupancy grid to skip empty regions. The HRM framework processes RPs in bundles. During this stage, the system handles ray divergence by selectively advancing rays within the same voxel while temporarily deferring others that propagate into distinct voxels [S1].
3.  **Fine Traversal:** Refine the traversal within non-empty coarse voxels. Continue handling divergence. The scheduler employs a "lag-first" approach: when rays diverge into distinct voxels, lagging rays are selectively advanced to promote convergence with leading rays [S1].
4.  **Leaf Traversal:** Traverse the leaf nodes of the hierarchy. Maintain the RP structure.
5.  **Micro Traversal:** Perform the final sample evaluation. Interpolated feature vectors (IFVs) are processed by the Tiny MLP Engine (TME) to compute sample color and density [S1].
6.  **Volume Rendering:** The Volume Rendering Unit (VRU) integrates sample colors along the ray to compute pixel values and transmittance [S1].
7.  **Termination Check:** An RP is considered fully processed once the transmittance of all remaining samples drops below a predefined threshold or all its rays exit the AABB [S1].
8.  **Output:** After an RP completes rendering, four pixel values and their coordinates are output to reconstruct the new-view image [S1].

**Parameters:**
*   **Gaussian Filter:** Used for occupancy grid noise mitigation. Specific kernel size or sigma is not given in the sources [S1].
*   **Transmittance Threshold:** Predefined threshold for RP termination. Value not given in the sources [S1].
*   **RP Buffer Capacity:** Determines the number of RPs under processing. Value not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Ray Divergence:** Rays within the same RP may enter distinct voxels, diminishing ray-level parallelism and increasing redundant Early-Marching-Ahead (EMA) computations [S1]. The lag-first approach mitigates this but requires careful scheduling [S1].
*   **Boundary Redundancy:** At CV boundaries, repeated loading and eviction of identical CV data contribute to redundant EMAs. Reordering RPs based on CV tags alleviates this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def four_tier_hrm(rays, occupancy_grid, aabb, tme, vr_u, rp_buffer_capacity):
    # Filter occupancy grid to mitigate noise
    filtered_grid = gaussian_filter(occupancy_grid)
    
    # Group rays into Ray Packets (RPs)
    rps = create_ray_packets(rays)
    
    # Assign unique pointer and ID to each RP
    for rp in rps:
        rp.pointer = get_unique_address(rp_buffer_capacity)
        rp.id = generate_identifier()
        
    # Limit active RPs to buffer capacity
    active_rps = rps[:rp_buffer_capacity]
    
    # Reorder RPs based on Coarse Voxel (CV) tags to reduce redundancy
    active_rps = sort_by_cv_tag(active_rps)
    
    results = []
    
    for rp in active_rps:
        # Initialize transmittance for all rays in RP
        transmittance = initialize_transmittance(rp.rays)
        
        # Four-Tier Traversal
        # Tier 1: Coarse
        coarse_voxels = traverse_coarse(rp, filtered_grid, aabb)
        
        # Tier 2: Fine
        fine_voxels = traverse_fine(rp, coarse_voxels, filtered_grid)
        
        # Tier 3: Leaf
        leaf_voxels = traverse_leaf(rp, fine_voxels, filtered_grid)
        
        # Tier 4: Micro
        samples = traverse_micro(rp, leaf_voxels, filtered_grid)
        
        # Process samples with TME
        # TME uses spatial partitioning: each CV has an independent FC network
        colors, densities = tme.evaluate(samples, rp.cv_tag)
        
        # Volume Rendering
        pixel_value, final_transmittance = vr_u.integrate(rp.rays, colors, densities, transmittance)
        
        # Termination Check
        if final_transmittance < threshold or all_rays_exited_aabb(rp.rays, aabb):
            rp.retire()
        else:
            rp.reschedule()
            
        results.append((pixel_value, rp.coordinates))
        
    return results
```

## Subtopics

### Lag-First Traversal
A ray divergence handling technique where lagging rays within an RP are selectively advanced to promote convergence with leading rays, mitigating throughput degradation caused by rays entering distinct voxels [S1]. When rays within an RP diverge into distinct voxels, the scheduler identifies lagging rays and advances them to align with the group [S1]. This technique achieves an average of 3.89 simultaneous rays per RP during coarse traversal and 3.48 during fine traversal [S1].

### Tiny MLP Engine (TME) with Spatial Partitioning
An MLP engine that employs a spatial partitioning strategy similar to KiloNeRF, assigning each Coarse Voxel (CV) an independent fully connected network [S1]. Each Ray Packet (RP) is assigned a unique pointer serving as an address in the global RP buffer and an identifier. The number of RPs under processing is determined by the global RP buffer capacity. An RP is considered fully processed once the transmittance of all remaining samples drops below a predefined threshold or all its rays exit the AABB [S1].

## Operating characteristics
*   **Computational Load:** The AABB module and Gaussian filtering lead to a 70.4% reduction in computational load during coarse traversal [S1].
*   **Parallelism:** Operates on Ray Packets (RPs) encompassing multiple rays to increase parallelism [S1].
*   **Performance Metrics:** The lag-first approach achieves an average of 3.89 simultaneous rays per RP during coarse traversal and 3.48 during fine traversal [S1].
*   **Memory Efficiency:** Reordering RPs based on CV tags alleviates redundant EMAs caused by repeated loading and eviction of identical CV data at boundaries [S1].
*   **Limitations:** Ray divergence within RPs remains a critical challenge, diminishing ray-level parallelism if not handled by the lag-first scheduler [S1].

## Background
The HRM framework was proposed to address the limitations of single-ray processing methods like HESS, which do not exploit the parallelism of ray packets [S1]. It integrates an AABB module with a Gaussian filter to mitigate noise in occupancy grids, thereby constraining occupancy grid bitmap checks to voxels within the AABB [S1]. The framework comprises four tiers: coarse, fine, leaf, and micro traversal [S1]. The TME employs a spatial partitioning strategy similar to KiloNeRF [S1].

## Sources
[S1] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
