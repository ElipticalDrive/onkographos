# Region-growing Iso-surface Extraction

Also called: Normal consistency constrained marching triangles, Novel region-growing algorithm, Region-growing based iso-surface extraction algorithm

## Overview
Region-growing Iso-surface Extraction is a method that generates high-quality, curvature-adaptive semi-regular meshes while preserving sharp features and extracting all disjoint components of an iso-surface [S1]. It improves upon the original Marching Triangles algorithm by introducing a normal consistency constraint that ensures the intersection of the Delaunay sphere of a new triangle and the iso-surface is a topological disk, providing robustness for complex topologies [S1]. Unlike standard grid-based methods that process cells independently, this approach builds the mesh iteratively from seed triangles, adapting triangle size to local surface curvature [S1].

## How it works
The algorithm operates on a volumetric dataset defined by a scalar field $f(\mathbf{x})$ and a target isovalue $c$. The output is a triangular mesh representing the iso-surface $f(\mathbf{x}) = c$.

### Data Structures
*   **Boundary Particles**: Points sampled on the iso-surface. Each particle stores its 3D position $\mathbf{p}$, normal vector $\mathbf{n}$, and classification status (e.g., feature vs. non-feature).
*   **Growth Front**: A queue or list of boundary particles/triangles eligible for expansion.
*   **Visited Set**: A collection of particles or triangles already incorporated into the mesh to prevent re-processing.
*   **Delaunay Triangulation**: Local triangulation of the growth front used to evaluate geometric constraints.

### Steps
1.  **Boundary Particle Sampling and Classification**: Sample points on the iso-surface. Classify them as feature points (sharp edges/corners) or smooth points based on local curvature or gradient changes [S1].
2.  **Feature Position Extraction**: Identify and mark positions of sharp features to ensure they are preserved in the final mesh [S1].
3.  **Prioritized Seed Initialization**: Select initial seed triangles from the classified particles. Seeds are prioritized based on feature importance or curvature to ensure robust coverage of complex regions [S1].
4.  **Front Propagation**: Iteratively expand the mesh from the current front:
    *   Select a candidate triangle from the front.
    *   Apply the **Normal Consistency Constraint**: Verify that the new triangle's normal is consistent with the surrounding surface orientation. Specifically, ensure that the intersection of the Delaunay sphere of the new triangle with the iso-surface forms a topological disk [S1]. This prevents topological errors and self-intersections.
    *   If the constraint is satisfied, add the triangle to the mesh and update the growth front with new boundary particles.
    *   If not, discard the candidate or adjust parameters [S1].
5.  **Visited Boundary Particle Identification**: Mark processed particles to avoid redundant checks and ensure all disjoint components are eventually reached by re-initializing seeds if necessary [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_growing_iso_extraction(volume_data, isovalue):
    # Step 1: Boundary particle sampling and classification
    particles = sample_boundary_particles(volume_data, isovalue)
    classify_particles(particles) # Classify as feature or smooth
    
    # Step 2: Feature position extraction
    feature_positions = extract_feature_positions(particles)
    
    # Step 3: Prioritized seed initialization
    seed_queue = prioritize_seeds(particles, feature_positions)
    
    mesh = empty_mesh()
    visited = set()
    
    # Loop until all components are extracted
    while seed_queue is not empty or front is not empty:
        if seed_queue is not empty:
            current_seed = seed_queue.pop()
            front = [current_seed]
        else:
            current_seed = front.pop()
            
        if current_seed in visited:
            continue
            
        # Step 4: Front propagation
        new_triangles = generate_candidate_triangles(current_seed, front)
        
        for tri in new_triangles:
            # Normal consistency constraint
            if is_normal_consistent(tri, front) and is_delaunay_disk(tri):
                mesh.add_triangle(tri)
                visited.add(tri.vertices)
                update_front(front, tri)
            else:
                # Constraint failed, skip or handle error
                pass
                
    return mesh

def is_normal_consistent(triangle, front):
    # Check if the new triangle's normal is consistent with the local surface orientation
    # This ensures the surface does not fold back on itself locally
    for neighbor in front:
        if dot_product(triangle.normal, neighbor.normal) < threshold:
            return False
    return True

def is_delaunay_disk(triangle):
    # Ensure the intersection of the Delaunay sphere of the new triangle 
    # and the iso-surface is a topological disk
    sphere = compute_delaunay_sphere(triangle)
    intersection = compute_intersection(sphere, iso_surface_approximation)
    return is_topological_disk(intersection)
```

### Parameters
*   **Isovalue ($c$)**: The scalar value defining the surface.
*   **Normal Consistency Threshold**: A tolerance for normal alignment to determine consistency [S1].
*   **Curvature Adaptation Factor**: Controls the size of triangles relative to local curvature [S1].

### Edge Cases and Pitfalls
*   **Disjoint Components**: The algorithm must re-initialize seeds to ensure all disconnected parts of the iso-surface are extracted [S1].
*   **Sharp Features**: Failure to properly classify and prioritize feature points may result in loss of sharp details [S1].
*   **Topological Errors**: Violation of the normal consistency constraint can lead to non-manifold geometries or self-intersections [S1].

## Subtopics

### [3D reconstruction based on edge growth](8730-3d-reconstruction-based-on-edge-growth.md)
A variant that propagates from seed triangles through a queue of "growth edges" to solve topological ambiguity and hole-generation problems inherent in standard Marching Cubes.

### [Region Growing Segmentation](9712-region-growing-segmentation.md)
A related technique that extracts a connected region of interest from a 3D volume by expanding from a seed point based on value range similarity, generating a binary mask rather than a mesh.

## Operating characteristics
*   **Time Complexity**: Not explicitly given in the sources. The iterative nature of front propagation suggests complexity dependent on the number of boundary particles and the cost of Delaunay checks [S1].
*   **Memory Complexity**: Not explicitly given in the sources. Requires storage for boundary particles, the growth front, and the resulting mesh [S1].
*   **Strengths**: Generates high-quality, curvature-adaptive semi-regular meshes; preserves sharp features; robust for complex topologies due to normal consistency constraints; extracts all disjoint components [S1].
*   **Limitations**: More complex than standard Marching Cubes; requires preprocessing steps (sampling, classification, feature extraction) [S1].
*   **When to Use**: When high-quality, adaptive meshes are required, especially for complex geometries with sharp features or varying curvature [S1].
*   **When Not to Use**: For simple, uniform geometries where standard methods like Marching Cubes are sufficient and faster [S1].

## Background
The algorithm was proposed by Jian Xi and Ye Duan in 2008 [S1]. It builds on the concept of the Marching Triangles algorithm, which generates evenly-sized and quasi-equilateral meshes by iteratively joining new triangles to the current region boundary [S1]. The original Marching Triangles algorithm did not adapt well to local surface properties, requiring many small triangles for surfaces with large curvature variations [S1]. This new algorithm addresses these limitations by incorporating curvature adaptation and normal consistency constraints [S1].

## Sources
[S1] A novel region-growing based iso-surface extraction algorithm, https://doi.org/10.1016/j.cag.2008.09.007, fetched 2026-10-27
