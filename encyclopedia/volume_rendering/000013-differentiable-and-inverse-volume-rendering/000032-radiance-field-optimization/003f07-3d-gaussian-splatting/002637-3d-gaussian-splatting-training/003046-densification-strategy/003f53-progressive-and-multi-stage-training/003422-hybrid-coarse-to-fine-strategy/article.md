# Hybrid Coarse-to-Fine Strategy

Also called: Multi-stage refinement, Hybrid training strategy

## Overview
The Hybrid Coarse-to-Fine Strategy is a training methodology for point-based radiance field representations that decouples the optimization of global spatial structure from local geometric details by dividing the training timeline into distinct stages [S1]. Unlike single-stage training, which optimizes the point cloud from initialization to completion in one continuous pass, this strategy employs a coarse initialization phase followed by a fine-tuning phase to improve view synthesis quality [S1]. It addresses the problem of converging to suboptimal local minima or producing blurry details when optimizing a dense set of points from scratch, yielding significantly higher rendering fidelity than single-stage baselines [S1].

## How it works
The method operates by initializing a sparse representation and progressively refining it, often involving specific densification or filtering mechanisms depending on the variant [S1, S2].

### Inputs and Outputs
*   **Inputs:** A set of input images (and associated camera poses), a target resolution, and training hyperparameters (learning rate, iteration counts).
*   **Outputs:** An optimized radiance field representation (e.g., a point cloud with attributes or a mesh) and the corresponding rendered images.

### Data Structures
The specific data structure depends on the variant. In the general point-based context, this involves a set of points with attributes (position, opacity, spherical harmonics coefficients) [S1]. In the mesh-based variant, it involves a set of vertices and a connectivity graph (triangles) [S2].

### Steps
1.  **Initialization:** Start with a sparse or low-fidelity representation. For point-based methods, this may involve initializing points based on feature detection or random sampling [S1]. For mesh-based variants, initialize a sparse mesh by computing Sobel gradient magnitudes on a downsampled target image and sampling vertex positions proportionally to edge strength, with a small uniform floor for coverage [S2].
2.  **Coarse Stage:** Optimize the initial sparse representation. This stage focuses on learning the global structure and approximate geometry [S1].
3.  **Refinement/Densification:** Transition to the fine stage. This may involve densifying the point cloud or refining the mesh structure. In mesh variants, this occurs every 500 iterations (starting at iteration 1,500 and ending at 15,000) [S2].
4.  **Fine Stage:** Optimize the refined representation with higher resolution or more detailed attributes. In mesh variants, this involves scoring triangles and inserting new vertices [S2].
5.  **Final Optimization:** Complete the training with the final representation [S1].

### Formulas
In the mesh-based variant, triangle scoring is defined as [S2]:

$$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$

Where:
*   $\text{score}_t$ is the score for triangle $t$.
*   $\overline{\text{DSSIM}}_t$ is the mean DSSIM (per-pixel structural dissimilarity) of triangle $t$.
*   $|\text{pixels}_t|$ is the number of pixels covered by triangle $t$.
*   $\alpha = 0.75$ is the area-weighting exponent.

### Parameters
*   **Iteration Interval:** Densification steps occur every 500 iterations [S2].
*   **Start/End Iterations:** Densification starts at iteration 1,500 and ends at 15,000 [S2].
*   **Area-weighting exponent ($\alpha$):** 0.75 [S2].
*   **Vertex Insertion Rate:** Up to $0.35 \times V$ new vertices, where $V$ is the current vertex count [S2].

