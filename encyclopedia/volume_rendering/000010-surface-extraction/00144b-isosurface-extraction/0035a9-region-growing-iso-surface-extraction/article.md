# Region-growing Iso-surface Extraction

Also called: Normal consistency constrained marching triangles, Novel region-growing algorithm, Region-growing based iso-surface extraction algorithm

## Overview
Region-growing Iso-surface Extraction is a method for generating high-quality, curvature-adaptive semi-regular meshes from volumetric data [S1]. It solves the limitations of standard marching cubes by preserving sharp features and extracting all disjoint components of the iso-surface without topological errors [S1]. Unlike [Marching Cubes](30-isosurface-extraction.md), which processes cells independently and often produces irregular triangles, this method iteratively grows the mesh from seed triangles, ensuring global consistency through a novel normal consistency constraint [S1].

## How it works
The algorithm operates on a discretized volume domain. It constructs a mesh by propagating a front of triangles across the iso-surface, adapting triangle size to local curvature [S1].

**Inputs:**
*   Scalar volume field $V$ defined on a grid.
*   Iso-value $T$ defining the surface.

**Outputs:**
*   A triangular mesh $M$ representing the iso-surface.

**Data Structures:**
*   **Boundary Particles:** Points on the iso-surface. Each particle $p$ stores its 3D position $\mathbf{p}$, normal vector $\mathbf{n}$, and a list of adjacent triangles.
*   **Growth Queue:** A priority queue of candidate triangles or edges to be added to the mesh.
*   **Visited Set:** A set of boundary particles or triangles already processed to prevent duplicates.

**Steps:**

1.  **Boundary Particle Sampling and Classification:**
    Sample points near the iso-surface (e.g., via marching cubes or grid traversal). Classify them as boundary particles if they lie within a threshold distance of the iso-surface $V(\mathbf{x}) = T$. Compute normals for these particles.

2.  **Feature Position Extraction:**
    Identify sharp features (edges, corners) in the volume. This is done by detecting high curvature or gradient changes in the scalar field. Mark particles near these features as "feature particles" to ensure they are preserved in the final mesh [S1].

3.  **Prioritized Seed Initialization:**
    Initialize the mesh with a set of seed triangles. Seeds are selected based on priority, often favoring regions with high curvature or feature points to ensure adaptive resolution [S1]. The initial seeds must form a valid topological disk.

4.  **Front Propagation:**
    While the growth queue is not empty:
    a. Extract the highest priority candidate triangle $t_{new}$ from the queue.
    b. **Normal Consistency Check:** Verify that the normal $\mathbf{n}_{new}$ of $t_{new}$ is consistent with the normals of the adjacent triangles in the current mesh. Specifically, ensure that the intersection of the Delaunay sphere of $t_{new}$ and the iso-surface is a topological disk. This prevents self-intersections and handles complex topology [S1].
    c. If the check passes, add $t_{new}$ to the mesh $M$.
    d. Generate new candidate triangles from the boundary edges of $t_{new}$ and add them to the queue with priorities based on local curvature [S1].

5.  **Visited Boundary Particle Identification:**
    Mark particles and triangles added in Step 4 as visited. Ensure that no particle is processed twice to maintain mesh coherence [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_growing_iso_extraction(volume, iso_value):
    # Step 1: Sampling
    boundary_particles = sample_boundary_particles(volume, iso_value)
    
    # Step 2: Feature Extraction
    feature_particles = extract_features(boundary_particles, volume)
    
    # Step 3: Seed Initialization
    seed_triangles = prioritize_seeds(boundary_particles, feature_particles)
    mesh = Mesh()
    for tri in seed_triangles:
        mesh.add_triangle(tri)
        
    growth_queue = PriorityQueue()
    for tri in seed_triangles:
        growth_queue.enqueue(tri, priority=curvature(tri))
        
    visited_particles = set()
    for tri in seed_triangles:
        visited_particles.update(tri.vertices)
        
    # Step 4 & 5: Propagation and Visited Check
    while not growth_queue.is_empty():
        candidate_tri = growth_queue.dequeue()
        
        # Check if vertices are already visited to avoid duplicates
        if not all(v in visited_particles for v in candidate_tri.vertices):
            # Normal Consistency Constraint
            if is_normal_consistent(candidate_tri, mesh):
                # Delaunay Sphere Intersection Check
                if is_topological_disk(candidate_tri, mesh):
                    mesh.add_triangle(candidate_tri)
                    visited_particles.update(candidate_tri.vertices)
                    
                    # Generate new candidates from boundary edges
                    new_candidates = generate_boundary_triangles(candidate_tri, volume, iso_value)
                    for new_tri in new_candidates:
                        if not is_visited(new_tri, visited_particles):
                            growth_queue.enqueue(new_tri, priority=curvature(new_tri))
                            
    return mesh

def is_normal_consistent(new_tri, mesh):
    # Check if normal of new_tri aligns with adjacent triangles
    # Returns True if angle between normals is within threshold
    pass

def is_topological_disk(new_tri, mesh):
    # Ensure Delaunay sphere intersection is a topological disk
    # This prevents self-intersection and handles complex topology
    pass
```

**Parameters:**
*   **Curvature Threshold:** Determines the priority of seeds and new triangles. Higher curvature leads to smaller triangles [S1].
*   **Normal Consistency Threshold:** The maximum allowed angle between the normal of a new triangle and its neighbors [S1].
*   **Delaunay Radius:** Used in the topological disk check [S1].

**Edge Cases:**
*   **Disjoint Components:** The algorithm must be run for each disjoint component of the iso-surface. Seeds must be initialized for each component [S1].
*   **Sharp Features:** If feature extraction fails, sharp features may be smoothed out. The feature extraction step is critical for preserving these details [S1].
*   **Complex Topology:** The normal consistency constraint is designed to handle complex topology, but extremely noisy data may still cause issues if the sampling is insufficient [S1].

## Subtopics

### [3D reconstruction based on edge growth](8730-3d-reconstruction-based-on-edge-growth.md)
A variant that propagates from seed triangles through a queue of "growth edges" to solve topological ambiguity and hole-generation.

### [Region Growing Segmentation](9712-region-growing-segmentation.md)
A voxel-based segmentation technique that extracts connected regions based on similarity criteria, often used as a pre-processing step.

## Operating characteristics
*   **Time Complexity:** Depends on the number of boundary particles and the efficiency of the priority queue. Generally $O(N \log N)$ where $N$ is the number of triangles, due to queue operations [S1].
*   **Memory Complexity:** Proportional to the number of boundary particles and triangles stored in the mesh and queue [S1].
*   **Performance:** Generates higher quality, evenly-sized, and quasi-equilateral meshes compared to marching cubes [S1]. It is robust for large-scale volumetric datasets with complex topology [S1].
*   **Strengths:** Curvature-adaptive, preserves sharp features, extracts all disjoint components, robust topology [S1].
*   **Limitations:** More complex implementation than marching cubes; requires careful parameter tuning for feature extraction and normal consistency [S1].

## Background
The method was proposed by Jian Xi and Ye Duan in 2008 [S1]. It builds on the concept of the marching triangles algorithm, which generates higher quality meshes by iteratively joining new triangles to the current region boundary [S1]. The original marching triangles algorithm did not adapt well to local surface properties, requiring many small triangles for high-curvature surfaces. This new algorithm introduces a normal consistency constraint to ensure robustness and adaptivity [S1].

## Sources
[S1] A novel region-growing based iso-surface extraction algorithm, https://doi.org/10.1016/j.cag.2008.09.007, fetched 2026-10-27
