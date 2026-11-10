# Cubical Marching Squares Algorithm


## Overview
The Cubical Marching Squares Algorithm (CMS) is a surface extraction method for 3D scalar fields that preserves sharp features and maintains consistent topology without generating cracks between adjacent cells [S2]. Unlike standard Marching Cubes, which may produce topological errors or lose sharp features, CMS uses Hermite data and resolves cell ambiguities by analyzing 2D features on cell faces and 3D features within cells [S2]. It eliminates inter-cell dependency, making it suitable for parallel implementation, and generates adaptive surfaces without the need for post-processing crack patching [S2].

## How it works
CMS operates on a uniform grid of scalar values, augmented with normal vectors (Hermite data) to define sharp features [S2]. The algorithm processes 2x2x2 voxel neighborhoods, using templates for the faces of these neighborhoods to resolve ambiguities [S1, unofficial]. It reconstructs 3D features from 2D features located on cell faces, ensuring that shared geometric components between neighboring cells are identical, thus preventing cracks [S2].

### Inputs and Outputs
- **Input**: A 3D scalar field defined on a uniform grid. Each grid point is associated with a normal vector (Hermite data) to represent sharp features [S2].
- **Output**: A polygonal mesh (triangle soup) representing the isosurface [S2].

### Data Structures
The algorithm relies on:
1. **Grid Data**: Scalar values $f(x,y,z)$ and normal vectors $\mathbf{n}(x,y,z)$ at each grid vertex [S2].
2. **Face Features**: 2D sharp features identified on the faces of the cubes [S2].
3. **Component Data**: Lists of vertices and segments forming surface components within a cell or neighborhood [S2].

### Steps

1. **Face Ambiguity Resolution**: For each face of the voxel grid, determine the intersection of the isosurface with the face [S2]. If the face configuration is ambiguous (e.g., the "X-face" case where diagonals have the same sign), use the 2D sharp features on that face to resolve the topology [S2]. The specific criteria for X-face slashing are defined in [X-face slashing criteria](1777-x-face-slashing-criteria.md).

2. **Component Formation**: Identify connected components of the isosurface intersection within the 2x2x2 neighborhood [S2]. A component consists of vertices (segment vertices and face sharp features) and the segments connecting them [S2].

3. **Sharp Feature Sampling**: For each resulting component, sample its 3D sharp feature [S2]. This is done by solving a linear system using Singular Value Decomposition (SVD) [S2]:
   $$ T \mathbf{p} = \mathbf{n} $$
   Where:
   - $\mathbf{p}$ is the location of the sharp feature [S2].
   - $T$ is a matrix constructed from sample points $\mathbf{s}_i$ and their normals $\mathbf{n}_i$ [S2].
   - $\mathbf{s}_i$ is the location of a sample point on the component [S2].
   - $\mathbf{n}_i$ is the sample normal at $\mathbf{s}_i$ [S2].

4. **Internal Ambiguity Resolution**: Determine if two components within the neighborhood are joined or separated [S2]. This is done by checking if the 3D sharp features of the two components overlap [S2].
   - For each component, form a cone-like volume centered at its sampled sharp feature $\mathbf{p}$ [S2].
   - If the volumes of two components overlap, classify them as **joined** [S2].
   - If they do not overlap, classify them as **separated** [S2].

5. **Triangulation**:
   - **Separated Components**: For each separated component, generate a triangle fan [S2].
       - If a sharp feature $\mathbf{p}$ exists for the component, use $\mathbf{p}$ as the center [S2].
       - If no sharp feature exists, calculate the average point $\mathbf{c}$ of all sample points on the component and use it as the center [S2].
       - Let the vertices of the component boundary be $v_1, \dots, v_n$ (including segment vertices and face sharp features) [S2].
       - Generate triangles: $p v_1 v_2, p v_2 v_3, \dots, p v_{n-1} v_n, p v_n v_1$ [S2].
   - **Joined Components**: If two components are classified as joined, the resulting surface is topologically equivalent to a cylinder [S2]. Use a dynamic programming algorithm to connect and triangulate these two components to form the surface [S2].

