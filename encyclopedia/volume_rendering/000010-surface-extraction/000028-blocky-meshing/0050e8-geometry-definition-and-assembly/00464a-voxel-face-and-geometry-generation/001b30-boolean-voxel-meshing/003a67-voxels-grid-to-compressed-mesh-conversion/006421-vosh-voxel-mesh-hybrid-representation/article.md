# Vosh (Voxel-Mesh Hybrid Representation)
Also called: Voxel-Mesh Hybrid Representation

## Overview
Vosh is a hybrid scene representation for NeRF-based real-time view synthesis that combines a sparse voxel grid with a compressed mesh to balance rendering quality and computational speed [S1, unofficial]. It differs from pure voxel methods by extracting a mesh for high-frequency geometry and from pure mesh methods by retaining voxels for volumetric details, using differentiable rendering to optimize the mesh geometry and topology [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained view-dependent MLP (inherited from SNeRG/MERF) and camera poses/images from the Mip-NeRF 360 dataset [S1, unofficial].
*   **Output:** A hybrid representation consisting of a sparse voxel grid (occupancy grid) and a triangle mesh, along with associated MLP weights for color/density [S1, unofficial].

**Data Structures:**
1.  **Sparse Voxel Grid:** Uses an occupying grid approach similar to SNeRG [S1, unofficial]. Stores "alive" voxels.
2.  **Mesh:** Stores vertices, faces, and normal vectors. The mesh is optimized based on rendering quality and normal angle consistency [S1, unofficial].
3.  **MLP:** A view-dependent Multi-Layer Perceptron storing color and density information, identical to the architecture in SNeRG and MERF [S1, unofficial].

**Parameters:**
*   `premove`: Threshold for meshing based on rendering quality difference.
    *   Default (Vosh-Base): 0.9 [S1, unofficial].
    *   Tested values: 0.5, 1.0 [S1, unofficial].
    *   Effect: `premove=1.0` meshes the entire density field regardless of quality, yielding small improvements. `premove=0.5` selects specific regions for meshing to combine voxel and mesh advantages [S1, unofficial].
*   `λvoxel`: Weight of the voxel pruning loss.
    *   Default (Vosh-Base): 0.001 [S1, unofficial].
    *   Tested values: 0, 0.01 [S1, unofficial].
    *   Effect: Increasing `λvoxel` compresses the number of alive voxels, providing speed benefits at the expense of rendering quality [S1, unofficial].
*   `rmesh`: Resolution for the mesh occupancy grid used to check/align mesh and voxels.
    *   Default (Vosh-Base): Off (not used) [S1, unofficial].
    *   Tested values: 128, 256, 512 [S1, unofficial].
    *   Effect: Reducing `rmesh` aids in compressing alive voxels and provides speed benefits at the expense of rendering quality [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Start with a view-dependent MLP and a sparse voxel grid (occupying grid) [S1, unofficial].
2.  **Meshing Strategy:**
    *   Evaluate rendering quality differences across the scene [S1, unofficial].
    *   Identify regions where the density field should be meshed based on the `premove` threshold [S1, unofficial].
    *   Extract an initial mesh from the density field in these regions [S1, unofficial].
3.  **Mesh Optimization:**
    *   Optimize the mesh geometry using differentiable rendering [S1, unofficial].
    *   **Normal Angle Constraint:** Unlike NeRF2Mesh, Vosh explicitly considers the angle of normals between adjacent triangles during optimization to improve rendering quality while reducing triangle count [S1, unofficial].
    *   **Subdivision and Simplification:** Apply mesh subdivision and simplification based on rendering quality and normal angle change to significantly improve quality while decreasing the number of triangles [S1, unofficial].
4.  **Voxel Pruning:**
    *   Apply voxel pruning loss weighted by `λvoxel` to compress the number of alive voxels [S1, unofficial].
    *   Use the mesh occupancy grid (if `rmesh` is set) to check consistency between the mesh and voxel components [S1, unofficial].
5.  **Final Hybrid Representation:** Combine the optimized mesh and the pruned voxel grid. The MLP weights are shared or adapted for both components [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vosh_optimize(mlp, voxel_grid, images, poses, premove=0.9, lambda_voxel=0.001, rmesh=0):
    """
    Optimizes the Vosh hybrid representation.
    """
    # Step 1: Identify meshing regions based on rendering quality
    # Calculate rendering quality metric (e.g., PSNR or error) for current voxel grid
    quality_map = compute_rendering_quality(mlp, voxel_grid, images, poses)
    
    # Threshold regions for meshing
    mesh_regions = quality_map < premove
    
    # Step 2: Extract initial mesh from density field in mesh_regions
    # This involves marching cubes or similar extraction on the density field
    # restricted to mesh_regions
    mesh = extract_mesh_from_density(mlp, voxel_grid, mesh_regions)
    
    # Step 3: Optimize mesh geometry
    # Differentiable rendering loop
    for iteration in range(num_iterations):
        # Render image from mesh
        mesh_image = render_mesh(mesh, poses)
        
        # Compute loss against ground truth
        loss_render = compute_rendering_loss(mesh_image, images)
        
        # Compute normal angle consistency loss
        # This is the key differentiator from NeRF2Mesh
        normal_angles = compute_adjacent_triangle_normals(mesh)
        loss_normals = compute_normal_angle_consistency(normal_angles)
        
        # Total mesh loss
        loss_mesh = loss_render + loss_normals
        
        # Update mesh vertices/positions
        mesh = optimize_mesh_geometry(mesh, loss_mesh)
        
        # Apply subdivision/simplification based on quality and normal changes
        if should_subdivide(mesh, loss_render, normal_angles):
            mesh = subdivide_mesh(mesh)
        if should_simplify(mesh, loss_render, normal_angles):
            mesh = simplify_mesh(mesh)
            
    # Step 4: Voxel Pruning
    # Compute voxel pruning loss
    loss_voxel = compute_voxel_pruning_loss(voxel_grid)
    
    # Update voxel grid with pruning
    voxel_grid = prune_voxels(voxel_grid, lambda_voxel * loss_voxel)
    
    # Step 5: Mesh Occupancy Grid Check (if rmesh > 0)
    if rmesh > 0:
        occupancy_grid = create_occupancy_grid(mesh, resolution=rmesh)
        # Align voxel grid with occupancy grid
        voxel_grid = align_voxels_with_occupancy(voxel_grid, occupancy_grid)
        
    return mesh, voxel_grid, mlp
```

## Operating characteristics
*   **Time and Memory Complexity:** Vosh offers a balance between quality and speed. It is more memory-efficient than pure voxel methods like 3DGS but less efficient than pure mesh methods like MERF in terms of disk storage [S1, unofficial].
*   **Measured Performance:**
    *   **VRAM (MB):** Vosh-Base: 506, Vosh-Light: 490. (Comparison: SNeRG++ 340, MERF 360, NeRF2Mesh 426, MobileNeRF 927, BakedSDF 953, 3DGS 3701) [S1, unofficial].
    *   **Disk (MB):** Vosh-Base: 488, Vosh-Light: 232. (Comparison: SNeRG++ 624, MERF 149, NeRF2Mesh 123, MobileNeRF 345, BakedSDF 807, 3DGS 1270) [S1, unofficial].
    *   **Quality:** Vosh-Base achieves PSNR 24.18, SSIM 0.623, LPIPS 0.364, and 1.41M faces/57438 alive voxels [S1, unofficial]. Vosh-Light achieves PSNR 23.74, SSIM 0.602, LPIPS 0.377, and 2489 faces/232 alive voxels [S1, unofficial].
*   **Scaling:** Reducing `rmesh` resolution and increasing `λvoxel` improves speed and reduces memory at the cost of quality [S1, unofficial].
*   **Strengths:** Controllable balance between rendering quality and speed; real-time view synthesis on mobile devices; improved mesh quality via normal angle optimization [S1, unofficial].
*   **Limitations:** Cannot model view-dependent colors for translucent objects; cannot represent extremely large scenes or objects with complex reflections; limitations of the underlying view-dependent MLP may degrade mesh optimization quality [S1, unofficial].
*   **When to use:** Real-time view synthesis on mobile devices where memory and speed are critical but quality must remain high [S1, unofficial].
*   **When not to use:** Scenes with translucent objects, extremely large scales, or complex reflections [S1, unofficial].

## Background
Vosh was introduced in the paper "Voxel-Mesh Hybrid Representation for Real-Time View Synthesis" [S1, unofficial]. It builds on the sparse voxel representation of SNeRG [S1, unofficial] and the mesh extraction techniques of MobileNeRF and NeRF2Mesh [S1, unofficial]. It was evaluated on outdoor scenes from the Mip-NeRF 360 dataset [S1, unofficial].

## Sources
[S1] Voxel-Mesh Hybrid Representation for Real-Time View Synthesis, https://doi.org/10.48550/arxiv.2403.06505, fetched 2026-10-26 (unofficial)
