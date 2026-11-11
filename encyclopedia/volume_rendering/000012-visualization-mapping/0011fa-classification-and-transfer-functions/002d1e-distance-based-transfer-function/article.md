# Distance-based Transfer Function

Also called: Distance-based TF, dbTF

## Overview
The Distance-based Transfer Function (dbTF) is a classification method that maps voxel optical properties using the raw data value and the Euclidean distance to a previously segmented reference structure [S1]. It solves the problem of highlighting anatomical or structural neighborhoods by distinguishing features based on their proximity to a specific landmark, rather than their intrinsic intensity or texture [S1]. Unlike sibling methods that rely on local derivatives or texture statistics, dbTF introduces a global spatial context relative to a previously segmented object, enabling the visualization of peritumoral regions or surgical paths [S1, S2].

## How it works
**Inputs:**
1.  Volumetric dataset $V$ of resolution $R_x \times R_y \times R_z$ [S1].
2.  Segmentation volume $S$ of resolution $R_x \times R_y \times R_z$ (or lower), where voxels belonging to the reference structure are marked (e.g., value 1) and background is unmarked (e.g., value 0) [S1].

**Outputs:**
*   Opacity $\alpha \in [0, 1]$ and color $\mathbf{c}$ for each sample point during raycasting [S1].

**Data Structures:**
*   **Distance Volume ($D$):** A 3D grid storing the Euclidean distance from each voxel center to the nearest surface of the segmented structure $S$ [S1].
    *   *Layout:* If $S$ is binary, $D_{i,j,k}$ stores $\min_{p \in \text{Surface}(S)} \| \mathbf{v}_{i,j,k} - \mathbf{p} \|$ [S1].
    *   *Resolution:* $D$ may have lower resolution than $V$ because distances are trilinearly interpolated during rendering, causing only small inaccuracies near surfaces [S1].
*   **Transfer Function ($TF$):** A 2D lookup table (LUT) mapping the pair $(V_i, D_i)$ to $(\alpha, \mathbf{c})$ [S1].

**Preprocessing:**
1.  **Segmentation:** Obtain the binary segmentation volume $S$ representing the reference structure (e.g., a tumor, vessel, or landmark) [S1].
2.  **Distance Transform:** Compute the Euclidean Distance Transform (EDT) of $S$ to generate $D$ [G].
    *   *Note:* For simple geometric references (points, spheres), $D$ can be computed analytically at render time without pre-computation [S1].
    *   *Performance:* Preprocessing time is approximately 20 seconds for a $256^3$ volume using CPU-based EDT [S1]. GPU acceleration or heuristics can reduce this significantly [S1].