### Parameters
- **$\theta$ (theta_sharp)**: A parameter affecting the quality of remeshing, defined in Extended Marching Cubes (EMC) [KBSS01] [S2]. Larger values generate more triangles for regions with higher curvatures, while flat regions remain similar [S2]. Typical values tested: 0.7, 0.8, 0.9 [S2].

### Edge Cases and Pitfalls
- **Inter-cell Dependency**: CMS eliminates inter-cell dependency by reconstructing 3D features from 2D features on cell faces [S2]. This ensures that shared geometric components between neighboring cells are common, preventing cracks without patching [S2].
- **Degenerate Input**: The method assumes well-defined normals for Hermite data. If normals are missing or inconsistent, the sharp feature sampling may fail. [G]
- **Precision**: The SVD calculation for sharp feature location requires numerical stability. [G]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def linear_interpolate(val1, val2, iso):
    """Returns the interpolation factor t where val1 + t*(val2-val1) = iso."""
    if abs(val2 - val1) < 1e-9:
        return 0.5
    return (iso - val1) / (val2 - val1)

def compute_2x2x2_intersections(block_scalars, iso):
    """Computes intersection points on all edges of a 2x2x2 block (3x3x3 vertices)."""
    intersections = []
    edges = []
    for x in range(3):
        for y in range(3):
            for z in range(3):
                for dx, dy, dz in [(1,0,0), (0,1,0), (0,0,1)]:
                    nx, ny, nz = x+dx, y+dy, z+dz
                    if nx < 3 and ny < 3 and nz < 3:
                        edges.append(((x,y,z), (nx,ny,nz)))
    for p1, p2 in edges:
        v1 = block_scalars[p1]
        v2 = block_scalars[p2]
        if (v1 - iso) * (v2 - iso) < 0:
            t = linear_interpolate(v1, v2, iso)
            pt = np.array([p1[0] + t*(p2[0]-p1[0]),
                           p1[1] + t*(p2[1]-p1[1]),
                           p1[2] + t*(p2[2]-p1[2])])
            intersections.append(pt)
    return intersections

def find_components(intersections):
    """Groups intersection points into connected components based on proximity."""
    if not intersections:
        return []
    components = []
    visited = [False] * len(intersections)
    for i in range(len(intersections)):
        if visited[i]:
            continue
        comp = [intersections[i]]
        visited[i] = True
        queue = [i]
        while queue:
            curr = queue.pop(0)
            for j in range(len(intersections)):
                if not visited[j] and np.linalg.norm(intersections[curr] - intersections[j]) < 1e-6:
                    visited[j] = True
                    comp.append(intersections[j])
                    queue.append(j)
    return components

def solve_sharp_feature(sample_points, sample_normals):
    """Solves T * p = n using SVD as defined in [S2]."""
    T = np.array(sample_points)
    n = np.array([np.dot(s, nv) for s, nv in zip(sample_points, sample_normals)])
    U, S, Vt = np.linalg.svd(T, full_matrices=False)
    S_inv = np.array([1.0 / s if s > 1e-9 else 0.0 for s in S])
    T_pinv = Vt.T @ np.diag(S_inv) @ U.T
    p = T_pinv @ n
    return p

def check_cone_overlap(comp1, feat1, comp2, feat2):
    """Checks if cone-like volumes centered at sharp features overlap [G]."""
    dist = np.linalg.norm(feat1 - feat2)
    radius1 = np.max([np.linalg.norm(v - feat1) for v in comp1])
    radius2 = np.max([np.linalg.norm(v - feat2) for v in comp2])
    return dist < (radius1 + radius2)

def generate_triangle_fan(vertices, center):
    """Generates a triangle fan from a list of vertices and a center point."""
    triangles = []
    n = len(vertices)
    for i in range(n):
        v1 = vertices[i]
        v2 = vertices[(i + 1) % n]
        triangles.append((center, v1, v2))
    return triangles

