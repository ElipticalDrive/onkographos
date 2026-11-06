# Frustum-based LOD Selection

Also called: Auto Level of Detail (LOD), Distance-based LOD, Frustum Face Distance LOD, Multi-resolution volumes (LOD), Simple LOD

## Overview
Frustum-based LOD Selection is a method for adaptive volumetric rendering that determines the appropriate resolution level for a volume entity by calculating the distance from the camera to the entity's position relative to the camera's frustum faces [S1, unofficial][S2]. It solves the problem of maintaining responsive frame rates for large-scale datasets by selecting lower resolutions for distant or peripheral rays, thereby minimizing memory bandwidth and sample counts [S2]. Unlike [Footprint-based LOD Selection](3088-footprint-based-lod-selection.md), which ensures the projected voxel size does not exceed a specific pixel threshold to prevent aliasing, this method relies on distance-to-frustum-face metrics to drive LOD selection [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera world-space position, camera view/projection matrices (or inverse MVP), entity world-space position and scale, and a precomputed array of depth thresholds.
*   **Outputs:** An integer LOD index (depth level) for the volume entity.

**Data Structures:**
*   `depthTresholds_`: An array of floats representing distance thresholds for each LOD level. For a maximum depth of $D$, the array contains $D$ elements. The value at index $i$ (where $i$ ranges from $0$ to $D-1$) is calculated as $1.0 / 2^{i+1}$ [S1, unofficial].
*   `transformComponent`: Contains the entity's world-space position and scale factor [S1, unofficial].
*   `activeCameraTransformComponent`: Contains the camera's world-space position [S1, unofficial].

**Numbered Steps:**
1.  **Calculate Camera-to-Entity Distance:** Compute the Euclidean distance between the camera position and the entity's transformed position [S1, S3, unofficial].
    $$ d = \| \mathbf{p}_{camera} - \mathbf{p}_{entity} \| $$
    Where $\mathbf{p}_{camera}$ is the camera position and $\mathbf{p}_{entity}$ is the entity position [S3, unofficial].

2.  **Determine Frustum Face Positions (Optional/Advanced):** Some implementations calculate the specific positions of the left and right frustum faces at the entity's depth to refine the distance metric or for culling [S1, unofficial].
    *   Define Normalized Device Coordinate (NDC) points, e.g., left edge $A = (-1, 0, 0, 1)$ and right edge $B = (1, 0, 0, 1)$ [S1, unofficial].
    *   Transform these NDC points to view space using the inverse Model-View-Projection matrix ($MVP^{-1}$) [S1, unofficial]:
        $$ \mathbf{v}_A = MVP^{-1} \cdot A, \quad \mathbf{v}_B = MVP^{-1} \cdot B $$
    *   Calculate the world-space position of the frustum faces at the entity's distance $d$:
        $$ \mathbf{f}_{left} = \mathbf{p}_{camera} + \mathbf{v}_A \cdot d $$
        $$ \mathbf{f}_{right} = \mathbf{p}_{camera} + \mathbf{v}_B \cdot d $$
    *   Calculate the distance between these faces normalized by screen width $W$:
        $$ \text{face\_dist} = \frac{\| \mathbf{f}_{left} - \mathbf{f}_{right} \|}{W} $$
    *   *Note:* This specific calculation is often used for frustum culling or precise footprint estimation rather than the primary LOD index selection in simpler implementations [S1, unofficial].

3.  **Select LOD Level:** Iterate through the `depthTresholds_` array to find the first threshold that the calculated distance exceeds, adjusted by the entity's scale [S1, unofficial].
    *   For each index $i$ from $0$ to $N-1$:
        *   Calculate the target depth: $L = i + 1$ [S1, unofficial].
        *   Check if the distance exceeds the threshold scaled by the entity's scale $S$:
            $$ \text{if } d > \text{depthTresholds\_}[i] \cdot S $$
        *   If true, set the LOD to $L$ and terminate the search [S1, unofficial].
    *   If no threshold is exceeded, the lowest LOD (highest resolution, typically index 0 or 1 depending on implementation) is selected.

**Formulas:**
*   **Distance:** $d = \| \mathbf{p}_{camera} - \mathbf{p}_{entity} \|$ [S3, unofficial].
*   **Thresholds:** $\text{depthTresholds\_}[i] = \frac{1.0}{2^{i+1}}$ for $i \in [0, \text{maxSvoDepth}-1]$ [S1, unofficial].
*   **Simple LOD Decision (Binary Example):**
    $$ LOD = \begin{cases} 2 & \text{if } \| \mathbf{p}_{camera} - (\mathbf{M} \cdot [\mathbf{p}_{voxel}, 1]) \| > 0.5 \\ 1 & \text{otherwise} \end{cases} $$
    Where $\mathbf{M}$ is the model matrix and $\mathbf{p}_{voxel}$ is the voxel position [S3, unofficial].

**Parameters:**
*   `depthTresholds_`: Array of float thresholds [S1, unofficial].
*   `lod`: Integer uniform variable controlling the selected LOD state in the shader [S3, unofficial].
*   `camera_position`: World-space position of the camera [S3, unofficial].
*   `transformComponent.getScale()`: Scale factor of the volume entity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_frustum_lod(camera_pos, entity_pos, entity_scale, depth_thresholds):
    """
    Selects the LOD level for a volume entity based on camera distance.
    
    Args:
        camera_pos: vec3, world-space position of the camera.
        entity_pos: vec3, world-space position of the entity.
        entity_scale: float, scale factor of the entity.
        depth_thresholds: list of float, precomputed thresholds [1/2, 1/4, 1/8, ...].
        
    Returns:
        int: The selected LOD level (1-based index).
    """
    # Step 1: Calculate Euclidean distance
    dx = camera_pos.x - entity_pos.x
    dy = camera_pos.y - entity_pos.y
    dz = camera_pos.z - entity_pos.z
    distance = (dx*dx + dy*dy + dz*dz)**0.5
    
    # Step 2: Iterate through thresholds
    # Note: depth_thresholds[i] corresponds to LOD level i+1
    for i, threshold in enumerate(depth_thresholds):
        # Adjust threshold by entity scale
        adjusted_threshold = threshold * entity_scale
        
        # Check if distance exceeds threshold
        if distance > adjusted_threshold:
            # Return the LOD level (1-based)
            return i + 1
            
    # If distance is smaller than all thresholds, return the highest resolution (LOD 1)
    return 1
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors in distance calculation may cause flickering between LOD levels if thresholds are very close. [G] Use a small epsilon or hysteresis when switching LODs to prevent visual popping.
*   **Boundaries:** If the entity is partially outside the frustum, simple distance-based LOD may not account for the visible portion. [G] Combine with frustum culling to avoid rendering invisible high-res volumes.
*   **Degenerate Input:** If `entity_scale` is zero or negative, the threshold adjustment fails. [G] Clamp scale to a minimum positive value (e.g., $10^{-6}$).

## Subtopics

### [Footprint-based LOD Selection](3088-footprint-based-lod-selection.md)
Determines LOD by ensuring the projected voxel size on the screen does not exceed a pixel threshold, preventing aliasing. It uses the actual projected footprint rather than distance to frustum faces.

### [Cone-Footprint LOD and Step Sizing](5178-cone-footprint-lod-and-step-sizing.md)
Uses sample density proportional to screen content and cone footprints, differing from the fixed median footprint or simple distance metrics of this method.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of LOD thresholds (typically small, e.g., 5-10), making it very fast for CPU-side LOD selection [S1, unofficial].
*   **Memory Complexity:** $O(D)$ for storing the threshold array, where $D$ is the maximum LOD depth [S1, unofficial]. The volume data itself requires multi-resolution storage (MIP-map pyramid) [S2].
*   **Measured Performance:** Suitable for large-scale datasets like geophysical simulations, significantly cutting memory and sample requirements while maintaining perceptual quality [S2].
*   **Strengths:** Simple to implement; effective for distant objects; reduces bandwidth for peripheral rays [S2].
*   **Limitations:** Does not explicitly account for the projected size of voxels on the screen, which can lead to aliasing or over-sampling if the camera moves laterally without changing distance [S2]. It is less precise than footprint-based methods for maintaining consistent visual quality across the screen [S2].
*   **When to Use:** Large volumes where performance is critical and visual artifacts from distance-based LOD are acceptable or less noticeable than those from footprint-based methods.
*   **When Not to Use:** When strict aliasing control is required, or when the camera moves primarily laterally relative to the volume.

## Background
Frustum-based LOD selection is a technique used in volume ray casting to optimize rendering performance. It is part of the broader family of Level-of-Detail (LOD) methods for volume rendering. The method leverages hierarchical representations such as octree-based voxel MIP-map pyramids, where each level reduces resolution [S2]. It is particularly useful for interactive previews and large-scale datasets [S2]. Implementations can be found in various graphics frameworks and research projects, including simple LOD activations in Marching Cubes/Tetrahedra visualizers [S3, unofficial] and SVO (Sparse Voxel Octree) renderers [S1, unofficial].

## Sources
[S1] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S3] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
