# Image Space Outline Generation

Also called: Edge extraction from depth/normal maps, Hertzmann's outline generation

## Overview
Image Space Outline Generation is a post-processing method that renders outlines by extracting edges from depth or normal maps [S1, unofficial][S2, unofficial]. It solves the problem of defining perceptual boundaries in non-photorealistic rendering by leveraging the simplicity of screen-space buffers, distinguishing itself from analytical methods by operating directly on pixel data rather than geometric primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A rendered depth buffer (Z-buffer) and/or a normal buffer (view-space or screen-space normals) of the scene.
*   **Outputs:** A binary or grayscale edge map indicating outline pixels, which is subsequently composited over the base color buffer.

**Data Structures:**
*   **Depth Buffer:** A 2D array of floating-point or fixed-point values representing the distance from the camera to the nearest surface for each pixel.
*   **Normal Buffer:** A 2D array of 3D vectors (typically normalized) representing the surface orientation at each pixel.

**Steps:**
1.  **Depth Edge Extraction:** Compute the absolute difference in depth values between adjacent pixels (typically horizontal and vertical neighbors). If the difference exceeds a threshold $\tau_d$, the pixel is marked as a depth edge. This detects C0 discontinuities (depth jumps) [S1, unofficial].
2.  **Normal Edge Extraction:** Compute the difference in normal vectors between adjacent pixels. This can be done via the dot product (1 - $\mathbf{n}_i \cdot \mathbf{n}_j$) or Euclidean distance. If the difference exceeds a threshold $\tau_n$, the pixel is marked as a normal edge. This detects C1 discontinuities (surface orientation changes) [S1, unofficial].
3.  **Edge Combination:** Combine the depth edges and normal edges using a logical OR operation. The union of these sets yields all perceptually significant silhouettes [S1, unofficial].
4.  **Post-Processing (Optional):** For stylized line qualities (width, smoothness), calculate analytical curves from the discrete edge map pixels [S1, unofficial].

**Formulas:**
Let $D(x,y)$ be the depth value at pixel $(x,y)$ and $\mathbf{N}(x,y)$ be the normal vector at $(x,y)$.
Depth edge condition:
$$ |D(x,y) - D(x+1,y)| > \tau_d \quad \text{or} \quad |D(x,y) - D(x,y+1)| > \tau_d $$
Normal edge condition:
$$ 1 - \mathbf{N}(x,y) \cdot \mathbf{N}(x+1,y) > \tau_n \quad \text{or} \quad 1 - \mathbf{N}(x,y) \cdot \mathbf{N}(x,y+1) > \tau_n $$

**Parameters:**
*   $\tau_d$: Depth threshold. Not given in the sources.
*   $\tau_n$: Normal threshold. Not given in the sources.
*   Filter: Hertzmann recommends the Sobel filter for edge detection, particularly for smoothly varying polygonal meshes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_outlines(depth_map, normal_map, depth_threshold, normal_threshold):
    """
    Generates an outline mask from depth and normal maps.
    
    Args:
        depth_map: 2D array of depth values.
        normal_map: 2D array of 3D normal vectors.
        depth_threshold: Scalar threshold for depth discontinuity.
        normal_threshold: Scalar threshold for normal discontinuity.
        
    Returns:
        edge_mask: 2D boolean array where True indicates an outline pixel.
    """
    height, width = depth_map.shape
    edge_mask = np.zeros((height, width), dtype=bool)
    
    # 1. Depth Edge Detection (C0 discontinuities)
    # Calculate horizontal and vertical depth differences
    depth_diff_x = np.abs(depth_map[:, 1:] - depth_map[:, :-1])
    depth_diff_y = np.abs(depth_map[1:, :] - depth_map[:-1, :])
    
    # Mark pixels where difference exceeds threshold
    # Note: Indices are shifted; we align to the center pixel or use a kernel
    # For simplicity, we mark the pixel if its right or bottom neighbor differs significantly
    depth_edges = np.zeros((height, width), dtype=bool)
    depth_edges[:, :-1] = depth_diff_x > depth_threshold
    depth_edges[:-1, :] = depth_edges[:-1, :] | (depth_diff_y > depth_threshold)
    
    # 2. Normal Edge Detection (C1 discontinuities)
    # Calculate dot product differences for horizontal and vertical neighbors
    # Using a small epsilon to avoid division by zero if normalizing, though inputs are assumed normalized
    normal_diff_x = 1.0 - np.sum(normal_map[:, 1:] * normal_map[:, :-1], axis=2)
    normal_diff_y = 1.0 - np.sum(normal_map[1:, :] * normal_map[:-1, :], axis=2)
    
    normal_edges = np.zeros((height, width), dtype=bool)
    normal_edges[:, :-1] = normal_diff_x > normal_threshold
    normal_edges[:-1, :] = normal_edges[:-1, :] | (normal_diff_y > normal_threshold)
    
    # 3. Combine Edges
    edge_mask = depth_edges | normal_edges
    
    return edge_mask
