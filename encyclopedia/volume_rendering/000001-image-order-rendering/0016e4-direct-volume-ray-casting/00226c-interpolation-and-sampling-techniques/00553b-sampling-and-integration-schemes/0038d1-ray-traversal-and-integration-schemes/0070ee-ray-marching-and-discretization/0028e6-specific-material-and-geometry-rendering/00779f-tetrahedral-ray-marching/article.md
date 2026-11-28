# Tetrahedral Ray Marching

Also called: Differentiable rendering in a tetrahedral mesh

## Overview
Tetrahedral Ray Marching is a ray traversal and integration method for non-uniform tetrahedral grids that efficiently identifies entry and exit points of viewing rays within a 3D mesh [S1, unofficial]. It solves the problem of sparse or irregular volumetric sampling by projecting vertices to determine exit faces and including camera centers in the mesh to simplify entry point identification [S1, unofficial]. It sets itself apart from sibling methods like [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md) by using a tetrahedral topology with explicit geometric traversal rather than a uniform voxel grid, and from [DVGO Density and Color Rendering](12839-dvgo-density-and-color-rendering.md) by relying on explicit mesh connectivity and Signed Distance Field (SDF) interpolation rather than implicit neural density fields on a grid.

## How it works
**Inputs:**
- Camera center $o$ and ray direction $v$.
- Tetrahedral mesh where each tetrahedron has 4 vertices.
- SDF values $sdf(v_i)$ at each vertex $v_i$.
- Color networks $g_{geo}$ and $g_{fine}$ [S1, unofficial].

**Outputs:**
- A list of visited tetrahedra with their entry points $in(t)$ and exit points $out(t)$ for each ray [S1, unofficial].

**Data Structures:**
- **Tetrahedral Mesh:** Defined by vertices and connectivity. The mesh includes "virtual" tetrahedra or markers containing camera centers to facilitate entry detection [S1, unofficial].
- **Segment List:** For each ray, a list of segments $s(t) = [in(t), out(t)]$ where $t \in [0, n]$ [S1, unofficial].

**Steps:**
1. **Entry Point Identification:** Determine the tetrahedron containing the camera center $o$. This tetrahedron provides the initial entry point for the ray [S1, unofficial].
2. **Ray Traversal Loop:** While the ray has not exited the mesh and the segment count is below the maximum:
    a. **Find Exit Face:**
        i. Project all four vertices of the current tetrahedron onto a plane centered at the camera center $o$ with normal vector $v$ [S1, unofficial].
        ii. Identify the exit face as the face that contains the origin (the projection of $o$) in this projected coordinate system, excluding the entry face [S1, unofficial].
    b. **Identify Next Tetrahedron:** Determine the tetrahedron sharing the identified exit face [S1, unofficial].
    c. **Pruning:** If the current tetrahedron exhibits null contribution (e.g., empty space), skip it [S1, unofficial].
    d. **Segment Definition:** Define the segment $s(t)$ from the current entry point $in(t)$ to the exit point $out(t)$ [S1, unofficial].
3. **Sampling and Feature Extraction:**
    a. Sample the midpoint $p(t) = \frac{in(t) + out(t)}{2}$ [S1, unofficial].
    b. Query color networks $g_{geo}$ and $g_{fine}$ at $p(t)$ to obtain color estimates [S1, unofficial].
    c. Compute SDF values at $in(t)$ and $out(t)$ by linearly interpolating the SDF values of the three vertices of the respective entry and exit faces [S1, unofficial].
    d. Compute normal vectors at $in(t)$ and $out(t)$ by interpolating vertex normals (or derived from SDF gradients) [S1, unofficial].
