# Point Light Shading

Also called: Simple Point Light Shading

## Overview
Point Light Shading is a local illumination model for volume rendering that computes shading intensity at a surface intersection point using a single point light source [S1, unofficial]. It determines illumination strength via the dot product of the normalized light direction and the surface normal (gradient) [S1, unofficial]. Unlike sibling methods that may use complex multi-light setups or Phong models, this approach relies on a single directional vector and a constant attenuation assumption, prioritizing computational simplicity over physical accuracy [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `pos`: The 3D position of the surface intersection point (or sampled voxel) [S1, unofficial][S2].
    *   `norm`: The surface normal (gradient) at `pos` [S1, unofficial].
    *   `lightPosition`: The 3D position of the point light source [S1, unofficial][S2].
    *   `mu`: The constant attenuation coefficient [S2].
*   **Outputs:**
    *   `S`: The scalar illumination value applied to the final image [S1, unofficial][S2].

**Data Structures:**
*   Positions and normals are represented as 3D vectors (or `float4` with unused components in some implementations) [S1, unofficial].
*   The light position is a fixed 3D vector [S1, unofficial].

**Steps:**
1.  **Define Light Source:** The user specifies the position of the point light source, denoted as $\mathbf{L}_{pos}$ [S1, unofficial][S2].
2.  **Determine Sample Point:** Identify the position $\mathbf{P}$ of the point on the ray being shaded (the intersection point or sampled voxel center) [S1, unofficial][S2].
3.  **Compute Light Direction:** Calculate the vector from the light source to the point: $\mathbf{d} = \mathbf{P} - \mathbf{L}_{pos}$ [S1, unofficial].
4.  **Normalize Direction:** Compute the normalized light direction vector: $\mathbf{l}_{dir} = \frac{\mathbf{d}}{\|\mathbf{d}\|}$ [S1, unofficial].
5.  **Compute Illumination:** Calculate the shading value $S$ using the dot product of the surface normal $\mathbf{n}$ and the normalized light direction $\mathbf{l}_{dir}$ [S1, unofficial].
    *   Formula: $S = \mathbf{n} \cdot \mathbf{l}_{dir}$ [S1, unofficial].
    *   *Note:* Some implementations clamp the result to $[0, 1]$ to prevent negative shading [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_light_shading(pos, norm, light_position):
    # pos: float3, position of the surface point
    # norm: float3, normalized surface normal (gradient) at pos
    # light_position: float3, position of the point light source
    
    # Step 1 & 2: Get positions (already provided as inputs)
    # pos is the intersection point
    # light_position is the source
    
    # Step 3: Compute vector from light to point
    light_to_point = pos - light_position
    
    # Step 4: Normalize the direction
    distance = length(light_to_point)
    if distance == 0:
        return 0.0 # Avoid division by zero if point is at light source
    normalized_light_dir = light_to_point / distance
    
    # Step 5: Compute dot product of normal and light direction
    # The normal is assumed to be normalized already
    illumination = dot(norm, normalized_light_dir)
    
    # Clamp to [0, 1] to handle back-facing surfaces or negative values
    if illumination < 0.0:
        illumination = 0.0
    if illumination > 1.0:
        illumination = 1.0
        
    return illumination
```

**Parameters:**
*   `lightPosition`: User-defined 3D coordinate [S1, unofficial][S2].
*   `mu`: Not used in the primary definition [S1, unofficial]. Used only in variants [S2].

**Edge Cases and Pitfalls:**
*   **Zero Distance:** If the sampled point coincides with the light source, the direction vector is zero. A guard against division by zero is required [G].
*   **Normal Accuracy:** The quality of shading depends heavily on the accuracy of the computed gradient/normal. Inaccurate normals lead to incorrect dot products [S1, unofficial].
*   **Single Light Source:** Shadows are not computed; occlusion is not handled by this specific model [S1, unofficial].

## Subtopics

### Exponential Attenuation Shading
A simpler lighting approach that assumes light attenuates exponentially from the source, avoiding the need for gradient/normal computation [S2]. It is chosen when normal estimation is difficult or when a simpler, distance-based falloff is preferred over directional shading [S2].
*   **Formula:** $S = \exp(-\mu l)$, where $S$ is the illumination value, $\mu$ is the constant attenuation coefficient, and $l$ is the distance from the illuminated point to the light source [S2].
*   **Steps:**
    1.  User sets the light source position [S2].
    2.  Determine the position of the sampled point along the ray [S2].
    3.  Calculate the distance $l$ between the light source and the sampled point [S2].
    4.  Calculate illumination $S$ using the formula and apply it to the final image [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per shading point, involving basic vector arithmetic (subtraction, normalization, dot product) [S1, unofficial].
*   **Memory Complexity:** $O(1)$, requiring storage for the light position and current sample point [S1, unofficial].
*   **Performance:** Very fast due to the lack of shadow ray tracing or complex BRDF evaluations [S1, unofficial][S2].
*   **Strengths:** Computationally inexpensive; simple to implement [S1, unofficial][S2].
*   **Limitations:** Does not account for shadows or occlusion; relies on accurate normal estimation which can be noisy in volume data; single light source limits realism [S1, unofficial].
*   **When to use:** When computational resources are limited, or for quick previews where directional shading is sufficient [S1, unofficial].
*   **When not to use:** When realistic shadows or complex lighting interactions are required [S1, unofficial].

## Background
The method originates from early volume raycasting implementations, specifically those utilizing OpenCL for parallel processing [S1, unofficial]. It was proposed as a simple illumination model to provide visual cues for shape and depth in volume data without the overhead of more complex lighting models like Phong shading or shadow rays [S1, unofficial][S2]. The exponential attenuation variant is also discussed in the context of X-ray computed tomography visualization, where simplicity is prioritized [S2].

## Sources
[S1] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S2] Method of Volume Rendering for Visualization Three-Dimensional Data in X-Ray Computed Tomography, https://doi.org/10.14529/ctcr160401, fetched 2026-10-26
