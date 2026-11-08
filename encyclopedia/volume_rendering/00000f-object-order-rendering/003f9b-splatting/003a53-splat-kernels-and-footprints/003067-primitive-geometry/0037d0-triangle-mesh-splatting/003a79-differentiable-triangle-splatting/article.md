# Differentiable Triangle Splatting

Also called: Triangle Splatting

## Overview
Differentiable Triangle Splatting is a real-time differentiable renderer that represents a scene as a soup of unstructured, disconnected 3D triangles and optimizes them via end-to-end gradients by rendering each triangle as a differentiable splat in screen space [S1]. It merges the adaptability of Gaussian splatting with the efficiency of triangle primitives, surpassing 3D Gaussian Splatting (3DGS), 2D Gaussian Splatting (2DGS), and 3D Convex Splatting (3DCS) in visual fidelity, training speed, and rendering throughput [S1]. Unlike Gaussian methods that require complex depth-dependent covariance adjustments, Triangle Splatting uses a single smoothness parameter $\sigma$ for all triangles, ensuring consistent spatial influence regardless of depth or projected scale [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D triangles $T_{3D}$, each defined by three vertices $v_i \in \mathbb{R}^3$, a color $c$, a smoothness parameter $\sigma$, and an opacity $o$. Camera parameters: intrinsic matrix $K$, rotation $R$, and translation $t$.
*   **Output:** A rendered image $I(p)$ and gradients for optimizing triangle vertices, colors, $\sigma$, and $o$.

**Data Structures:**
Each triangle primitive stores:
*   `vertices`: Array of 3 vectors, each size 3 (float32).
*   `color`: Vector of size 3 (float32).
*   `sigma`: Scalar (float32).
*   `opacity`: Scalar (float32).

**Steps:**

1.  **Projection:** Project each 3D vertex $v_i$ to the 2D image plane using the pinhole camera model:
    $$q_i = K(R v_i + t)$$
    where $q_i \in \mathbb{R}^2$ forms the projected 2D triangle $T_{2D}$ [S1].

2.  **Signed Distance Field (SDF) Computation:** For each pixel $p$ within the bounding box of $T_{2D}$, compute the SDF $\phi(p)$:
    $$\phi(\mathbf{p}) = \max_{i \in \{1,2,3\}} L_i(\mathbf{p})$$
    where $L_i(\mathbf{p}) = \mathbf{n}_i \cdot \mathbf{p} + d_i$. The $\mathbf{n}_i$ are unit normals of the triangle edges pointing outward, and $d_i$ are offsets defining the zero-level set [S1]. $\phi(\mathbf{p})$ is positive outside the triangle, negative inside, and zero on the boundary [S1].

3.  **Incenter Calculation:** Find the incenter $\mathbf{s} \in \mathbb{R}^2$ of $T_{2D}$, defined as the point within the triangle where $\phi(\mathbf{p})$ is minimum [S1].

4.  **Window Function Evaluation:** Compute the window function $I(\mathbf{p})$ for pixel $p$:
    $$I(\mathbf{p}) = \text{ReLU}\left(\frac{\phi(\mathbf{p})}{\phi(\mathbf{s})}\right)^\sigma$$
    At the incenter $\mathbf{s}$, $I(\mathbf{s})=1$; at the boundary where $\phi(\mathbf{p})=0$, $I(\mathbf{p})=0$; outside the triangle where $\phi(\mathbf{p})>0$, $I(\mathbf{p})=0$ [S1].

5.  **Accumulation:** Accumulate the color of each image pixel $p$ by summing contributions from all overlapping triangles in depth order, using $I(\mathbf{p})$ as the opacity weight for each triangle's color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_window_function(p, triangle_vertices, sigma):
    """
    Computes the window function value I(p) for a pixel p relative to a projected triangle.
    
    Args:
        p: tuple (x, y) of pixel coordinates.
        triangle_vertices: list of 3 tuples [(x1,y1), (x2,y2), (x3,y3)].
        sigma: smoothness parameter.
        
    Returns:
        float: Window function value I(p).
    """
    # Step 2: Compute SDF components
    # Calculate edge normals pointing outward and offsets
    # Edge i connects vertex i to vertex (i+1)%3
    edges = []
    for i in range(3):
        v1 = triangle_vertices[i]
        v2 = triangle_vertices[(i + 1) % 3]
        # Edge vector
        ex = v2[0] - v1[0]
        ey = v2[1] - v1[1]
        # Outward normal (rotate edge vector 90 degrees clockwise for standard screen coords? 
        # Source says "pointing outside". For counter-clockwise winding, outward is (dy, -dx).
        # We assume standard math convention where outward normal is computed from edge geometry.
        # Let's compute the line equation n . p + d = 0.
        # Normal vector (nx, ny) perpendicular to edge.
        # If edge is (dx, dy), normal is (dy, -dx) or (-dy, dx).
        # We need the one pointing OUTSIDE.
        # Centroid is inside. Check which normal points away from centroid.
        cx = sum(v[0] for v in triangle_vertices) / 3.0
        cy = sum(v[1] for v in triangle_vertices) / 3.0
        
        # Candidate normals
        nx1 = ey
        ny1 = -ex
        nx2 = -ey
        ny2 = ex
        
        # Check direction against centroid
        # Vector from edge midpoint to centroid
        mx = (v1[0] + v2[0]) / 2.0
        my = (v1[1] + v2[1]) / 2.0
        dx_c = cx - mx
        dy_c = cy - my
        
        # Dot product of normal and vector to centroid. 
        # If dot > 0, normal points towards centroid (inside). We want outside.
        dot1 = nx1 * dx_c + ny1 * dy_c
        dot2 = nx2 * dx_c + ny2 * dy_c
        
        if dot1 > 0:
            nx, ny = nx2, ny2
        else:
            nx, ny = nx1, ny1
            
        # Normalize
        length = math.sqrt(nx*nx + ny*ny)
        if length == 0:
            return 0.0
        nx /= length
        ny /= length
        
        # Offset d such that n . v1 + d = 0 => d = - n . v1
        d = -(nx * v1[0] + ny * v1[1])
        edges.append((nx, ny, d))
        
    # Compute Li(p) for each edge
    li_values = []
    for (nx, ny, d) in edges:
        li = nx * p[0] + ny * p[1] + d
        li_values.append(li)
        
    # SDF phi(p) = max(Li(p))
    phi_p = max(li_values)
    
    # Step 3: Find incenter s (minimum phi inside triangle)
    # The incenter is the point where the distance to all edges is equal and maximal inside.
    # For a triangle, the incenter is the center of the inscribed circle.
    # However, the source defines s as the point where phi is minimum.
    # Since phi is negative inside, the minimum is the most negative value.
    # The incenter is the center of the inscribed circle.
    # We can compute it using barycentric coordinates or standard geometry formulas.
    # Let a, b, c be side lengths opposite to vertices 0, 1, 2.
    a = math.sqrt((triangle_vertices[1][0]-triangle_vertices[2][0])**2 + (triangle_vertices[1][1]-triangle_vertices[2][1])**2)
    b = math.sqrt((triangle_vertices[0][0]-triangle_vertices[2][0])**2 + (triangle_vertices[0][1]-triangle_vertices[2][1])**2)
    c = math.sqrt((triangle_vertices[0][0]-triangle_vertices[1][0])**2 + (triangle_vertices[0][1]-triangle_vertices[1][1])**2)
    
    # Incenter coordinates
    s_x = (a * triangle_vertices[0][0] + b * triangle_vertices[1][0] + c * triangle_vertices[2][0]) / (a + b + c)
    s_y = (a * triangle_vertices[0][1] + b * triangle_vertices[1][1] + c * triangle_vertices[2][1]) / (a + b + c)
    s = (s_x, s_y)
    
    # Compute phi(s)
    li_values_s = []
    for (nx, ny, d) in edges:
        li = nx * s[0] + ny * s[1] + d
        li_values_s.append(li)
    phi_s = max(li_values_s)
    
    # Step 4: Window function
    # I(p) = ReLU(phi(p) / phi(s))^sigma
    # Note: phi(s) is negative. phi(p) is negative inside.
    # Ratio is positive.
    if phi_s == 0:
        return 0.0
        
    ratio = phi_p / phi_s
    
    if ratio < 0:
        return 0.0
    else:
        return ratio ** sigma

def render_triangle_soup(triangles, camera_K, camera_R, camera_t, image_width, image_height):
    """
    Renders a soup of triangles using differentiable splatting.
    """
    # Initialize image buffer
    image = [[0.0 for _ in range(image_width)] for _ in range(image_height)]
    
    # Sort triangles by depth (average z of projected vertices) for correct alpha blending
    # Note: Source says "depth order".
    projected_triangles = []
    for tri in triangles:
        verts_3d = tri['vertices']
        # Project vertices
        verts_2d = []
        avg_z = 0.0
        for v in verts_3d:
            # v is (x,y,z)
            # Apply rotation and translation
            # R is 3x3, t is 3x1
            # p_cam = R * v + t
            # q = K * p_cam
            # Simplified projection for 2D coords
            # Assuming standard pinhole: x' = fx * X/Z + cx, y' = fy * Y/Z + cy
            # We need the full 3D transform first.
            # Let's assume helper functions for matrix mult exist or implement inline.
            # For brevity, we assume `project_vertex` returns (x,y,z) in clip space or similar.
            # Actually, source says q_i = K(R v_i + t).
            # Let's implement basic projection.
            # R v + t
            rx = camera_R[0][0]*v[0] + camera_R[0][1]*v[1] + camera_R[0][2]*v[2] + camera_t[0]
            ry = camera_R[1][0]*v[0] + camera_R[1][1]*v[1] + camera_R[1][2]*v[2] + camera_t[1]
            rz = camera_R[2][0]*v[0] + camera_R[2][1]*v[1] + camera_R[2][2]*v[2] + camera_t[2]
            
            if rz <= 0: # Behind camera
                continue
                
            # K * [rx, ry, rz]^T
            # K = [[fx, 0, cx], [0, fy, cy], [0, 0, 1]]
            qx = camera_K[0][0] * (rx / rz) + camera_K[0][2]
            qy = camera_K[1][1] * (ry / rz) + camera_K[1][2]
            verts_2d.append((qx, qy))
            avg_z += rz
        avg_z /= 3
        
        if len(verts_2d) == 3:
            projected_triangles.append({
                'verts': verts_2d,
                'color': tri['color'],
                'sigma': tri['sigma'],
                'opacity': tri['opacity'],
                'depth': avg_z
            })
            
    # Sort by depth (far to near or near to far? Standard alpha blending is near to far)
    # Source says "depth order". Usually implies sorting.
    projected_triangles.sort(key=lambda t: t['depth'])
    
    for tri in projected_triangles:
        verts = tri['verts']
        sigma = tri['sigma']
        color = tri['color']
        opacity = tri['opacity']
        
        # Determine bounding box
        min_x = max(0, int(min(v[0] for v in verts)))
        max_x = min(image_width, int(max(v[0] for v in verts)) + 1)
        min_y = max(0, int(min(v[1] for v in verts)))
        max_y = min(image_height, int(max(v[1] for v in verts)) + 1)
        
        for y in range(min_y, max_y):
            for x in range(min_x, max_x):
                p = (x, y)
                # Check if pixel is roughly inside bounding box of triangle to save computation
                # More precise check: compute window function
                w = compute_window_function(p, verts, sigma)
                
                # If w is effectively 0, skip
                if w < 1e-6:
                    continue
                    
                # Accumulate color
                # I(p) = sum( I_tri(p) * c_tri * alpha_tri ) / sum( I_tri(p) * alpha_tri ) ?
                # Source says: "treating the value I(p) as opacity" and "accumulating contributions... in depth order"
                # Standard alpha blending: C_out = C_in * (1 - alpha) + C_new * alpha
                # Here, alpha = I(p) * opacity? Or just I(p)?
                # Source: "using I(p) as the opacity weight for each triangle's color"
                # Let's assume alpha_blend = I(p) * opacity
                alpha_blend = w * opacity
                
                # Current pixel color
                curr_r, curr_g, curr_b = image[y][x]
                
                # New color contribution
                new_r = color[0] * alpha_blend
                new_g = color[1] * alpha_blend
                new_b = color[2] * alpha_blend
                
                # Blend
                # C_out = C_in * (1 - alpha_blend) + C_new
                # This assumes I(p) acts as the alpha channel directly.
                image[y][x] = (
                    curr_r * (1 - alpha_blend) + new_r,
                    curr_g * (1 - alpha_blend) + new_g,
                    curr_b * (1 - alpha_blend) + new_b
                )
                
    return image
```

**Parameters:**
*   $\sigma$ (smoothness parameter): Controls the falloff of the window function. As $\sigma$ decreases, the function approximates a hard triangle boundary; as $\sigma$ increases, the support exceeds the triangle footprint, becoming globally supported with value 0.5 everywhere in the limit [S1].

**Edge Cases and Pitfalls:**
*   If $\sigma$ is too large, the window function support exceeds the triangle footprint, causing every triangle to contribute to every pixel (globally supported with value 0.5), which is unsuitable for rasterization [S1].
*   Precision issues may arise when computing the incenter or SDF for degenerate triangles (zero area).

## Subtopics

### [Depth-dependent scaling in Triangle Splatting](9765-depth-dependent-scaling-in-triangle-splatting.md)
This property ensures that the spatial influence of a triangle splat remains consistent regardless of its depth or projected scale, allowing a single smoothness parameter $\sigma$ to be used for all triangles in the scene. It distinguishes Triangle Splatting from methods like 3D Gaussian Splatting, which require complex depth-dependent covariance adjustments to maintain consistent visual appearance across different distances.

### MeshSplatting
MeshSplatting extends Triangle Splatting by optimizing fully opaque, connected triangles, jointly refining geometry and appearance through differentiable rendering to produce coherent surfaces [S2, unofficial]. The method produces coherent surfaces that integrate seamlessly with real-time rendering engines, enabling applications such as augmented and virtual reality without any postprocessing [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achieved, with reported throughput of over 2,400 FPS at 1280×720 resolution for the optimized triangle soup [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Surpasses 3D Gaussian Splatting (3DGS), 2D Gaussian Splatting (2DGS), and 3D Convex Splatting (3DCS) in visual fidelity, training speed, and rendering throughput [S1].
*   **Strengths:** Direct compatibility with standard mesh-based renderers and game engines; leverages decades of GPU-accelerated triangle processing [S1][S2, unofficial].
*   **Limitations:** Requires careful tuning of $\sigma$ to avoid global support artifacts [S1].
*   **When to use:** When high visual fidelity and real-time performance are required, and integration with existing graphics pipelines is desired [S1].
*   **When not to use:** When $\sigma$ cannot be tuned appropriately, leading to performance degradation due to global contributions [S1].

## Background
Triangle Splatting was introduced in "Triangle Splatting for Real-Time Radiance Field Rendering" [S1]. It represents a step toward mesh-aware neural rendering, bridging classical computer graphics with modern differentiable frameworks [S2, unofficial]. MeshSplatting, an extension, is described in the thesis "Bridging neural rendering and classical computer graphics" [S2, unofficial].

## Sources
[S1] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S2] Bridging neural rendering and classical computer graphics, https://core.ac.uk/works/363431310, fetched 2026-10-29 (unofficial)
