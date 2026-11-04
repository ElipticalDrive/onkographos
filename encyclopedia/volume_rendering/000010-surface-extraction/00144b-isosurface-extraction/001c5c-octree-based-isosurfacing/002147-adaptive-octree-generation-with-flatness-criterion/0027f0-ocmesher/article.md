# OcMesher

Also called: OcMesher Octree Construction and Mesh Extraction, View-Dependent Octree-based Mesh Extraction

## Overview
OcMesher is a method for extracting memory-efficient, highly detailed triangle meshes from unbounded scenes represented by Signed Distance Functions (SDFs) [S1, unofficial]. It uses a view-dependent adaptive octree structure driven by a given set of camera views to refine geometry only where visible [S1, unofficial]. Unlike parent methods that generate octrees based on surface flatness alone, OcMesher incorporates camera projection metrics to prioritize resolution based on projected pixel density [S1, unofficial].

## How it works
**Inputs:**
- `cameras`: A tuple `(cam_poses, Ks, Hs, Ws)` where `cam_poses` are $N \times 4 \times 4$ world-to-camera matrices, `Ks` are $N \times 3 \times 3$ intrinsic matrices, and `Hs`, `Ws` are heights and widths [S1, unofficial].
- `bounds`: Axis-aligned bounding box limits `[min_x, max_x, min_y, max_y, min_z, max_z]` [S1, unofficial].
- `kernels`: A list of callable functions $K_i(\mathbf{p})$ that return the SDF value at point $\mathbf{p}$ [S1, unofficial].
- Parameters: `pixels_per_cube` (default 8), `inv_scale` (default 10), `min_dist` (default 1), `memory_limit_mb` (default 1000), `bisection_iters` (default 15), `enclosed` (default True), `simplify_occluded` (default True), `visible_relax_iter` (default 2), `coarse_count` (default 500000) [S1, unofficial].

**Outputs:**
- A list of `trimesh.Trimesh` objects containing vertices, faces, and visibility tags [S1, unofficial].

