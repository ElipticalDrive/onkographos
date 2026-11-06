# Mesh Culling via 3D Projection and Connected Components

Also called: 2D Gaussian Splatting Mesh Post-Processing, Depth-Map Visibility Culling

## Overview
This method optimizes a discrete voxel-derived mesh by removing faces occluded from specific camera viewpoints and filtering out small geometric artifacts. It solves the problem of redundant geometry in surface-extracted meshes by projecting the mesh onto 2D depth maps to identify visible faces, then optionally grouping remaining triangles into connected components to discard noise. It differs from [Block Visibility Culling](5334-block-visibility-culling.md) by operating on individual triangle faces rather than volumetric blocks, and from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by using explicit depth-buffer occlusion tests rather than static coverage masks.

## How it works
The method takes a triangular mesh $M$ (vertices $V$, faces $F$) and a set of camera poses with intrinsic parameters as input, and outputs a reduced mesh $M'$.

**Data Structures:**
*   **Mesh**: Defined by vertex positions $v_i \in \mathbb{R}^3$ and face indices $f_j \in \mathbb{Z}^{3}$.
*   **Camera Parameters**: Intrinsic matrix $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$, extrinsic pose $T_{c2w}$, image dimensions $H \times W$, and far plane $d_{far}$ (typically 20.0).
*   **Depth Map**: A 2D array $D \in \mathbb{R}^{H \times W}$ storing the minimum depth of projected geometry.
*   **Face Mask**: A boolean array $m_F \in \{0,1\}^{|F|}$ indicating retained faces.

**Steps:**

1.  **Depth Map Extraction**: For each camera pose $k$, render the current mesh to generate a depth map $D_k$. This involves projecting all vertices $v_i$ to screen space, rasterizing the triangles, and storing the depth of the closest fragment at each pixel $(u,v)$.
2.  **Vertex Visibility Test**: For each vertex $v_i$ of the mesh, determine if it is visible from camera $k$. Project $v_i$ to screen coordinates $(u_i, v_i)$. If $(u_i, v_i)$ is within image bounds and the depth of $v_i$ in world space (transformed to camera space) is less than the value $D_k(u_i, v_i)$ (with a small epsilon for precision), the vertex is marked visible for that camera.
3.  **Face Visibility Mask**: A face $f_j = (v_a, v_b, v_c)$ is considered visible if **all** its vertices are visible across the set of cameras. In the reference implementation, visibility is aggregated such that a face is kept if it is visible in the primary view or all views, depending on the specific `point_masks` logic. Specifically, `face_mask = mask[mesh.faces].all(axis=1)` implies a face is retained only if all its constituent vertices are marked visible.
4.  **Mesh Update**: Apply the face mask to remove invisible faces. Remove any vertices that are no longer referenced by any remaining face to clean up the mesh topology.
5.  **Connected Component Filtering (Optional)**: If a forecast radius or cluster filtering is enabled:
    *   Identify connected components of triangles in the mesh.
    *   Sort components by triangle count.
    *   Retain the top $N$ largest components (where $N$ is `cluster_to_keep`) and discard smaller ones.
    *   Alternatively, discard any component with fewer than `min_cluster_size` triangles.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_mesh_by_projection(mesh, cameras, params):
    """
    mesh: object with .vertices (N,3) and .faces (M,3)
    cameras: list of dicts with 'K', 'c2w', 'H', 'W', 'far'
    params: dict with 'points_batch_size', 'cluster_to_keep', 'min_cluster_size'
    """
    vertices = mesh.vertices
    faces = mesh.faces
    
    # Step 1 & 2: Compute visibility for each vertex across all cameras
    # Initialize visibility matrix: num_vertices x num_cameras
    vertex_visible = np.zeros((len(vertices), len(cameras)), dtype=bool)
    
    for cam_idx, cam in enumerate(cameras):
        # Project vertices to camera space
        # v_cam = (c2w^-1) * v_world
        v_cam = transform_to_camera_space(vertices, cam['c2w'])
        
        # Project to screen space
        # u = fx * X/Z + cx, v = fy * Y/Z + cy
        u = cam['K'][0,0] * v_cam[:,0] / v_cam[:,2] + cam['K'][0,2]
        v = cam['K'][1,1] * v_cam[:,1] / v_cam[:,2] + cam['K'][1,2]
        z = v_cam[:,2]
        
        # Generate depth map for this camera (simplified: assume we have a function for this)
        depth_map = render_depth_map(mesh, cam)
        
        # Check visibility for each vertex
        for i in range(len(vertices)):
            if 0 <= u[i] < cam['W'] and 0 <= v[i] < cam['H']:
                # Check if vertex depth is consistent with depth map (front-most)
                # Note: In practice, this requires precise rasterization or depth buffer lookup
                if z[i] <= depth_map[int(v[i]), int(u[i])] + 1e-6:
                    vertex_visible[i, cam_idx] = True
                    
    # Step 3: Face Mask
    # A face is kept if all its vertices are visible in the relevant view(s)
    # The source uses .all(axis=1) on the face indices mapped to vertex visibility
    # Assuming we want faces visible in ALL cameras for robustness, or specific logic:
    face_mask = vertex_visible[faces].all(axis=1)
    
    # Step 4: Update Mesh
    new_faces = faces[face_mask]
    mesh.vertices = vertices
    mesh.faces = new_faces
    mesh.remove_unreferenced_vertices()
    
    # Step 5: Connected Component Filtering (Optional)
    if params.get('cluster_to_keep', 0) > 0 or params.get('min_cluster_size', 0) > 0:
        mesh = filter_connected_components(mesh, params)
        
    return mesh

