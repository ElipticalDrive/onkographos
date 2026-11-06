# LOG-based Silhouette Detection

Also called: Laplacian of Gaussian edge detection

## Overview
LOG-based Silhouette Detection is a volumetric edge detection technique used to identify silhouette voxels for rendering sketch-like curves in non-photorealistic volume rendering [S1]. It replaces an exhaustive 3x3x3 subspace search with a precomputed Laplacian of Gaussian (LOG) value to improve performance while maintaining virtually identical visual results [S1]. This method simplifies boundary computation, allowing for faster per-frame calculation compared to the exhaustive search approach [S1].

## How it works
The method operates in two phases: a preprocessing step and a rendering step.

**Inputs and Outputs**
*   **Input:** A volumetric dataset $V$, where each voxel has a scalar value $V_i$.
*   **Output:** A set of line segments representing silhouette curves.

**Data Structures**
*   **Precomputed LOG Volume:** A 3D array of the same dimensions as the input volume, storing the LOG value for each voxel. Let this be $L$, where $L_i = \text{LOG}(V_i)$.
*   **Thresholds:** Three user-controllable scalar values: $T_{hLOG}$, $T_{heye}$, and $T_{hgrad}$ [S1].

**Numbered Steps**

1.  **Preprocessing:** Compute the LOG value for each voxel in the volume dataset. This involves applying the Laplacian of Gaussian operator to the input volume $V$ to generate the precomputed LOG volume $L$ [S1].
2.  **Rendering Loop:** For each voxel $i$ in the volume:
    a.  Retrieve the voxel value $V_i$, the precomputed LOG value $L_i$, and the gradient vector $\nabla \vec{V_i}$ [S1].
    b.  Determine if the voxel is a silhouette voxel by checking three criteria simultaneously [S1]:
        i.  The product of the voxel value and its LOG value is below a threshold: $V_i \cdot L_i < T_{hLOG}$ [S1].
        ii. The dot product of the eye vector $\vec{E}$ and the gradient $\nabla \vec{V_i}$ is below a threshold: $(\vec{E} \cdot \nabla \vec{V_i}) < T_{heye}$ [S1].
        iii. The magnitude of the gradient exceeds a threshold: $\|\nabla \vec{V_i}\| > T_{hgrad}$ [S1].
    c.  If a voxel satisfies all three criteria, draw a line segment through the center of the voxel in the direction of $\nabla \vec{V_i} \times \vec{E}$ [S1].

**Formulas**
*   **Silhouette Criterion 1:** $V_i \cdot \text{LOG}(V_i) < T_{hLOG}$ [S1].
    *   $V_i$: Voxel value.
    *   $\text{LOG}(V_i)$: Precomputed Laplacian of Gaussian value.
    *   $T_{hLOG}$: User-controllable threshold for the LOG value product [S1].
*   **Silhouette Criterion 2:** $(\vec{E} \cdot \nabla \vec{V_i}) < T_{heye}$ [S1].
    *   $\vec{E}$: Eye (or view) vector [S1].
    *   $\nabla \vec{V_i}$: Gradient at the voxel [S1].
    *   $T_{heye}$: User-controllable threshold for the dot product of the eye vector and gradient [S1].
*   **Silhouette Criterion 3:** $\|\nabla \vec{V_i}\| > T_{hgrad}$ [S1].
    *   $T_{hgrad}$: User-controllable threshold for the gradient magnitude [S1].
*   **Line Direction:** $\nabla \vec{V_i} \times \vec{E}$ [S1].

**Parameters**
*   $T_{hLOG}$: Threshold for the LOG value product, user-controllable [S1].
*   $T_{heye}$: Threshold for the dot product of the eye vector and gradient, user-controllable [S1].
*   $T_{hgrad}$: Threshold for the gradient magnitude, user-controllable [S1].

