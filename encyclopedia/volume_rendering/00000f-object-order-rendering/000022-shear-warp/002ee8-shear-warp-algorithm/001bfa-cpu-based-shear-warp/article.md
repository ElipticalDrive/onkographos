# CPU-based Shear-Warp

Also called: CPU Shear-Warp

## Overview
CPU-based Shear-Warp is a CPU implementation of the Shear-Warp algorithm that accelerates volume rendering by using Run-Length Encoding (RLE) and Hounsfield scale classification to skip empty voxels [S1, S2, unofficial]. It differs from its siblings by relying on 1D run-lists per axis rather than 3D spatial structures like octrees [S1, S2, unofficial].

## How it works
**Inputs:** Volume data (voxel values), transfer function parameters (lower/upper bounds).
**Outputs:** Rendered image.

**Data Structures:**
Three sets of lists of runs, one for each coordinate axis (X, Y, Z). Each set corresponds to a principal view direction. Each list in a set corresponds to one scan line of one slice. A run is defined by a start index, length, and type (transparent or opaque) [S1, S2, unofficial].

**Steps:**
1. **Classification:** Convert input voxel values to opacities using the Hounsfield scale and a linear transfer function with lower and upper bounds [S1, S2, unofficial].
2. **Compression:** For each coordinate axis and each slice, compress each scan line of voxels into a list of runs using RLE [S1, S2, unofficial].
   - Two types of runs: completely transparent (empty voxels) and not completely transparent [S1, S2, unofficial].
