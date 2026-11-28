# Neural Duplex Radiance Field

Also called: Duplex Radiance Field

## Overview
The Neural Duplex Radiance Field is a real-time view synthesis method that replaces dense volumetric ray marching with a two-surface proxy representation. It solves the computational bottleneck of evaluating deep MLPs for every sample point by learning to map the two endpoints of a ray segment (defined by intersections with inner and outer meshes) and the view direction to the final pixel color. It sets itself apart from its parent [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) by abandoning per-point MLP queries in favor of vertex-attached features interpolated via hardware rasterization, and from siblings like [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md) by relying on explicit mesh proxies rather than hybrid voxel/grid structures for the primary geometry.

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained NeRF density field (used to generate proxies), camera parameters (view direction $\mathbf{d}$), and two proxy meshes (inner $\mathcal{T}_i$ and outer $\mathcal{T}_o$).
*   **Output:** Pixel color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
1.  **Inner Mesh ($\mathcal{T}_i$):** Extracted from the density field using a threshold of $10^{-4}$. Contains vertices $V_i$ and triangles $T_i$.
2.  **Outer Mesh ($\mathcal{T}_o$):** Extracted from the density field using a threshold of $10^{-2}$. Contains vertices $V_o$ and triangles $T_o$.
3.  **Vertex Features:** Each vertex $v_k$ in both meshes has an associated learnable feature vector $\mathbf{f}_k \in \mathbb{R}^N$.
    *   $N=20$ for CUDA implementation.
    *   $N=8$ for WebGL implementation.

**Numbered Steps:**
1.  **Mesh Extraction:** Extract $\mathcal{T}_i$ and $\mathcal{T}_o$ from a pretrained NeRF density field using marching cubes at the specified density thresholds.
2.  **Feature Initialization:** Initialize learnable feature vectors $\mathbf{f}_k$ for every vertex $v_k$ in $\mathcal{T}_i$ and $\mathcal{T}_o$.
3.  **Ray Intersection:** For each pixel, cast a ray with direction $\mathbf{d}$. Compute the depth-sorted intersection points $\mathbf{x}_1$ with the outer mesh $\mathcal{T}_o$ and $\mathbf{x}_2$ with the inner mesh $\mathcal{T}_i$.
4.  **Feature Interpolation:** For each intersection point $\mathbf{x} \in \{\mathbf{x}_1, \mathbf{x}_2\}$, determine the containing triangle and compute barycentric coordinates. Interpolate the vertex features to obtain $\mathbf{f}(\mathbf{x})$. This is performed via hardware rasterization.
5.  **Color Aggregation:** Pass the interpolated features and view direction through a shallow aggregation network $F_\Theta$ to predict the color.

**Formula:**
The rendering equation is defined as:
$$
\mathbf{c} = F_{\Theta}(\mathbf{f}(\mathbf{x}_1), \mathbf{f}(\mathbf{x}_2), \mathbf{d})
$$
where:
*   $\mathbf{x}_1 \in \mathcal{T}_o$ is the intersection with the outer mesh.
*   $\mathbf{x}_2 \in \mathcal{T}_i$ is the intersection with the inner mesh.
*   $\mathbf{d}$ is the view direction vector.
*   $F_{\Theta}$ is a shallow network (3-layer convolutional network in CUDA, simplified in WebGL).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, inner_mesh, outer_mesh, aggregation_net):
    """
    Renders a single ray using the Neural Duplex Radiance Field.
    
    Args:
        ray_origin: 3D vector (x, y, z)
        ray_direction: 3D vector (x, y, z), normalized
        inner_mesh: Object containing vertices V_i, triangles T_i, and features F_i
        outer_mesh: Object containing vertices V_o, triangles T_o, and features F_o
        aggregation_net: Function F_Theta taking (f1, f2, d) -> color
        
    Returns:
        color: 3D vector (R, G, B)
    """
    
    # Step 1: Find intersections with outer mesh
    # Returns intersection point x1 and barycentric coordinates (u, v, w)
    # If no intersection, x1 is None
    x1, bary1 = ray_triangle_intersection(ray_origin, ray_direction, outer_mesh)
    
    # Step 2: Find intersections with inner mesh
    # Returns intersection point x2 and barycentric coordinates (u, v, w)
    # If no intersection, x2 is None
    x2, bary2 = ray_triangle_intersection(ray_origin, ray_direction, inner_mesh)
    
    # Step 3: Interpolate features
    # Initialize feature vectors to zero or a default value if intersection fails
    f1 = zeros(N)
    f2 = zeros(N)
    
    if x1 is not None:
        # Interpolate features from outer mesh vertices using barycentric coords
        f1 = interpolate_features(outer_mesh.vertices, outer_mesh.features, x1, bary1)
        
    if x2 is not None:
        # Interpolate features from inner mesh vertices using barycentric coords
        f2 = interpolate_features(inner_mesh.vertices, inner_mesh.features, x2, bary2)
        
    # Step 4: Aggregate color
    # The aggregation network handles cases where one intersection is missing
    # by relying on the view direction and available features
    color = aggregation_net(f1, f2, ray_direction)
    
    return color