def filter_connected_components(mesh, params):
    # Identify connected triangle clusters
    # Using a graph traversal on the mesh adjacency
    clusters = find_connected_triangles(mesh)
    
    # Sort clusters by size
    cluster_sizes = [len(c) for c in clusters]
    sorted_indices = np.argsort(cluster_sizes)[::-1]
    
    # Determine threshold
    max_clusters = params.get('cluster_to_keep', 1000)
    min_size = params.get('min_cluster_size', 50)
    
    # Select clusters to keep
    keep_mask = np.zeros(len(mesh.faces), dtype=bool)
    
    for idx in sorted_indices[:max_clusters]:
        cluster = clusters[idx]
        if len(cluster) >= min_size:
            for face_idx in cluster:
                keep_mask[face_idx] = True
                
    # Remove triangles not in keep_mask
    mesh.remove_triangles_by_mask(~keep_mask)
    mesh.remove_unreferenced_vertices()
    mesh.remove_degenerate_triangles()
    
    return mesh
```

**Parameters:**
*   `points_batch_size`: Batch size for processing vertices during visibility tests (default 500,000) [S1, unofficial].
*   `cluster_to_keep`: Maximum number of largest connected clusters to retain (default 1000) [S1, unofficial].
*   `min_cluster_size`: Minimum number of triangles a cluster must have to be kept (default 50) [S1, unofficial].

**Edge Cases:**
*   **Precision**: Depth comparison requires an epsilon to avoid z-fighting artifacts where vertices are exactly on the depth buffer boundary [G].
*   **Degenerate Input**: Vertices with $Z \le 0$ (behind camera) must be filtered before projection to avoid division by zero or invalid screen coordinates [G].

## Subtopics

### Mesh Post-Processing (Cluster Filtering)
This variant focuses on the topological cleanup of the mesh after visibility culling. It identifies connected components of triangles and removes small, isolated clusters that likely represent noise or artifacts rather than meaningful geometry. It is chosen when the mesh contains significant floating-point noise or disconnected fragments. It differs from simple visibility culling by operating on graph connectivity rather than camera projection.

## Operating characteristics
*   **Time Complexity**: $O(V \cdot C)$ for visibility testing, where $V$ is the number of vertices and $C$ is the number of cameras. Connected component analysis is $O(V + F)$ where $F$ is the number of faces.
*   **Memory Complexity**: $O(V \cdot C)$ for storing visibility masks, plus $O(F)$ for the mesh.
*   **Strengths**: Effectively removes occluded geometry, reducing rendering load. Cluster filtering removes noise.
*   **Limitations**: Requires multiple camera views. Computationally expensive for large meshes with many cameras.
*   **When to use**: When optimizing meshes for rendering from known viewpoints.
*   **When not to use**: When a view-independent representation is required.

## Background
This method is part of the post-processing pipeline for 2D Gaussian Splatting, specifically described in the context of the "2D Gaussian Splatting for Geometrically Accurate Radiance Fields" paper [S1, unofficial]. It was introduced to clean up the mesh extracted from the 2D Gaussians, ensuring that only visible and significant geometry is retained.

## Sources
[S1] 2D Gaussian Splatting for Geometrically Accurate Radiance Fields, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