3. **Rendering:** Traverse the data structures slice by slice and line by line [S1, S2, unofficial].
4. **Skipping:** Ignore completely transparent runs to skip empty voxels during rendering [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(volume_data, lower_bound, upper_bound):
    """
    Preprocess volume data into RLE run-lists for each axis.
    volume_data: 3D array of voxel values.
    lower_bound, upper_bound: floats defining the linear transfer function.
    Returns: dict with keys 'x', 'y', 'z', each mapping to a list of lists of runs.
    """
    axes = ['x', 'y', 'z']
    run_lists = {axis: [] for axis in axes}
    
    # Determine dimensions
    D, H, W = volume_data.shape
    
    for axis in axes:
        axis_runs = []
        # Iterate over slices perpendicular to the axis
        if axis == 'x':
            # Iterate over y, z slices
            for y in range(H):
                for z in range(W):
                    scan_line = volume_data[:, y, z]
                    axis_runs.append(compress_scan_line(scan_line, lower_bound, upper_bound))
        elif axis == 'y':
            # Iterate over x, z slices
            for x in range(D):
                for z in range(W):
                    scan_line = volume_data[x, :, z]
                    axis_runs.append(compress_scan_line(scan_line, lower_bound, upper_bound))
        elif axis == 'z':
            # Iterate over x, y slices
            for x in range(D):
                for y in range(H):
                    scan_line = volume_data[x, y, :]
                    axis_runs.append(compress_scan_line(scan_line, lower_bound, upper_bound))
        run_lists[axis] = axis_runs
    return run_lists

def compress_scan_line(scan_line, lower_bound, upper_bound):
    """
    Compress a 1D scan line into RLE runs based on opacity.
    scan_line: 1D array of voxel values.
    lower_bound, upper_bound: floats.
    Returns: list of tuples (start_index, length, is_transparent).
    """
    runs = []
    if not scan_line:
        return runs
        
    # Calculate opacity for each voxel
    # Linear transfer function: opacity = 0 if val < lower, 1 if val > upper, linear in between
    opacities = []
    for val in scan_line:
        if val < lower_bound:
            opacities.append(0.0)
        elif val > upper_bound:
            opacities.append(1.0)
        else:
            opacities.append((val - lower_bound) / (upper_bound - lower_bound))
            
    # Compress into runs
    current_run_start = 0
    current_is_transparent = (opacities[0] == 0.0)
    current_run_length = 1
    
    for i in range(1, len(opacities)):
        is_transparent = (opacities[i] == 0.0)
        if is_transparent == current_is_transparent:
            current_run_length += 1
        else:
            runs.append((current_run_start, current_run_length, current_is_transparent))
            current_run_start = i
            current_is_transparent = is_transparent
            current_run_length = 1
            
    # Append the last run
    runs.append((current_run_start, current_run_length, current_is_transparent))
    return runs

def render(run_lists, image_width, image_height):
    """
    Render using the preprocessed run-lists.
    run_lists: dict from preprocess_volume.
    image_width, image_height: int.
    Returns: rendered image array.
    """
    image = np.zeros((image_height, image_width, 4)) # RGBA
    
    # Note: The actual shear-warp projection and blending steps are defined in the parent article [Parent].
    # This function illustrates the traversal and skipping logic specific to the CPU-based RLE optimization.
    
    for axis in ['x', 'y', 'z']:
        axis_runs = run_lists[axis]
        # Iterate through slices and scan lines
        # The exact iteration order and mapping to screen coordinates depends on the view direction
        # and the shear-warp projection defined in the parent article.
        # For each scan line, iterate through runs.
        for run_list in axis_runs:
            for start, length, is_transparent in run_list:
                if is_transparent:
                    # Skip empty voxels
                    continue
                else:
                    # Process opaque voxels in this run
                    # Apply shear, project, and blend into the image
                    # This part relies on the parent algorithm's projection logic
                    pass
                    
    return image
```

**Edge Cases and Pitfalls:**
- The transfer function bounds must be chosen appropriately for the data range to avoid skipping relevant structures or including noise [S1, S2, unofficial].
- Precision issues may arise if the linear interpolation in the transfer function is not handled carefully for values near the bounds [G].

## Subtopics

### [Parallel Perspective Shear-Warp Rendering](802-parallel-perspective-shear-warp-rendering.md)
Extends the method for parallel execution.

### [Permutation Proof for Projection and Warp](1702-permutation-proof-for-projection-and-warp.md)
Provides theoretical foundation for the projection and warp steps.

### [Warped Isosurface Extraction (WISE)](1906-warped-isosurface-extraction-wise.md)
Applies the warping concept to isosurface extraction.

## Operating characteristics
**Time and Memory Complexity:**
- Preprocessing: $O(N)$ where $N$ is the number of voxels, due to classification and RLE compression [S1, S2, unofficial].
- Rendering: Proportional to the number of non-transparent runs, which is typically much less than $N$ for sparse data [S1, S2, unofficial].
- Memory: Storage for three sets of run-lists. The size depends on the sparsity of the data [S1, S2, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Performance scales with the sparsity of the volume data. More empty voxels lead to greater speedups due to skipping [S1, S2, unofficial].

**Strengths and Limitations:**
- Strengths: Significant speedup for sparse data by skipping empty voxels [S1, S2, unofficial]. Simple data structures (1D lists) [S1, S2, unofficial].
- Limitations: Does not use 3D spatial structures like octrees, which might offer better compression for certain data types [S1, S2, unofficial]. Performance may degrade for dense data where most voxels are opaque [G].

**When to Use:**
- Use when rendering sparse volume data on a CPU and interactive performance is required [S1, S2, unofficial].

**When Not to Use:**
- Do not use for dense volume data where the overhead of managing run-lists outweighs the benefits of skipping [G].
- Do not use if 3D spatial indexing (e.g., octrees) is more appropriate for the data structure [S1, S2, unofficial].

## Background
The CPU-based Shear-Warp algorithm was implemented as part of research into GPU-based volume rendering, aiming to first establish a baseline CPU implementation [S1, S2, unofficial]. It builds on the original Shear-Warp algorithm [S1, S2, unofficial]. The use of Hounsfield scale classification is specific to medical imagery applications [S1, S2, unofficial].

## Sources
[S1] GPU-Based Volume Rendering for Medical Imagery, https://inria.hal.science/inria-00461484v1, fetched 2026-10-26 (unofficial)
[S2] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