**Rendering Steps:**
1.  For each ray sample point $\mathbf{p}$ with data value $v = V(\mathbf{p})$ [S1].
2.  Retrieve the distance $d = D(\mathbf{p})$ from the distance volume $D$ via trilinear interpolation [S1].
3.  Query the 2D Transfer Function: $(\alpha, \mathbf{c}) = TF(v, d)$ [S1].
4.  Composite $\alpha$ and $\mathbf{c}$ into the ray accumulator using front-to-back blending [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_interpolate(volume, x, y, z):
    """Standard trilinear interpolation [G]."""
    nx, ny, nz = volume.shape
    x = max(0.0, min(float(nx) - 1.0, x))
    y = max(0.0, min(float(ny) - 1.0, y))
    z = max(0.0, min(float(nz) - 1.0, z))
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    def v(i, j, k): return volume[min(i, nx-1)][min(j, ny-1)][min(k, nz-1)]
    c00 = v(ix, iy, iz) * (1 - fx) + v(ix+1, iy, iz) * fx
    c01 = v(ix, iy, iz+1) * (1 - fx) + v(ix+1, iy, iz+1) * fx
    c10 = v(ix, iy+1, iz) * (1 - fx) + v(ix+1, iy+1, iz) * fx
    c11 = v(ix, iy+1, iz+1) * (1 - fx) + v(ix+1, iy+1, iz+1) * fx
    return (c00 * (1 - fy) + c01 * fy) * (1 - fz) + (c10 * (1 - fy) + c11 * fy) * fz

def compute_distance_volume(segmentation_volume):
    """Computes Euclidean Distance Transform for binary segmentation [G].
    For simple shapes (points, spheres), returns analytical distance [S1].
    Time: ~20s for 256^3 on CPU [S1]."""
    # Standard EDT algorithm (e.g., Meijster et al.)
    pass

def raycast_dbtf(dataset, distance_volume, tf_lut, ray_origin, ray_direction, t_min, t_max, step_size):
    """Renders a single ray using the distance-based transfer function.
    tf_lut: 2D array mapping (data_value, distance) -> (alpha, color) [S1]."""
    color_accum = (0.0, 0.0, 0.0)
    alpha_accum = 0.0
    
    t = t_min
    while t <= t_max:
        point = ray_origin + t * ray_direction
        data_val = trilinear_interpolate(dataset, point[0], point[1], point[2])
        dist_val = trilinear_interpolate(distance_volume, point[0], point[1], point[2])
        
        alpha, color = tf_lut[data_val, dist_val]
        
        alpha_weighted = alpha * (1.0 - alpha_accum)
        color_accum = tuple(c + col * alpha_weighted for c, col in zip(color_accum, color))
        alpha_accum += alpha_weighted
        
        if alpha_accum >= 1.0:
            break
        t += step_size
        
    return color_accum
```

**Parameters:**
*   **Segmentation Accuracy:** Critical. Errors in $S$ propagate directly to $D$ [S1].
*   **Distance Volume Resolution:** Can be lower than $V$ to save memory, as interpolation smooths minor discretization errors [S1].

**Edge Cases and Pitfalls:**
*   **Time-Dependent Data:** Requires re-segmentation and distance recomputation for every frame, making it unfeasible for real-time animation of dynamic data [S1].
*   **Segmentation Dependency:** Automatic segmentation is not always applicable, and manual segmentation increases user burden compared to 1D TFs [S1].
*   **Resolution Trade-off:** Lowering the distance volume resolution saves memory but may reduce precision near the segmented surface, though trilinear interpolation keeps inaccuracies small [S1].

## Subtopics

### Distance-Based Enhancement
A technique proposed by Rieder et al. for functional data display and lesion detection, as well as for path-to-target exploration [S2]. It involves designing individual transfer functions for each imaging modality to enhance the visibility of specific features relative to a distance metric [S2].

## Operating characteristics
*   **Time Complexity:** Rendering is real-time capable, requiring only 2 3D texture lookups per sample (one for data, one for distance) [S1]. Preprocessing takes approximately 20 seconds for a $256^3$ volume on CPU [S1].
*   **Memory Consumption:** Requires 1 additional volume for distances during rendering [S1]. This volume does not need to be full resolution [S1].
*   **Strengths:** Excellent for visualizing neighborhoods of specific structures (e.g., tumor margins, vessel walls) [S1]. Intuitive for surgical planning and path exploration [S2].
*   **Limitations:** Dependent on the quality of the initial segmentation [S1]. Not suitable for fully automatic or time-varying datasets without significant optimization [S1].
*   **When to use:** When the clinical or scientific question concerns the relationship between a structure and its immediate surroundings (e.g., edema around a tumor) [S1].
*   **When not to use:** For time-series data where segmentation cannot be updated in real-time, or when no clear reference structure exists [S1].

## Background
The distance-based transfer function was introduced by Tschirhart et al. [TPD06] and surveyed in [S1]. It emerged from the need to differentiate tissue types in medical imaging where intensity alone was insufficient, leveraging spatial context provided by segmentation [S1]. It is primarily used in medical visualization, particularly for neurosurgical tumor treatment planning and coronary artery disease diagnosis, where highlighting the interface between healthy and pathological tissue is critical [S1, S2].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