**Edge Cases and Pitfalls**
*   The method relies on the precomputed LOG volume. If the volume data changes, the LOG volume must be recomputed [S1].
*   The performance of the rendering phase depends on the number of voxels satisfying the criteria. In typical datasets, the number of silhouette curves ranges from 1,000 to 300,000 [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(volume):
    """
    Compute the LOG value for each voxel in the volume dataset.
    volume: 3D array of voxel values.
    Returns: 3D array of LOG values.
    """
    # Implementation of Laplacian of Gaussian operator
    # This step is performed once per volume dataset
    log_volume = apply_laplacian_of_gaussian(volume)
    return log_volume

def render_silhouettes(volume, log_volume, eye_vector, t_hlog, t_hey e, t_hgrad):
    """
    Determine silhouette voxels and generate line segments.
    volume: 3D array of voxel values.
    log_volume: 3D array of precomputed LOG values.
    eye_vector: 3D vector representing the view direction.
    t_hlog, t_hey e, t_hgrad: User-controllable thresholds.
    Returns: List of line segments.
    """
    line_segments = []
    dimensions = volume.shape
    
    for i in range(dimensions[0]):
        for j in range(dimensions[1]):
            for k in range(dimensions[2]):
                voxel_value = volume[i, j, k]
                log_value = log_volume[i, j, k]
                
                # Compute gradient at voxel (i, j, k)
                # Using central differences or similar method
                gradient = compute_gradient(volume, i, j, k)
                
                # Criterion 1: V_i * LOG(V_i) < T_hLOG
                criterion_1 = voxel_value * log_value < t_hlog
                
                # Criterion 2: (E . grad(V_i)) < T_hey e
                criterion_2 = dot_product(eye_vector, gradient) < t_hey e
                
                # Criterion 3: ||grad(V_i)|| > T_hgrad
                criterion_3 = magnitude(gradient) > t_hgrad
                
                if criterion_1 and criterion_2 and criterion_3:
                    # Draw line segment through center of voxel
                    # Direction: grad(V_i) x E
                    direction = cross_product(gradient, eye_vector)
                    center = (i, j, k)
                    line_segments.append(LineSegment(center, direction))
                    
    return line_segments

def compute_gradient(volume, i, j, k):
    """
    Compute the gradient vector at voxel (i, j, k).
    """
    # Example using central differences
    # Handle boundaries appropriately
    dx = volume[i+1, j, k] - volume[i-1, j, k]
    dy = volume[i, j+1, k] - volume[i, j-1, k]
    dz = volume[i, j, k+1] - volume[i, j, k-1]
    return Vector3(dx, dy, dz)

def apply_laplacian_of_gaussian(volume):
    """
    Apply Laplacian of Gaussian operator to the volume.
    """
    # This is a placeholder for the actual LOG computation
    # The source states this is done in preprocessing
    # Implementation details of LOG are not provided in the source
    # [G] Standard LOG involves convolving with Gaussian derivative kernels
    log_volume = convolve_with_log_kernel(volume)
    return log_volume

def convolve_with_log_kernel(volume):
    """
    Placeholder for LOG convolution.
    """
    # [G] In practice, this would involve separable filtering
    # with Gaussian smoothing followed by Laplacian computation
    return volume # Placeholder
```

## Operating characteristics
*   **Time Complexity:** The preprocessing step requires computing the LOG value for each voxel, which takes time proportional to the number of voxels. The rendering step iterates over each voxel, performing constant-time checks and potentially generating a line segment. The total time is proportional to the number of voxels plus the number of silhouette curves generated [S1].
*   **Memory Complexity:** Requires storing the precomputed LOG volume, which is the same size as the input volume [S1].
*   **Measured Performance:** On a dual processor Intel Xeon 2.00 GHz computer with a Geforce 3 Ti 500 display card, the preprocessing time varies from seconds to a minute. Frame rates can reach 20 to 30 frames per second for stipple rendering [S1]. The total number of point primitives in a typical dataset ranges from 5,000 to 2,000,000, and the silhouette curves range from 1,000 to 300,000 [S1].
*   **Strengths:** Provides virtually identical results to the exhaustive 3x3x3 subspace search but is much faster to calculate per frame because it simplifies boundary computation [S1].
*   **Limitations:** Performance can be improved by further reducing cache exchange and floating point operations [S1]. The method is specific to volumetric data and requires preprocessing.

## Background
LOG-based Silhouette Detection was introduced in the context of non-photorealistic volume rendering using stippling techniques [S1]. It was developed as an alternative to an exhaustive 3x3x3 subspace search for identifying silhouette voxels, aiming to improve performance while maintaining visual quality [S1]. The technique is used to highlight structures in volume datasets, such as the bones of a foot dataset [S1].

## Sources
[S1] Non-Photorealistic Volume Rendering Using Stippling Techniques, https://www.csee.umbc.edu/~rheingan/pubs/vis_stipple.pdf, fetched 2026-10-27