**Data Structures:**
- `struct node`: Contains `cube c` and `int nxts[8]`. `nxts[i]` is the index of the $i$-th child in the `nodes` vector, or `-1` if absent. Leaf nodes have `nxts[0] = -2` [S1, unofficial].
- `struct cube`: Contains `int coords[3]` and `int L`. `L` is the level. Special values: `L >= 0` (regular), `L == -1` (boundary), `L == -2` (exterior) [S1, unofficial].
- `struct computed_vertex`: Contains `T c[3]` (coordinates), `l`, `r` (SDF values) [S1, unofficial].
- Camera data is flattened into a numpy array of shape `(23 * n_cameras)` [S1, unofficial]. Each camera entry stores: inverse pose (first 12 values, $3 \times 4$), intrinsic matrix $K$ (9 values), height $H$ (1 value), width $W$ (1 value) [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   Compute scene center $\mathbf{c}_{center}$ as the midpoint of `bounds`. Compute size $S = 1.1 \times \max(\text{bounds dimensions})$ [S1, unofficial]. Flatten camera data into the `cameras` array [S1, unofficial].

2. **Coarse Visibility:**
   Call `run_coarse` with center, size, camera count, camera data, `pixels_per_cube`, `inv_scale`, `min_dist`, `coarse_count`, `memory_limit_mb`, and number of SDF kernels [S1, unofficial].
   - Initialize root node at `coords=[0,0,0]`, `L=0` [S1, unofficial].
   - Push root to priority queue ordered by `projected_size` [S1, unofficial].
   - While queue not empty and node count < `coarse_count`:
     - Pop node with highest `projected_size` [S1, unofficial].
     - If `projected_size < inv_scale`, break [S1, unofficial].
     - Expand node fully using `expand_octree` [S1, unofficial].
     - Push 8 children to queue [S1, unofficial].
   - Mark remaining nodes as grid nodes and return `n_blocks` [S1, unofficial].

3. **Coarse Refinement (SDF-aware):**
   Loop while `fine_group()` returns `inc > 0`:
   - `fine_group()` batches nodes into memory-limited chunks [S1, unofficial].
   - Call `fine_iteration()` with SDF pointer [S1, unofficial].
   - While `n > 0`:
     - Get positions via `fine_iteration_output()` [S1, unofficial].
     - Compute SDFs: $\text{sdf} = \min_{k} K_k(\text{positions})$ [S1, unofficial].
     - Pass SDFs to `fine_iteration()` [S1, unofficial].
   - `fine_iteration` identifies bipolar edges (SDF sign changes) and expands nodes containing them [S1, unofficial].

4. **Visibility Filtering:**
   Call `vis_filter(simplify_occluded, visible_relax_iter)` to return `n_vis_block` [S1, unofficial]. This removes blocks occluded by the mesh geometry [S1, unofficial].

5. **Fine Mesh Construction:**
   Initialize `nv` array [S1, unofficial].
   - **Visible Blocks:**
     - While `final_iteration()` returns `n > 0`:
       - Get positions via `final_iteration2()` [S1, unofficial].
       - Compute SDFs using `kernel_caller` [S1, unofficial].
       - Call `final_iteration3()` with SDFs [S1, unofficial].
   - **Occluded Blocks:**
     - Call `final_iteration_occluded()` [S1, unofficial].
     - If `n != 0`, get positions, compute SDFs, call `final_iteration3_occluded()` [S1, unofficial].
   - **Remaining:**
     - Reset `nv` to zeros [S1, unofficial].
     - Call `final_remaining()` [S1, unofficial].

6. **Per-Element Mesh Extraction:**
   For each element $e$:
   - Get vertex centers via `get_verts_center(e, centers)` [S1, unofficial].
   - Compute center SDFs: `center_sdf = kernel_caller(kernels[e], centers)` [S1, unofficial].
   - Initialize cubes array [S1, unofficial].
   - Call `update_verts(e, empty_sdf, empty_sdf, cubes)` [S1, unofficial].
   - **Bisection Refinement:**
     - Repeat `bisection_iters` times:
       - Compute SDFs for cubes: `sdf = kernel_caller(kernels[e], cubes)` [S1, unofficial].
       - Call `update_verts(e, sdf, center_sdf, cubes)` [S1, unofficial].
   - Get left/right vertices via `get_lr_verts(e, cubes, cubes_r)` [S1, unofficial].
   - Compute SDFs for left/right: `sdf_l`, `sdf_r` [S1, unofficial].
   - Call `finalize_verts(e, sdf_l, sdf_r, vertices)` [S1, unofficial].
   - Construct faces via `construct_faces(e, vertices, cnts)` returning `nve`, `nvf`, `nf` [S1, unofficial].

7. **Edge and Face Vertex Refinement:**
   - Get extra vertex centers: `edge_vertices_c`, `face_vertices_c` via `get_extra_verts_center` [S1, unofficial].
   - Compute their SDFs: `ecenter_sdf`, `fcenter_sdf` [S1, unofficial].
   - Initialize left/right extra vertices arrays [S1, unofficial].
   - Call `update_extra_verts` with empty SDFs [S1, unofficial].
   - **Bisection Refinement:**
     - Repeat `bisection_iters` times:
       - Compute SDFs for left/right edge/face vertices [S1, unofficial].
       - Call `update_extra_verts` with computed SDFs [S1, unofficial].
   - Get refined left/right extra vertices via `get_lr_extra_verts` [S1, unofficial].
   - Compute SDFs for refined vertices [S1, unofficial].
   - Call `finalize_extra_verts` to produce final `edge_vertices` and `face_vertices` [S1, unofficial].

8. **Final Assembly:**
   - Concatenate `vertices`, `edge_vertices`, `face_vertices` [S1, unofficial].
   - Get visibility tags via `get_in_view_tag` [S1, unofficial].
   - Create `trimesh.Trimesh` objects [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ocmesher(cameras, bounds, kernels, 
             pixels_per_cube=8, inv_scale=10, min_dist=1, 
             memory_limit_mb=1000, bisection_iters=15, 
             enclosed=True, simplify_occluded=True, 
             visible_relax_iter=2, coarse_count=500000):
    # 1. Initialization
    cam_poses, Ks, Hs, Ws = cameras
    n_cameras = len(cam_poses)
    
    # Flatten cameras: [inv_pose(12), K(9), H(1), W(1)] per camera
    cameras_flat = np.zeros(23 * n_cameras, dtype=np.float64)
    for i in range(n_cameras):
        inv_pose = np.linalg.inv(cam_poses[i])[:3, :4].reshape(-1)
        K = Ks[i].reshape(-1)
        cameras_flat[23*i : 23*(i+1)] = np.concatenate([inv_pose, K, [Hs[i]], [Ws[i]]])
        
    center = (bounds[0::2] + bounds[1::2]) / 2.0
    size = 1.1 * np.max(bounds[1::2] - bounds[0::2])
    
    # 2. Coarse Visibility
    # Note: run_coarse is a C++ function call in implementation
    n_blocks = run_coarse(
        center, size, n_cameras, cameras_flat,
        pixels_per_cube, inv_scale, min_dist,
        coarse_count, memory_limit_mb, len(kernels)
    )
    
    # 3. Coarse Refinement
    while True:
        inc = fine_group()
        if inc == 0: break
        n = fine_iteration(np.zeros(0, dtype=np.float32))
        while n > 0:
            positions = np.zeros((n, 3), dtype=np.float64)
            fine_iteration_output(positions)
            # Compute min SDF across all kernels
            sdf_batch = kernel_caller(kernels, positions)
            sdf_min = np.min(sdf_batch, axis=-1)
            n = fine_iteration(sdf_min.astype(np.float32))
            
    # 4. Visibility Filtering
    n_vis_block = vis_filter(simplify_occluded, visible_relax_iter)
    
    # 5. Fine Mesh Construction
    nv = np.zeros(1, dtype=np.int32)
    
    # Visible
    while True:
        n = final_iteration(nv)
        if n == 0: break
        positions = np.zeros((n, 3), dtype=np.float64)
        final_iteration2(positions)
        sdf = kernel_caller(kernels, positions)
        inc = final_iteration3(sdf.astype(np.float32))
        
    # Occluded
    n = final_iteration_occluded(nv)
    if n != 0:
        positions = np.zeros((n, 3), dtype=np.float64)
        final_iteration2(positions)
        sdf = kernel_caller(kernels, positions)
        final_iteration3_occluded(sdf.astype(np.float32))
        
    # Remaining
    nv = np.zeros(len(kernels), dtype=np.int32)
    final_remaining(nv)
    
    # 6. Per-Element Mesh Extraction
    meshes = []
    in_view_tags = []
    
    for e in range(len(kernels)):
        k_e = [kernels[e]]
        centers = np.zeros((nv[e], 3), dtype=np.float64)
        get_verts_center(e, centers)
        center_sdf = kernel_caller(k_e, centers)
        
        cubes = np.zeros((nv[e] * 8, 3), dtype=np.float64)
        update_verts(e, np.zeros(0, dtype=np.float32), np.zeros(0, dtype=np.float32), cubes)
        
        for _ in range(bisection_iters):
            sdf = kernel_caller(k_e, cubes)
            update_verts(e, sdf.astype(np.float32), center_sdf.astype(np.float32), cubes)
            
        cubes_r = np.zeros_like(cubes)
        get_lr_verts(e, cubes, cubes_r)
        sdf_l = kernel_caller(k_e, cubes)
        sdf_r = kernel_caller(k_e, cubes_r)
        
        vertices = np.zeros((0, 3), dtype=np.float64)
        finalize_verts(e, sdf_l.astype(np.float32), sdf_r.astype(np.float32), vertices)
        
        faces = np.zeros((0, 3), dtype=np.int32)
        nve, nvf, nf = construct_faces(e, vertices, faces)
        
        # 7. Edge/Face Refinement
        edge_vertices_c = np.zeros((nve, 3), dtype=np.float64)
        face_vertices_c = np.zeros((nvf, 3), dtype=np.float64)
        get_extra_verts_center(edge_vertices_c, face_vertices_c)
        
        ecenter_sdf = kernel_caller(k_e, edge_vertices_c)
        fcenter_sdf = kernel_caller(k_e, face_vertices_c)
        
        edge_vertices_lr = np.zeros((nve * 2, 3), dtype=np.float64)
        face_vertices_lr = np.zeros((nvf * 4, 3), dtype=np.float64)
        update_extra_verts(
            np.zeros(0, dtype=np.float32), np.zeros(0, dtype=np.float32),
            np.zeros(0, dtype=np.float32), np.zeros(0, dtype=np.float32),
            edge_vertices_lr, face_vertices_lr
        )
        
        for _ in range(bisection_iters):
            e_sdf = kernel_caller(k_e, edge_vertices_lr)
            f_sdf = kernel_caller(k_e, face_vertices_lr)
            update_extra_verts(
                e_sdf.astype(np.float32), f_sdf.astype(np.float32),
                ecenter_sdf.astype(np.float32), fcenter_sdf.astype(np.float32),
                edge_vertices_lr, face_vertices_lr
            )
            
        edge_vertices_r = np.zeros((nve * 2, 3), dtype=np.float64)
        face_vertices_r = np.zeros((nvf * 4, 3), dtype=np.float64)
        get_lr_extra_verts(edge_vertices_lr, edge_vertices_r, face_vertices_lr, face_vertices_r)
        
        esdf_l = kernel_caller(k_e, edge_vertices_lr)
        esdf_r = kernel_caller(k_e, edge_vertices_r)
        fsdf_l = kernel_caller(k_e, face_vertices_lr)
        fsdf_r = kernel_caller(k_e, face_vertices_r)
        
        final_edge_vertices = np.zeros((nve, 3), dtype=np.float64)
        final_face_vertices = np.zeros((nvf, 3), dtype=np.float64)
        finalize_extra_verts(
            esdf_l.astype(np.float32), esdf_r.astype(np.float32), final_edge_vertices,
            fsdf_l.astype(np.float32), fsdf_r.astype(np.float32), final_face_vertices
        )
        
        # 8. Assembly
        all_vertices = np.concatenate([vertices, final_edge_vertices, final_face_vertices])
        in_view_tag = np.zeros(all_vertices.shape[0], dtype=bool)
        get_in_view_tag(e, in_view_tag)
        in_view_tags.append(in_view_tag)
        
        meshes.append(trimesh.Trimesh(vertices=all_vertices, faces=faces, process=False))
        
    return meshes, in_view_tags

def kernel_caller(kernels, XYZ_all, enclosed=True, bounds=None):
    if len(XYZ_all) == 0:
        return np.zeros((0, len(kernels)), dtype=np.float32)
    
    step = 10000000
    sdfs = []
    
    for i in range(0, len(XYZ_all), step):
        XYZ = XYZ_all[i:i+step]
        sdfs_i = []
        
        out_bound = np.zeros(len(XYZ), dtype=bool)
        if enclosed:
            for c in range(3):
                out_bound |= XYZ[:, c] <= bounds[c*2]
                out_bound |= XYZ[:, c] >= bounds[c*2+1]
                
        for kernel in kernels:
            sdf = kernel(XYZ)
            if enclosed:
                sdf[out_bound] = 1.0
            sdfs_i.append(sdf)
            
        sdfs.append(np.stack(sdfs_i, axis=-1).astype(np.float32))
        
    return np.concatenate(sdfs, axis=0)
```

**Subtopics:**
- [Octree Leaf Construction with QEF Surface Fitting](4268-octree-leaf-construction-with-qef-surface-fitting.md): Uses Quadratic Error Functions to fit surfaces at octree leaves. Choose when higher geometric accuracy per vertex is needed at the cost of increased computation per leaf.

## Operating characteristics
- **Time Complexity:** Dominated by SDF evaluations. The coarse step is $O(N \log N)$ where $N$ is the number of octree nodes, prioritized by projected size [S1, unofficial]. The fine step is proportional to the number of visible blocks and bisection iterations [S1, unofficial].
- **Memory Complexity:** Controlled by `memory_limit_mb` during coarse refinement and `coarse_count` for the initial octree size [S1, unofficial].
- **Performance:** Used in Infinigen for procedural synthetic data generation [S1, unofficial]. It is memory-efficient for unbounded scenes because it only generates geometry in view-dependent regions [S1, unofficial].
- **Strengths:** Handles unbounded scenes; view-dependent refinement reduces mesh size; handles multiple SDF kernels [S1, unofficial].
- **Limitations:** Requires camera poses and intrinsics; SDF evaluation can be slow for complex kernels; `enclosed` mode clamps out-of-bounds values which may affect boundary accuracy [S1, unofficial].

## Background
OcMesher was developed by Zeyu Ma at Princeton University [S1, unofficial]. It is used by default in the Infinigen project to speed up video generation and export terrain meshes [S1, unofficial]. The method was presented in the paper "View-Dependent Octree-based Mesh Extraction in Unbounded Scenes for Procedural Synthetic Data" (arXiv:2312.08364) [S1, unofficial].

## Sources
[S1] princeton-vl/OcMesher, https://github.com/princeton-vl/OcMesher, fetched 2026-10-26 (unofficial)