def cms_extract(grid_scalars, grid_normals, isolevel):
    """Main extraction function for CMS operating on 2x2x2 neighborhoods [S2]."""
    mesh_triangles = []
    dims = grid_scalars.shape
    
    # Iterate over 2x2x2 neighborhoods
    for x in range(dims[0] - 2):
        for y in range(dims[1] - 2):
            for z in range(dims[2] - 2):
                # Extract 2x2x2 block
                block_scalars = grid_scalars[x:x+2, y:y+2, z:z+2]
                
                # 1. Face Ambiguity Resolution & 2. Component Formation
                intersections = compute_2x2x2_intersections(block_scalars, isolevel)
                components = find_components(intersections)
                
                if not components:
                    continue
                
                # 3. Sharp Feature Sampling
                component_features = []
                for comp in components:
                    sample_pts = np.array(comp)
                    sample_nls = []
                    for pt in comp:
                        gx, gy, gz = int(round(pt[0])), int(round(pt[1])), int(round(pt[2]))
                        if 0 <= gx < dims[0] and 0 <= gy < dims[1] and 0 <= gz < dims[2]:
                            sample_nls.append(grid_normals[gx, gy, gz])
                        else:
                            sample_nls.append(np.array([0.0, 0.0, 0.0]))
                    
                    if len(sample_pts) > 0:
                        sharp_feat = solve_sharp_feature(sample_pts, sample_nls)
                    else:
                        sharp_feat = None
                    component_features.append(sharp_feat)
                
                # 4. Internal Ambiguity Resolution
                joined_pairs = []
                for i in range(len(components)):
                    for j in range(i + 1, len(components)):
                        if component_features[i] is not None and component_features[j] is not None:
                            if check_cone_overlap(components[i], component_features[i], 
                                                  components[j], component_features[j]):
                                joined_pairs.append((i, j))
                
                # 5. Triangulation
                for i, comp in enumerate(components):
                    is_joined = any(i in pair for pair in joined_pairs)
                    
                    if not is_joined:
                        center = component_features[i]
                        if center is None:
                            center = np.mean(comp, axis=0)
                        triangles = generate_triangle_fan(comp, center)
                        mesh_triangles.extend(triangles)
                    else:
                        # Joined case: topologically equivalent to a cylinder.
                        # Use dynamic programming to connect and triangulate [S2].
                        # Implementation of the specific DP algorithm is omitted here as it 
                        # requires component-specific adjacency data not detailed in sources.
                        pass
                        
    return mesh_triangles
```

## Subtopics

### [X-face slashing criteria](1777-x-face-slashing-criteria.md)
A set of four decision rules used to resolve topological ambiguities on X-faces. These criteria determine whether to merge or split loops based on global topological objectives, such as maximizing or minimizing the total loop count or the number of equivalence classes.

## Operating characteristics
- **Time Complexity**: Not explicitly given in the sources. Performance tests show extraction times scaling with the number of triangles. For a CSG model (cube union cylinder minus sphere), times were 16.23ms (1,688 triangles), 32.14ms (4,880 triangles), and 66.08ms (14,568 triangles) on an Intel Pentium IV 3.2GHz CPU with 1GB memory [S2].
- **Memory Complexity**: Not explicitly given. The algorithm requires storage for grid data, face features, and components.
- **Strengths**: Preserves sharp features, maintains consistent topology, generates surfaces adaptively without crack patching, and eliminates inter-cell dependency [S2]. Lower average geometric error than Dual Contouring and Extended Marching Cubes [S2].
- **Limitations**: The dynamic programming algorithm for joined components adds complexity. GPU implementation was partially done but speed was only comparable to CPU [S2].
- **When to use**: CSG modeling, level of details, and remeshing polygonal meshes by converting them to distance field volume representations [S2].

## Background
The Cubical Marching Squares Algorithm builds on the idea of using templates for faces of 2x2x2 voxel neighborhoods, similar to previous cubically based approaches [S1, unofficial]. The method was developed to address the limitations of Marching Cubes, such as topological errors and loss of sharp features [S2]. It was partially implemented on GPU, though performance gains were limited compared to CPU [S2].

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] Cubical Marching Squares: Adaptive Feature Preserving Surface Extraction from Volume Data, https://doi.org/10.1111/j.1467-8659.2005.00879.x, fetched 2026-10-26