### Edge Cases and Pitfalls
*   **Degeneracy:** In mesh variants, the mesh may become degenerate (e.g., vertices becoming colinear), requiring re-meshing, although this rarely occurs in practice [S2].
*   **Small Triangles:** Triangles with fewer than 3 pixels are excluded from scoring to prevent instability [S2].
*   **Precision:** The method relies on accurate gradient computation and DSSIM calculation, which may be sensitive to noise in low-texture regions [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Hybrid Coarse-to-Fine Strategy (Mesh Variant)
# Based on [S2]

def hybrid_coarse_fine_training(images, camera_poses, max_iterations=15000):
    # 1. Initialization
    # Compute Sobel gradient magnitudes on a downsampled target image
    gradients = compute_sobel_gradients(downsample(images[0]))
    
    # Sample initial vertex positions proportionally to edge strength
    # with a small uniform floor for coverage
    vertices = sample_vertices_proportional_to_gradients(gradients, floor=0.01)
    
    # Create a Delaunay triangulation of sampled points
    # with boundary vertices ensuring full image coverage
    mesh = create_delaunay_triangulation(vertices)
    
    # 2. Coarse Stage Optimization
    # Optimize the initial sparse mesh
    optimize_mesh(mesh, images, camera_poses, iterations=1500)
    
    # 3. Fine Stage / Densification Loop
    for iteration in range(1500, max_iterations, 500):
        # Score triangles
        for triangle in mesh.triangles:
            # Calculate mean DSSIM for the triangle
            dssim = compute_mean_dssim(triangle, images, camera_poses)
            # Count pixels covered by the triangle
            pixel_count = count_pixels_covered(triangle)
            
            # Exclude triangles with fewer than 3 pixels
            if pixel_count < 3:
                triangle.score = 0
                continue
            
            # Calculate score using the formula
            alpha = 0.75
            triangle.score = dssim * (pixel_count ** alpha)
        
        # Identify top-scoring triangles
        top_triangles = get_top_scoring_triangles(mesh.triangles, threshold=0.9)
        
        # Insert centroid vertices for top-scoring triangles
        # up to 0.35 * V new vertices
        v_count = len(mesh.vertices)
        max_new_vertices = int(0.35 * v_count)
        new_vertices = []
        
        for triangle in top_triangles:
            if len(new_vertices) >= max_new_vertices:
                break
            
            # Insert centroid vertex
            centroid = compute_centroid(triangle)
            # Initialize features by averaging parent triangle's vertex features
            centroid_features = average_vertex_features(triangle.vertices)
            new_vertex = create_vertex(centroid, centroid_features)
            new_vertices.append(new_vertex)
        
        # Add new vertices to the mesh
        mesh.add_vertices(new_vertices)
        
        # Re-run Delaunay triangulation on the full vertex set
        mesh = re_triangulate(mesh)
        
        # Check for degeneracy and re-mesh if necessary
        if is_degenerate(mesh):
            mesh = remesh(mesh)
        
        # Optimize the refined mesh
        optimize_mesh(mesh, images, camera_poses, iterations=500)
    
    return mesh
```

## Subtopics

### Coarse-to-Fine Densification
A training strategy where a sparse mesh is progressively refined by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. It is chosen when a mesh-based representation is preferred over a point-based one, offering structured geometry [S2]. The key differentiator is the explicit use of triangle scoring based on DSSIM and area weighting to guide densification [S2].

## Operating characteristics
*   **Time Complexity:** The densification step adds overhead due to scoring and re-triangulation, but the overall training time is still efficient compared to neural radiance fields. Training generally takes between 1-3 minutes on an RTX A6000 for the mesh variant [S2].
*   **Memory Complexity:** Depends on the number of points or vertices. The mesh variant requires storing connectivity information [S2].
*   **Performance:** On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
*   **Strengths:** Improves rendering quality over single-stage training [S1]. Allows for efficient optimization of complex scenes [S1].
*   **Limitations:** Requires careful tuning of densification parameters (e.g., iteration intervals, scoring thresholds) [S2]. May be sensitive to initialization [S1].
*   **When to use:** When high-quality view synthesis is required and single-stage training yields suboptimal results [S1]. When a structured mesh representation is beneficial [S2].
*   **When not to use:** When computational resources are extremely limited and the quality gain is not worth the overhead [S1]. When a simple point-based representation suffices [S1].

## Background
The Hybrid Coarse-to-Fine Strategy was introduced in the context of differentiable point-based radiance fields for efficient view synthesis [S1]. It departs from volumetric representations and coordinate-network representations that require ray-marching with hundreds of network evaluations per ray [S1]. The strategy was validated through ablation experiments showing its contribution to rendering quality [S1]. The mesh-based variant was introduced in the context of neural harmonic textures for high-quality primitive-based neural reconstruction [S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
