# Fast Voxelization for Microgeometry

Also called: Voxelization of Microgeometry

## Overview
Fast Voxelization for Microgeometry is a GPU-accelerated method designed to voxelize sparse, high-frequency microgeometry (such as fibers, hair, and fabrics) while preserving directional orientation data across multiple levels of detail (LoD) [S1]. Unlike traditional raster-based approaches like OpenVDB that prioritize storage density over processing speed, this method maximizes voxelization throughput by distributing workload between CPU and GPU, enabling the processing of highly detailed scenes that would otherwise exceed memory limits [S1]. It specifically addresses the limitation of simple averaging in prior works (e.g., Lopez-Moreno et al.'s Lumislice adaptation) by clustering and preserving orientation distributions, ensuring accurate anisotropic scattering simulation at coarser resolutions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse microgeometry data, which may include splines, textured orientation data, or triangle meshes representing fibers/hair [S1].
*   **Output:** A hierarchical volumetric data model (SGGX-H) containing density and orientation distributions at multiple resolution levels [S1].

**Data Structures:**
The method utilizes a hierarchical structure based on Symmetrical GGX (SGGX) distributions [S1].
*   **SGGX-H Model:** Stores not just occupancy and optical density, but also fitted directional distributions (normals/tangents) within each voxel [S1].
*   **Resolution Levels:** The output is generated at two distinct resolution levels controlled by user parameters: `first level resolution` and `second level resolution` [S1].

**Parameters:**
1.  **First level resolution:** Controls the base resolution of the voxelization [S1].
2.  **Second level resolution:** Controls the finer resolution of the voxelization [S1].
3.  **Sample number:** A user-controllable parameter determining the sampling density during the voxelization process [S1].

**Algorithm Steps:**
1.  **Workload Distribution:** The system distributes the voxelization workload strategically between CPU and GPU cores to maximize speed and memory efficiency [S1].
2.  **Primitive Processing:** Input primitives (splines, meshes) are processed. For each primitive, samples are generated [S1].
3.  **Orientation Preservation:** Unlike methods that generate samples directly at the finer level and apply simple averaging, this method preserves the orientation distributions across multiple detail levels [S1]. This involves clustering directional data to maintain the characteristic anisotropic appearance of complex materials [S1].
4.  **Hierarchical Generation:** The method fits distributions to generate lower-resolution volumes suitable for LoD rendering [S1]. The LoD level selection can follow standard Path Tracer MipMap selection based on projected width on the camera [S1].
5.  **Memory Management:** The implementation ensures that the voxelization fits within GPU memory limits by balancing the `sample number` and resolution parameters [S1].

**Pitfalls and Edge Cases:**
*   **Memory Limits:** Combining an elevated number of samples with high resolution can lead to out-of-memory errors [S1]. Users must balance these parameters based on available GPU memory [S1].
*   **Flat Geometry:** For models where one dimension is almost flat compared to the largest dimension, regular cubic voxel shapes require a significant increase in global resolution to ensure the smaller dimension has enough nodes for proper voxelization [S1].
*   **Parameter Heuristics:** There is no automatic heuristic provided for selecting suitable combinations of input parameters for a given model; users must manually adjust resolutions and sample counts [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Fast Voxelization for Microgeometry
# Note: Specific CUDA kernel implementations and SGGX fitting algorithms are not detailed in the source text.
# The following represents the logical flow described in [S1].

def fast_voxelization_microgeometry(input_primitives, first_level_res, second_level_res, sample_count):
    """
    Voxelizes sparse microgeometry using a GPU-CPU hybrid approach.
    
    Args:
        input_primitives: List of splines, meshes, or textured orientation data.
        first_level_res: Resolution for the coarser level.
        second_level_res: Resolution for the finer level.
        sample_count: Number of samples per primitive.
        
    Returns:
        SGGX_Hierarchical_Data: Volumetric data with density and orientation distributions.
    """
    
    # 1. Validate parameters to prevent out-of-memory errors [S1]
    if sample_count is too_high_for_memory(first_level_res, second_level_res):
        raise MemoryError("Combination of samples and resolution exceeds GPU memory.")

    # 2. Distribute workload between CPU and GPU [S1]
    # The source mentions strategic workload distribution but does not specify the exact split logic.
    cpu_tasks = []
    gpu_tasks = []
    for primitive in input_primitives:
        # Assign tasks based on primitive type and size
        if primitive.type == "spline":
            gpu_tasks.append(primitive)
        else:
            cpu_tasks.append(primitive)
            
    # 3. Process primitives and preserve orientation distributions [S1]
    # Unlike Lopez-Moreno et al., we do not use simple averaging [S1]
    voxel_data_coarse = []
    voxel_data_fine = []
    
    # Process GPU tasks (CUDA implementation implied) [S1]
    for primitive in gpu_tasks:
        # Generate samples based on sample_count [S1]
        samples = generate_samples(primitive, sample_count)
        
        # Cluster and preserve orientation distributions [S1]
        # This step fits SGGX distributions to the samples [S1]
        coarse_voxels = fit_sggx_distributions(samples, first_level_res)
        fine_voxels = fit_sggx_distributions(samples, second_level_res)
        
        voxel_data_coarse.extend(coarse_voxels)
        voxel_data_fine.extend(fine_voxels)
        
    # Process CPU tasks
    for primitive in cpu_tasks:
        # [G] Standard CPU voxelization logic would go here, 
        # but the source focuses on the GPU optimization.
        pass

    # 4. Construct Hierarchical SGGX-H Model [S1]
    hierarchical_model = SGGX_Hierarchical_Data(
        coarse_level=voxel_data_coarse,
        fine_level=voxel_data_fine
    )
    
    return hierarchical_model

def generate_samples(primitive, count):
    """
    Generates samples from a primitive (spline, mesh, etc.).
    Implementation details not provided in source [S1].
    """
    # [G] Placeholder for specific sampling strategy (e.g., uniform, stratified)
    # The source mentions handling splines and textured orientation data [S1]
    return []

def fit_sggx_distributions(samples, resolution):
    """
    Fits Symmetrical GGX distributions to samples for a given resolution.
    Implementation details not provided in source [S1].
    """
    # [G] Placeholder for SGGX fitting algorithm
    # The source states this preserves orientation data unlike simple averaging [S1]
    return []
```

## Operating characteristics
*   **Time Complexity:** Optimized for speed via GPU parallelization. The method is designed to maximize voxelization time efficiency compared to CPU-centric frameworks like OpenVDB [S1].
*   **Memory Complexity:** Memory usage is a critical constraint. The method is memory-efficient for sparse data, but combining high resolution with a high sample count can lead to out-of-memory problems [S1].
*   **Performance:** Validated on a CUDA-based implementation for triangle meshes and other primitives [S1]. It outperforms traditional raster-based methods in processing speed for microgeometry [S1].
*   **Strengths:**
    *   Handles sparse microgeometry (fibers, hair, vegetation, fabrics) efficiently [S1].
    *   Preserves orientation distributions across LoD levels, enabling accurate anisotropic scattering [S1].
    *   GPU-optimized for speed, unlike OpenVDB/NanoVDB which prioritize storage density [S1].
*   **Limitations:**
    *   Requires manual tuning of resolution and sample parameters to avoid memory issues [S1].
    *   Regular cubic voxels are inefficient for flat geometries, requiring excessive global resolution [S1].
    *   Limited per-voxel accuracy with complex geometries at lower resolutions, potentially requiring future improvements (e.g., neural methods) [S1].
*   **When to use:** Ideal for rendering sparse, high-frequency microgeometry where anisotropic appearance is critical and GPU memory allows for the required resolution/sample trade-off [S1].
*   **When not to use:** Not suitable for dense, non-sparse geometry where standard voxelization suffices, or for flat geometries where cubic voxels would require prohibitive resolution increases [S1].

## Background
The method was introduced in the paper "Fast Voxelization and Level of Detail for Microgeometry Rendering" [S1]. It addresses the prevalence of sparse microgeometry materials in real-world applications (cloth, hair, vegetation) where traditional voxelization methods fall short [S1]. Prior approaches, such as Lopez-Moreno et al.'s adaptation of Lumislice, failed to preserve orientation data at coarser levels due to simple averaging [S1]. This work builds on Symmetrical GGX (SGGX) distributions [18] to create the SGGX-H hierarchical model [S1]. It contrasts with industry standards like OpenVDB [19, 20] and NanoVDB [21, 22], which focus on storage density and CPU parallelization rather than GPU-optimized voxelization speed for specific primitives [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
