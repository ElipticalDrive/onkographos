# Distance-based Transfer Function

Also called: Distance-based TF, dbTF

## Overview
The distance-based transfer function (dbTF) is a feature-based visualization mapping that uses the distance to a previously segmented structure as an additional metric for color and opacity mapping, alongside the primary data value [S1]. It solves the problem of highlighting anatomical or structural neighborhoods by coloring features in the vicinity of a reference object differently from those further away [S1]. Unlike sibling methods that derive metrics from local data properties (such as texture or size), dbTF relies on global spatial relationships to a user-defined or pre-segmented target [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric dataset $V$, segmentation volume $S$ (binary or label map indicating the reference structure), and a 2D transfer function $TF(v, d)$ mapping data value $v$ and distance $d$ to color and opacity [S1].
*   **Output:** Rendered image with enhanced neighborhood visualization [S1].

**Data Structures:**
1.  **Segmentation Volume ($S$):** A volume where voxels belonging to the reference structure are marked (e.g., 1) and others are 0 [S1].
2.  **Distance Volume ($D$):** A volume storing the Euclidean distance from each voxel to the nearest surface of the segmented structure. This volume may be stored at a lower resolution than the input dataset $V$ [S1].
3.  **Transfer Function ($TF$):** A 2D look-up table indexed by data value and distance [S1].

**Preprocessing:**
1.  Compute the distance volume $D$ from the segmentation volume $S$. For complex surfaces, this typically involves a signed distance field computation [S1].
2.  [G] If the reference structure is a simple geometric primitive (point, sphere, plane), the distance can be computed analytically during rendering without precomputing $D$ [S1].

**Rendering Steps:**
1.  For each ray sample point $p$ in the volume:
    a.  Retrieve the data value $v = V(p)$ [S1].
    b.  Retrieve the distance $d = D(p)$. If $D$ is lower resolution, perform trilinear interpolation to estimate $d$ at $p$ [S1].
    c.  Sample the 2D transfer function: $(c, \alpha) = TF(v, d)$, where $c$ is color and $\alpha$ is opacity [S1].
    d.  Composite the sample into the final pixel color using standard volume rendering integration [S1].

**Parameters:**
*   **Segmentation:** The definition of the reference structure $S$.
*   **Distance Resolution:** The resolution of $D$ relative to $V$. Lower resolution is acceptable due to interpolation [S1].

**Edge Cases and Pitfalls:**
*   **Time-Dependent Data:** Requires segmentation, making it unfeasible for time-dependent data unless segmentation is also dynamic [S1].
*   **Precision:** Trilinear interpolation of the distance volume causes small inaccuracies near surfaces if the distance volume resolution is significantly lower than the dataset [S1].
*   **Preprocessing Time:** Computing the distance volume can be slow (e.g., ~20 seconds for a $256^3$ volume) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_dbtf(volume, segmentation, tf_2d, ray_origin, ray_direction, bounds):
    """
    Renders a volume using a distance-based transfer function.
    volume: 3D array of data values.
    segmentation: 3D array of binary labels for the reference structure.
    tf_2d: 2D array mapping (data_value, distance) to (color, opacity).
    ray_origin, ray_direction: Ray parameters.
    bounds: Bounding box of the volume.
    """
    # Precompute distance volume if not done externally
    # distance_volume = compute_distance_field(segmentation)
    # For simplicity, assume distance_volume is provided or computed on-the-fly for simple shapes
    
    color = (0.0, 0.0, 0.0)
    alpha = 0.0
    
    # Ray marching loop
    t = 0.0
    while t < bounds.max_dist:
        p = ray_origin + t * ray_direction
        if not in_bounds(p, bounds):
            break
            
        # Sample data value
        v = sample_volume(volume, p)
        
        # Sample distance
        # If segmentation is a simple shape (e.g., sphere at center C with radius R):
        # d = max(0, distance(p, C) - R)
        # Else, interpolate from precomputed distance volume
        d = sample_distance_field(segmentation, p)
        
        # Map to color and opacity
        c, a = tf_2d[v, d]
        
        # Alpha blending
        color = color * (1.0 - a) + c * a * (1.0 - alpha)
        alpha += a * (1.0 - alpha)
        
        if alpha > 0.99:
            break
            
        t += step_size
        
    return color
```

## Subtopics

### Distance-Based Enhancement
A technique proposed by Rieder et al. for functional data display and lesion detection, as well as for path to target exploration, designed individual TFs for each imaging modality [S2]. It applies the distance-based metric specifically to enhance the visibility of functional data or lesions in neurosurgical contexts [S2].

## Operating characteristics
*   **Time Complexity:** Real-time capable during rendering, requiring 2 3D texture lookups per sample (one for data, one for distance) [S1]. Preprocessing time depends on segmentation and distance field computation (e.g., 20 seconds for $256^3$) [S1].
*   **Memory Consumption:** Requires 1 additional volume during rendering (the distance volume), which does not need to be full resolution [S1].
*   **Strengths:** Effective for highlighting neighborhoods of specific structures; flexible user interaction via TF design [S1].
*   **Limitations:** Requires prior segmentation, making it unsuitable for automatic or time-dependent data without dynamic segmentation [S1].
*   **When to Use:** When the user wants to emphasize features adjacent to a known anatomical or structural boundary [S1].
*   **When Not to Use:** For time-dependent data or when segmentation is not feasible or reliable [S1].

## Background
The distance-based transfer function was introduced by Tzormatzoglou et al. in 2006 [S1]. It is part of a family of feature-based transfer functions that extend 1D data-based TFs by incorporating additional metrics to improve tissue differentiation [S1]. It is primarily used in medical visualization to highlight pathological regions or anatomical interfaces [S1, S2].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