4. **Transmittance and Color Accumulation:**
    a. Compute transmittance $\alpha_t$ using the SDF values and a scale factor $\beta$ [S1, unofficial]:
       $$
       \alpha_t = \text{clip}\left( \frac{1 + e^{-\beta \cdot sdf(in(t))}}{1 + e^{-\beta \cdot sdf(out(t))}}, 0, 1 \right)
       $$
       where $sdf(in(t))$ is the SDF at the entry point and $sdf(out(t))$ is the SDF at the exit point [S1, unofficial].
    b. Accumulate color using the standard volume rendering equation with weights $\omega(t) = \alpha(t)T(t)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedral_ray_march(ray_origin, ray_direction, mesh, sdf_values, color_networks, beta, max_segments=1024):
    # 1. Find initial tetrahedron containing ray_origin
    current_tet = find_tetrahedron_containing_point(mesh, ray_origin)
    
    visited_segments = []
    
    for t in range(max_segments):
        if current_tet is None:
            break
            
        # 2. Find exit face
        # Project vertices of current_tet onto plane at ray_origin with normal ray_direction
        projected_vertices = []
        for vertex in current_tet.vertices:
            # Vector from origin to vertex
            vec = vertex - ray_origin
            # Project onto plane: p = v - (v . n) * n
            proj = vec - dot(vec, ray_direction) * ray_direction
            projected_vertices.append(proj)
            
        # Identify exit face: the face containing the origin (0,0) in projected coords
        # Exclude the entry face (the face containing the previous entry point)
        exit_face = find_face_containing_origin(projected_vertices, current_tet.faces)
        
        if exit_face is None:
            break # Ray exited mesh
            
        # 3. Define segment
        entry_point = current_tet.entry_point if t == 0 else visited_segments[-1].exit_point
        exit_point = interpolate_face_sdf(exit_face, sdf_values) # Linear interp of 3 vertices
        
        # Pruning check (simplified)
        if is_null_contribution(current_tet):
            current_tet = get_next_tetrahedron(current_tet, exit_face)
            continue
            
        # 4. Sample and Compute Features
        mid_point = (entry_point + exit_point) / 2.0
        
        # Query color networks
        c_geo = color_networks['geo'](mid_point)
        c_fine = color_networks['fine'](mid_point)
        
        # Compute SDF at entry and exit
        sdf_in = interpolate_face_sdf(current_tet.faces[entry_face_index], sdf_values)
        sdf_out = interpolate_face_sdf(exit_face, sdf_values)
        
        # Compute normals (simplified: assume precomputed or derived from SDF)
        n_in = compute_normal(entry_point, current_tet)
        n_out = compute_normal(exit_point, exit_face)
        
        # 5. Compute Transmittance
        # alpha_t = clip((1 + exp(-beta * sdf_in)) / (1 + exp(-beta * sdf_out)), 0, 1)
        exp_in = exp(-beta * sdf_in)
        exp_out = exp(-beta * sdf_out)
        alpha_t = min(max((1 + exp_in) / (1 + exp_out), 0.0), 1.0)
        
        visited_segments.append({
            'entry': entry_point,
            'exit': exit_point,
            'mid': mid_point,
            'alpha': alpha_t,
            'c_geo': c_geo,
            'c_fine': c_fine,
            'sdf_in': sdf_in,
            'sdf_out': sdf_out
        })
        
        # Move to next tetrahedron
        current_tet = get_next_tetrahedron(current_tet, exit_face)
        
    return visited_segments
```

**Parameters:**
- $\beta$: Scale factor for SDF-based transmittance, gradually increased during optimization [S1, unofficial].
- $\lambda$: Weight factor for loss function, 1 at coarse stage, 0.5 later [S1, unofficial].
- $\epsilon$: Small value for loss function stability, 0.1 [S1, unofficial].
- Max segments per ray: 1024 [S1, unofficial].

**Pitfalls:**
- When $\beta$ is large, long segments crossing the surface may generate too small gradients due to large SDF differences; these segments should be subdivided [S1, unofficial].
- Precision issues in projecting vertices to the plane if the ray direction is not normalized or if the camera center is far from the mesh [G].

## Subtopics

### SDF Field Regularization
Ensures smooth gradients in the discrete SDF field by solving a linear system for spatial gradients within each tetrahedron [S1, unofficial].

### CVT Optimization
Uses Centroidal Voronoi Tessellation to optimize site locations for regular tetrahedral discretization [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of tetrahedra intersected by the ray. With pruning, it avoids empty space. Max segments limit caps complexity [S1, unofficial].
- **Memory Complexity:** Proportional to the number of vertices and tetrahedra in the mesh, plus storage for SDF values and network parameters [S1, unofficial].
- **Strengths:** Efficient for non-uniform meshes; explicit geometric traversal allows for precise surface representation via SDF [S1, unofficial].
- **Limitations:** Requires complex traversal logic compared to voxel grids; sensitive to $\beta$ scaling [S1, unofficial].
- **When to use:** When rendering differentiable 3D models with explicit surface geometry represented by tetrahedral meshes [S1, unofficial].

## Background
Originating from the "VortSDF" method presented at WACV 2025, this technique adapts ray marching for non-uniform tetrahedral grids generated via Centroidal Voronoi Tessellation (CVT) [S1, unofficial]. It builds on earlier tetrahedral structures proposed for fast ray marching [S1, unofficial].

## Sources
[S1] WACV 2025 Open Access Repository, https://openaccess.thecvf.com/content/WACV2025/html/Thomas_VortSDF_3D_Modeling_with_Centroidal_Voronoi_Tesselation_on_Signed_Distance_WACV_2025_paper.html, fetched 2026-10-30 (unofficial)
