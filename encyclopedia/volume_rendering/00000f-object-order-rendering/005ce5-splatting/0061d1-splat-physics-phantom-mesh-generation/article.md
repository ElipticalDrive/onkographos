# Splat Physics: Phantom Mesh Generation

Also called: Phantom Structure, Splat-to-Mesh

## Overview
Phantom Mesh Generation augments a visual splat representation (typically 3D Gaussians) with an underlying surface mesh to enable standard graphics engine physics simulations, such as collision detection and shadow casting, while relying solely on the splat for visual rendering [S1]. It solves the problem that splats lack explicit surface topology, which prohibits direct use of standard physics models [S1]. It is distinct from pure visual splatting methods by introducing a dual-representation system: the splat handles the framebuffer, and the "phantom" mesh handles the physics engine [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of splat primitives (e.g., Gaussian ellipsoids from a PLY file) and configuration parameters (algorithm selection, grid threshold, scale).
*   **Output:** A surface mesh (OBJ format) co-aligned with the splat, parented to the splat’s game object, and equipped with mesh colliders [S1].

**Data Structures:**
*   **Splat Grid:** A volumetric or grid-based representation constructed from the densities of the input Gaussian ellipsoids [S1].
*   **Isosurface:** A scalar field derived from the splat grid, representing the boundary of the object [S1].
*   **Mesh:** A triangle mesh containing vertices and faces, generated via Marching Cubes or Poisson reconstruction [S1].

**Steps:**
The method proceeds as follows [S1]:

1.  **Algorithm Selection:** The developer selects either **Object-Optimized** (Marching Cubes) or **Space-Optimized** (Poisson) generation based on whether the splat represents a discrete object or a large-scale environment [S1].
2.  **Density Field Construction:**
    *   *For Object-Optimized:* Create a mesh for each Gaussian ellipsoid from the input PLY file, arranged in a mesh grid. Construct a splat grid and an isosurface based on the Gaussian ellipsoid densities [S1].
    *   *For Space-Optimized:* Construct a density field suitable for Poisson reconstruction, typically using a coarser resolution to manage compute at scale [S1].
3.  **Surface Reconstruction:**
    *   *Marching Cubes:* Perform marching cubes over the isosurface result to create a smoothed surface mesh [S1].
    *   *Poisson:* Employ the Poisson surface reconstruction algorithm to recover the largest surfaces (e.g., floors, walls) [S1].
4.  **Optimization and Cleanup:**
    *   *Decimation:* Decimate the mesh for performance [S1].
    *   *Outlier Removal:* For Poisson reconstruction, crop results by removing vertex outliers [S1].
    *   *Flattening:* Implement a flattening procedure to adjust vertices at the base of the mesh along a specified axis, preventing undesirable behavior (e.g., toppling) when placing objects on flat surfaces [S1].
5.  **Integration:** Create the mesh in OBJ format, auto-align it with the input splat, parent it to the splat game object, and assign mesh colliders [S1].

**Parameters:**
*   **Algorithm:** "Marching Cubes" (for convex objects) or "Poisson" (for non-convex spaces/large-scale scenes) [S1].
*   **Splat Scale:** A parameter to adjust object reconstruction accuracy based on structural characteristics [S1].
*   **Meshing Grid Threshold:** A parameter to control the resolution and detail of the generated mesh [S1].
*   **Resolution:** Coarser resolution is used for large-scale scenes (Poisson) compared to individual objects to manage compute [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_phantom_mesh(splats, algorithm, scale, threshold):
    """
    Generates a phantom mesh from splat primitives.
    
    Args:
        splats: List of Gaussian ellipsoids (position, covariance, density).
        algorithm: 'marching_cubes' or 'poisson'.
        scale: Float, scaling factor for reconstruction accuracy.
        threshold: Float, grid threshold for isosurface extraction.
        
    Returns:
        mesh: A mesh object with vertices and faces.
    """
    if algorithm == 'marching_cubes':
        # Object-Optimized Path
        # 1. Create intermediate mesh for each Gaussian ellipsoid
        gaussian_meshes = []
        for gauss in splats:
            # Create a mesh representation of the Gaussian ellipsoid
            g_mesh = create_ellipsoid_mesh(gauss.position, gauss.covariance, gauss.density)
            gaussian_meshes.append(g_mesh)
        
        # 2. Construct splat grid and isosurface from densities
        splat_grid = build_density_grid(gaussian_meshes, threshold)
        isosurface = extract_isosurface(splat_grid)
        
        # 3. Perform marching cubes for smoothed surface
        mesh = marching_cubes(isosurface)
        
        # 4. Decimate for performance
        mesh = decimate_mesh(mesh)
        
    elif algorithm == 'poisson':
        # Space-Optimized Path
        # 1. Construct density field for Poisson reconstruction
        # Use coarser resolution for large-scale scenes
        density_field = build_poisson_density_field(splats, resolution='coarse')
        
        # 2. Perform Poisson surface reconstruction
        mesh = poisson_reconstruction(density_field)
        
        # 3. Crop results by removing vertex outliers
        mesh = remove_outliers(mesh)
    
    # 5. Flatten base vertices if necessary (e.g., for upright placement)
    mesh = flatten_base(mesh, axis='y')
    
    return mesh
```

**Edge Cases and Pitfalls:**
*   **Convexity Assumption:** Marching Cubes assumes convexity, making it unsuitable for complex spaces [S1].
*   **Uneven Surfaces:** Uneven surface reconstruction at the base of splat objects can cause instability (e.g., toppling) when placed on flat surfaces; flattening is required [S1].
*   **Compute Scale:** Large-scale reconstructions require coarser resolutions to manage compute time [S1].

## Subtopics

### Marching Cubes Reconstruction
Produces fine detail but assumes convexity; suitable for discrete objects [S1].

### Poisson Surface Reconstruction
Handles non-convex spaces; used for large-scale environments and spaces where convexity does not hold [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Marching Cubes produces fine detail but is computationally heavier for large scenes; Poisson uses coarser resolution for efficiency in large-scale scenes [S1].
*   **Strengths:** Enables standard physics simulations (collision, shadows) without altering visual rendering pipeline [S1].
*   **Limitations:** Requires additional mesh generation step; Marching Cubes fails on non-convex shapes [S1].
*   **When to use:** When physics interactions (collision, shadows) are required for splat objects [S1].
*   **When not to use:** When only visual rendering is needed, or for non-convex spaces if using Marching Cubes [S1].

## Background
Phantom Mesh Generation was introduced in **SplatXR**, a toolkit for multi-layered extended reality via on-headset 3D Gaussian Splatting [S1]. It was presented at **UIST ’26** in November 2026 [S1]. The method addresses the lack of surface information in splats, which prohibits direct use of standard graphics models for physics simulations like shadow map computations or collision detection [S1]. It allows the creation of a "phantom structure" that is accessed by the 3D engine for physics while the splat writes to the visual framebuffer [S1].

## Sources
[S1] SplatXR: A Toolkit for Multi-Layered Extended Reality via On-Headset 3D Gaussian Splatting, https://arxiv.org/abs/2610.26338, fetched 2026-10-26