```

**Edge Cases and Pitfalls:**
*   **Precision:** Depth buffer precision limits the accuracy of depth-based edge detection, especially at large distances or with wide view frustums [S1, unofficial].
*   **Noise:** The method assumes smooth surfaces; noisy geometry or compression artifacts in the depth/normal maps may produce false edges [S1, unofficial].
*   **Discrete Nature:** Detected edges are lists of pixels; they lack continuous geometry, requiring additional processing for smooth line rendering [S1, unofficial].

## Subtopics

### Object Contour NPR
Focuses on silhouettes of closed objects. Choose when object boundaries are the primary visual cue. [Object Contour NPR](787-object-contour-npr.md)

### Lasso Contour Rendering
Uses user-defined or algorithmically generated lasso shapes. Choose for artistic control over contour placement. [Lasso Contour Rendering](1855-lasso-contour-rendering.md)

### Curvature-Controlled Style Contours
Varies line style based on surface curvature. Choose when highlighting geometric complexity is desired. [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)

### Silhouette Rendering
Specifically targets the boundary between front-facing and back-facing surfaces. Choose for clear object separation against backgrounds. [Silhouette Rendering](7959-silhouette-rendering.md)

### Topological Landscape
Maps topological features to visual styles. Choose for abstract or data-driven visualization. [Topological Landscape](8481-topological-landscape.md)

### Boundary Enhancement Volume Style
Enhances boundaries within volumetric data. Choose for volume rendering contexts where internal structures need emphasis. [Boundary Enhancement Volume Style](10863-boundary-enhancement-volume-style.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels $O(W \times H)$, as it involves a constant number of neighbor comparisons per pixel.
*   **Memory Complexity:** $O(W \times H)$ for storing the depth, normal, and edge maps.
*   **Performance:** Described as "reasonably fast" and "simple to implement" compared to analytical methods [S1, unofficial][S2, unofficial].
*   **Strengths:** Simplicity, speed, and ease of integration as a post-processing pass [S1, unofficial][S2, unofficial].
*   **Limitations:** Relies on the quality of the depth/normal buffers; prone to artifacts from buffer precision or noise; edges are pixel-based and may appear jagged without smoothing [S1, unofficial].
*   **When to use:** When real-time performance is critical and the scene geometry is well-defined in screen space.
*   **When not to use:** When analytical line quality is required or when depth/normal information is unavailable or unreliable.

## Background
The method was surveyed by Hertzmann [Her99] and is often associated with his work on image-space outline generation. It is widely used in real-time graphics for its simplicity, as seen in implementations like the outline post-processing pass in voxel engines [S2, unofficial]. The approach distinguishes between C0 (depth) and C1 (normal) discontinuities to capture a comprehensive set of visual edges [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] lazergenixdev/First-Voxel-Engine, https://github.com/lazergenixdev/First-Voxel-Engine, fetched 2026-10-26 (unofficial)
