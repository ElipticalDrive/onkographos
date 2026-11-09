# LOG-based Silhouette Detection

Also called: Laplacian of Gaussian edge detection

## Overview
LOG-based Silhouette Detection is a volumetric edge detection technique used to identify silhouette voxels for rendering sketch-like curves, replacing a more exhaustive 3x3x3 subspace search to improve performance [S1]. It identifies boundaries based on local volumetric properties (gradient magnitude and Laplacian of Gaussian values) to highlight structures such as bones or internal features [S1]. It sets itself apart from sibling methods like Line Illustration Detection by operating directly on volumetric scalar fields using a precomputed Laplacian of Gaussian operator rather than image-space depth discontinuities or surface normals [S1][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$ where each voxel $i$ has a scalar value $V_i$.
*   **Output:** A set of line segments drawn through the centers of identified silhouette voxels.

**Data Structures:**
*   **Precomputed LOG Volume:** A 3D array of the same dimensions as $V$, storing the Laplacian of Gaussian value for each voxel.
*   **Thresholds:** Three user-controllable scalar values: $T_{hLOG}$, $T_{heye}$, and $T_{hgrad}$.

**Preprocessing Step:**
1.  Compute the LOG value for each voxel in the volume dataset. This involves convolving the volume with a Gaussian kernel and then applying the Laplacian operator (or equivalently, convolving with a Laplacian of Gaussian kernel) [S1].

**Rendering Step:**
For each voxel $i$ in the volume:
1.  Retrieve the voxel value $V_i$, the precomputed LOG value $\text{LOG}(V_i)$, the gradient $\nabla \vec{V_i}$, and the eye vector $\vec{E}$ (view direction) at the voxel center.
2.  Check if the voxel satisfies all three of the following criteria simultaneously:
    *   **Criterion 1:** The product of the voxel value and its LOG value is below a threshold:
        $$ V_i * \text{LOG}(V_i) < T_{hLOG} $$
    *   **Criterion 2:** The dot product of the eye vector and the gradient is below a threshold:
        $$ (\vec{E} \cdot \nabla \vec{V_i}) < T_{heye} $$
    *   **Criterion 3:** The magnitude of the gradient exceeds a threshold:
        $$ \|\nabla \vec{V_i}\| > T_{hgrad} $$
3.  If all three criteria are met, the voxel is classified as a silhouette voxel.
4.  Draw a line segment through the center of the voxel in the direction of $\nabla \vec{V_i} \times \vec{E}$ [S1].

**Parameters:**
*   $T_{hLOG}$: Threshold for the LOG value product. User-controllable [S1].
*   $T_{heye}$: Threshold for the dot product of the eye vector and gradient. User-controllable [S1].
*   $T_{hgrad}$: Threshold for the gradient magnitude. User-controllable [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_silhouette_curves(volume, precomputed_log, eye_vector, 
                             t_hlog, t_hey, t_hgrad):
    """
    Renders silhouette curves for a volume using LOG-based detection.
    
    Args:
        volume: 3D array of voxel values V_i.
        precomputed_log: 3D array of precomputed LOG(V_i) values.
        eye_vector: 3D vector E representing the view direction.
        t_hlog: Threshold for V_i * LOG(V_i).
        t_hey: Threshold for E . grad(V_i).
        t_hgrad: Threshold for |grad(V_i)|.
        
    Returns:
        List of line segments (start_point, end_point, direction).
    """
    segments = []
    dimensions = volume.shape
    
    # Iterate over all voxels
    for x in range(dimensions[0]):
        for y in range(dimensions[1]):
            for z in range(dimensions[2]):
                # Get voxel value and precomputed LOG value
                v_i = volume[x, y, z]
                log_v_i = precomputed_log[x, y, z]
                
                # Compute gradient at voxel (e.g., using central differences)
                # [G] Gradient computation details are not explicitly given in S1,
                # but standard finite differences are implied for volumetric data.
                grad_v = compute_gradient(volume, x, y, z)
                
                # Criterion 1: V_i * LOG(V_i) < T_hLOG
                cond1 = (v_i * log_v_i) < t_hlog
                
                # Criterion 2: (E . grad(V_i)) < T_heye
                dot_product = dot(eye_vector, grad_v)
                cond2 = dot_product < t_hey
                
                # Criterion 3: |grad(V_i)| > T_hgrad
                grad_magnitude = magnitude(grad_v)
                cond3 = grad_magnitude > t_hgrad
                
                # If all criteria are met, draw line segment
                if cond1 and cond2 and cond3:
                    # Direction of line segment: grad(V_i) x E
                    line_direction = cross(grad_v, eye_vector)
                    
                    # Normalize direction for consistent segment length if needed
                    # [G] S1 does not specify segment length; assume unit or fixed length
                    line_direction = normalize(line_direction)
                    
                    # Center of voxel
                    center = [x, y, z]
                    
                    # Add segment to list
                    segments.append((center, line_direction))
                    
    return segments

def compute_gradient(volume, x, y, z):
    """
    Computes the gradient at a voxel using central differences.
    Handles boundaries by clamping or using one-sided differences.
    """
    # [G] Standard central difference implementation
    dx = volume[x+1, y, z] - volume[x-1, y, z] if 0 < x < volume.shape[0]-1 else 0
    dy = volume[x, y+1, z] - volume[x, y-1, z] if 0 < y < volume.shape[1]-1 else 0
    dz = volume[x, y, z+1] - volume[x, y, z-1] if 0 < z < volume.shape[2]-1 else 0
    return [dx, dy, dz]
```

## Subtopics

### Silhouette Detection via Laplacian Filter
A method to detect object silhouettes to stop advection over geometric boundaries, preventing 'particles' from flowing across surface boundaries which would make geometry and topology unrecognizable [S2, book]. It applies a standard three-by-three Laplacian filter convolution mask to depth values, thresholds the result, and stores the silhouette image $e_{x,y}$ in the red color channel of the output texture [S2, book]. The convolution mask is:
$$
\begin{bmatrix} 
0 & 1 & 0 \\ 
1 & -4 & 1 \\ 
0 & 1 & 0 
\end{bmatrix}
$$
This approach is image-space based and distinct from the volumetric LOG method [S2, book].

## Operating characteristics
*   **Performance:** The LOG technique provides virtually identical results to the exhaustive 3x3x3 subspace search but is much faster to calculate per frame because it simplifies boundary computation [S1].
*   **Speed:** Interactive rendering is possible on modern PCs. Frame rates can reach 20-30 frames per second for stipple rendering [S1].
*   **Preprocessing:** Preprocessing time varies from seconds to a minute depending on dataset size [S1].
*   **Output Size:** The total number of point primitives in a typical dataset ranges from 5,000 to 2,000,000, with silhouette curves ranging from 1,000 to 300,000 [S1].
*   **Hardware:** Tested on a dual processor Intel Xeon 2.00 GHz computer with a Geforce 3 Ti 500 display card [S1].
*   **Strengths:** Significantly improves image quality by highlighting structures, especially internal features like bones [S1].
*   **Limitations:** Performance can be improved by further reducing cache exchange and floating point operations [S1].

## Background
The method was introduced in the context of Non-Photorealistic Volume Rendering Using Stippling Techniques [S1]. It was developed to address the performance bottleneck of exhaustive 3x3x3 subspace searches for silhouette detection in volumetric data. The technique allows for the efficient generation of sketch-like curves that enhance the perceptual clarity of volumetric structures.

## Sources
[S1] Non-Photorealistic Volume Rendering Using Stippling Techniques, https://www.csee.umbc.edu/~rheingan/pubs/vis_stipple.pdf, fetched 2026-10-27
[S2] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