def ray_triangle_intersection(origin, direction, mesh):
    """
    Finds the closest intersection of a ray with any triangle in the mesh.
    Returns the intersection point and barycentric coordinates.
    """
    closest_dist = infinity
    closest_point = None
    closest_bary = None
    
    for triangle in mesh.triangles:
        # Perform ray-triangle intersection test
        dist, barycentric = intersect_ray_triangle(origin, direction, triangle)
        
        if dist is not None and dist < closest_dist:
            closest_dist = dist
            closest_point = origin + direction * dist
            closest_bary = barycentric
            
    return closest_point, closest_bary

def interpolate_features(vertices, features, point, barycentric):
    """
    Interpolates feature vector using barycentric coordinates.
    """
    # barycentric is (w0, w1, w2) corresponding to vertices 0, 1, 2
    v0, v1, v2 = vertices[0], vertices[1], vertices[2]
    f0, f1, f2 = features[0], features[1], features[2]
    
    return f0 * barycentric[0] + f1 * barycentric[1] + f2 * barycentric[2]
```

**Parameters:**
*   **Feature Dimension ($N$):** 20 (CUDA), 8 (WebGL).
*   **Inner Mesh Density Threshold:** $10^{-4}$.
*   **Outer Mesh Density Threshold:** $10^{-2}$.
*   **Aggregation Network:** 3-layer convolutional network (CUDA); simplified network without final convolution layer (WebGL).

**Edge Cases and Pitfalls:**
*   **Missing Intersections:** Not all rays hit both proxy surfaces. The view direction $\mathbf{d}$ is appended to the input of $F_\Theta$ to allow the network to handle cases where one intersection is missing.
*   **Occlusion Artifacts:** Local sampling around a single surface suffers from missing geometry or visible occlusions, leading to hole artifacts. The duplex field mitigates this by learning to combine locations on two estimated surfaces.
*   **High-Frequency Artifacts:** Training from scratch fails to match the fidelity of original NeRFs. Multi-view distillation from a pretrained teacher model (e.g., TensoRF) is required to guide optimization.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray for feature lookup and aggregation, dominated by rasterization and ray-triangle intersection. Significantly faster than dense volumetric sampling $O(N)$ where $N$ is the number of samples.
*   **Memory Complexity:** $O(V)$ where $V$ is the number of vertices in the proxy meshes. Much lower than storing dense voxel grids or large hash tables for every sample point.
*   **Performance:** On an NVIDIA RTX 2080 Ti, achieves real-time FPS (specific FPS values depend on resolution and scene complexity, but cited as comparable to KiloNeRF/SNeRG in terms of efficiency class). WebGL implementation offers cross-platform compatibility with some performance degradation.
*   **Strengths:** Real-time rendering capability, reduced memory footprint compared to dense volumetric methods, handles view-dependent appearance via the aggregation network.
*   **Limitations:** Relies on the quality of the proxy meshes; poor mesh extraction leads to artifacts. Requires a pretrained teacher model for stable training. May struggle with complex internal structures not captured by the two surface proxies.
*   **When to use:** Real-time view synthesis where high fidelity is needed but dense volumetric rendering is too slow.
*   **When not to use:** Scenes with highly complex internal geometry that cannot be approximated by two surfaces, or when training from scratch without distillation is required.

## Background
The Neural Duplex Radiance Field was introduced in "Learning Neural Duplex Radiance Fields for Real-Time View Synthesis" (CVPR 2023) [S1]. It addresses the computational cost of densely evaluating deep MLPs in NeRF-based methods. By leveraging two geometric proxy meshes and hardware rasterization, it enables real-time rendering. The method builds on the concept of using explicit geometry to guide neural rendering, contrasting with implicit volumetric approaches. It uses multi-view distillation from a pretrained teacher model (like TensoRF) to overcome the lack of high-frequency details when training from scratch.

## Sources
[S1] Learning Neural Duplex Radiance Fields for Real-Time View Synthesis, https://doi.org/10.1109/cvpr52729.2023.00803, fetched 2026-10-26
