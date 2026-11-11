# Point Cloud Rendering


## Overview
Point Cloud Rendering projects a set of 3D points into screen space and aggregates their contributions to determine the final pixel color [S1, unofficial]. It is primarily distinguished by its differentiability, allowing gradients to flow from the rendered image back to the 3D point positions and attributes [S1, unofficial].

## How it works
**Inputs:** A set of 3D points with attributes (e.g., color) and camera parameters [S1, unofficial].
**Outputs:** A 2D image with color values for each pixel [S1, unofficial].

**Steps:**

1. **Calculate Screen Space Coordinates:**
   Compute the screen space coordinates $p_{uv}$ of a 3D point $P_{xyz}$ via matrix multiplication [S1, unofficial]. This step is differentiable with respect to the point positions [S1, unofficial].
   $$
   \begin{bmatrix} u_{clip} \\ v_{clip} \\ w_{clip} \end{bmatrix} = M \cdot \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}
   $$
   $$
   u = \frac{u_{clip}}{w_{clip}}, \quad v = \frac{v_{clip}}{w_{clip}}
   $$
   where $M$ is the camera transformation matrix [G].

2. **Compute Influence:**
   Calculate the spatial influence $I_i$ of each point on target pixels [S1, unofficial]. Assuming an influence size of one pixel leads to very sparse images [S1, unofficial]. To solve this, influences larger than one pixel are created using truncated Gaussian blur [S1, unofficial] or distance-based influence values [S1, unofficial]. A large influence size prevents derivatives from becoming zero for distant points but reduces rendering quality [S1, unofficial].

3. **Aggregate Pixel Color:**
   Aggregate points based on their influences and z-values to determine the final color for each pixel [S1, unofficial]. Methods include:
   * **Weighted Sum:** Compute a weighted sum of the points' colors based on their influences [S1, unofficial]. This approach does not account for occlusion [S1, unofficial].
   * **Closest Point:** Select the single closest point to the camera for a given pixel to handle occlusion [S1, unofficial].
   * **K-Nearest Neighbors:** Select the $K$ nearest 3D points to the camera and weigh them based on spatial influence [S1, unofficial].
   * **Distance-Weighted Sum:** Weigh all points by both spatial influence and distance from the camera [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_point_cloud(points, camera_matrix, image_width, image_height, sigma=1.0, radius=3.0):
    """
    Render a point cloud using weighted sum aggregation with truncated Gaussian influence.
    points: List of dicts with 'pos' (vec3), 'color' (vec3)
    camera_matrix: 4x4 transformation matrix
    """
    image = [[(0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    z_buffer = [[float('inf') for _ in range(image_width)] for _ in range(image_height)]

    def mat_vec_mul(M, v):
        return [sum(M[r][c] * v[c] for c in range(4)) for r in range(4)]

    def truncated_gaussian(dist):
        if dist > radius:
            return 0.0
        return (1.0 / (2.0 * 3.14159 * sigma**2)) * (2.71828 ** (-dist**2 / (2.0 * sigma**2)))

    for p in points:
        p_homo = [p['pos'][0], p['pos'][1], p['pos'][2], 1.0]
        proj = mat_vec_mul(camera_matrix, p_homo)
        if proj[3] <= 0.0:
            continue
        u = proj[0] / proj[3]
        v = proj[1] / proj[3]
        z = proj[2] / proj[3]
        if not (0 <= u < image_width and 0 <= v < image_height):
            continue

        for dy in range(-int(radius), int(radius) + 1):
            for dx in range(-int(radius), int(radius) + 1):
                y_idx = int(v) + dy
                x_idx = int(u) + dx
                if 0 <= y_idx < image_height and 0 <= x_idx < image_width:
                    dist = (dx**2 + dy**2)**0.5
                    I = truncated_gaussian(dist)
                    if I > 0:
                        if z < z_buffer[y_idx][x_idx]:
                            z_buffer[y_idx][x_idx] = z
                            image[y_idx][x_idx] = (
                                image[y_idx][x_idx][0] + I * p['color'][0],
                                image[y_idx][x_idx][1] + I * p['color'][1],
                                image[y_idx][x_idx][2] + I * p['color'][2]
                            )
    return image
# Note: Sources describe alternative aggregation strategies (closest point, K-nearest, distance-weighted) but do not provide pseudocode for them [S1, unofficial].
```

**Edge Cases and Pitfalls:**
* **Sparsity:** Assuming point influence size is one pixel results in very sparse images [S1, unofficial].
* **Occlusion:** Simple weighted sum approaches do not account for occlusion [S1, unofficial].
* **Convergence Trade-off:** Large influence sizes aid optimization convergence by preventing zero derivatives for distant points but degrade rendering quality [S1, unofficial].
* **Depth Density:** When converting depth images to point clouds, point density decreases inversely proportional to distance [S1, unofficial].

## Subtopics

### Voxel-based Point Rendering
Generates 3D voxels from the point cloud and renders them using a voxel-based method [S1, unofficial]. See [Voxel Rendering](52-voxel-rendering.md).

### Depth Image Rendering
Converts depth to a point cloud and projects it to a virtual camera view, serving as a special case of point cloud rendering [S1, unofficial]. Used in self-supervised learning of monocular depth estimation [S1, unofficial]. See [Depth Rendering](52-depth-rendering.md).

## Operating characteristics
* **Time Complexity:** Not given in the sources. [G]
* **Memory Complexity:** Not given in the sources. [G]
* **Strengths:** Differentiable with respect to point positions and attributes, enabling optimization in neural networks [S1, unofficial].
* **Limitations:** Sparse images if influence is small. Occlusion handling is non-trivial with simple blending [S1, unofficial]. Quality-convergence trade-off with influence size [S1, unofficial].
* **When to use:** Differentiable rendering tasks, neural implicit representations, and scenarios with reduced 3D supervision [S1, unofficial].
* **When not to use:** Not given in the sources. [G]

## Background
Point clouds have become a natural choice for data representation in differentiable rendering and scene understanding with reduced 3D supervision [S1, unofficial]. The method is used in a variety of practical 3D problems [S1, unofficial]. It supports self-supervised learning of monocular depth estimation and neural implicit representations [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
