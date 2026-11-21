# Region-growing Iso-surface Extraction

Also called: Normal consistency constrained marching triangles, Novel region-growing algorithm, Region-growing based iso-surface extraction algorithm

## Overview
Region-growing Iso-surface Extraction is a surface extraction method that generates high-quality, curvature-adaptive semi-regular meshes while preserving sharp features and extracting all disjoint components of an iso-surface [S1]. It improves upon the original Marching Triangles algorithm by introducing a novel normal consistency constraint, which ensures robustness for complex topologies by guaranteeing that the intersection of the Delaunay sphere of a new triangle and the iso-surface is a topological disk [S1]. Unlike standard marching cubes which processes cubes independently, this method builds on [Region-growing and Advancing Front](17871-region-growing-and-advancing-front.md) by explicitly tracking connectivity across cube boundaries to solve topological ambiguity and hole-generation problems [S1].

## How it works
The algorithm operates on a volumetric dataset to extract an iso-surface defined by a threshold value. It produces a triangular mesh where triangle sizes adapt to local surface curvature.

**Inputs:**
*   Volumetric dataset $V$ (scalar field).
*   Iso-surface threshold value $T$.

**Outputs:**
*   Triangular mesh $M$ representing the iso-surface.

**Data Structures:**
*   **Boundary Particles:** A set of points on the iso-surface. Each particle $p$ stores its 3D position $\mathbf{p}$, its surface normal $\mathbf{n}$, and a status flag (e.g., unvisited, visited, boundary).
*   **Growth Queue:** A priority queue storing candidate triangles or edges for expansion. Prioritization is based on curvature or feature importance [S1].
*   **Visited Set:** A hash map or grid-based structure tracking processed boundary particles to prevent re-processing.

**Algorithm Steps:**

1.  **Boundary Particle Sampling and Classification:**
    Sample points on the iso-surface from the volume. Classify them based on local geometric properties (e.g., curvature, gradient magnitude) to identify potential seed locations and feature points [S1].

2.  **Feature Position Extraction:**
    Identify sharp features and high-curvature regions. These positions are marked as high-priority seeds to ensure sharp features are preserved in the final mesh [S1].

3.  **Prioritized Seed Initialization:**
    Initialize the growth front by selecting seed triangles from the high-priority feature positions and other classified boundary particles. Seeds are added to the growth queue with priorities determined by their geometric significance [S1].

4.  **Front Propagation:**
    Loop until the queue is empty or all disjoint components are extracted:
    a.  Dequeue the highest-priority candidate triangle $t_{new}$.
    b.  **Normal Consistency Check:** Verify that the normal vector of $t_{new}$ is consistent with the normals of its neighboring existing triangles on the front. Specifically, ensure that the intersection of the Delaunay sphere of $t_{new}$ and the iso-surface is a topological disk [S1]. This constraint is more robust than the original Marching Triangles' Delaunay face property check alone [S1].
    c.  If the check passes, add $t_{new}$ to the mesh $M$.
    d.  Generate new candidate triangles from the boundary edges of $t_{new}$.
    e.  Classify and prioritize these new candidates based on local curvature and feature strength.
    f.  Add valid candidates to the growth queue.

5.  **Visited Boundary Particle Identification:**
    Mark particles associated with added triangles as visited. Ensure that particles belonging to disjoint components are identified and initialized as new seeds if they were not reached by the current front [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_growing_iso_surface_extraction(volume, threshold):
    # Step 1: Boundary particle sampling and classification
    boundary_particles = sample_boundary_particles(volume, threshold)
    classify_particles(boundary_particles)

    # Step 2: Feature position extraction
    feature_positions = extract_feature_positions(boundary_particles)

    # Step 3: Prioritized seed initialization
    growth_queue = PriorityQueue()
    visited = set()
    mesh = Mesh()

    # Add feature positions as high-priority seeds
    for pos in feature_positions:
        seed_triangle = create_seed_triangle(pos, boundary_particles)
        growth_queue.push(seed_triangle, priority=HIGH)
        visited.add(pos.id)

    # Add other classified particles as lower-priority seeds
    for particle in boundary_particles:
        if particle not in visited and particle.is_seed_candidate:
            seed_triangle = create_seed_triangle(particle, boundary_particles)
            growth_queue.push(seed_triangle, priority=LOW)
            visited.add(particle.id)

    # Step 4 & 5: Front propagation and visited identification
    while not growth_queue.is_empty():
        candidate = growth_queue.pop()
        
        # Normal consistency constraint check
        if is_normal_consistent(candidate, mesh, boundary_particles):
            # Ensure Delaunay sphere intersection is a topological disk
            if is_topological_disk(candidate.delaunay_sphere, volume, threshold):
                mesh.add_triangle(candidate)
                
                # Generate new candidates from boundary edges
                new_candidates = generate_boundary_candidates(candidate, boundary_particles)
                
                for new_cand in new_candidates:
                    if new_cand not in visited:
                        # Prioritize based on curvature/feature strength
                        priority = calculate_priority(new_cand)
                        growth_queue.push(new_cand, priority=priority)
                        visited.add(new_cand.id)
        else:
            # Discard inconsistent candidates
            pass

    return mesh

def is_normal_consistent(triangle, mesh, particles):
    # Check if triangle normal is consistent with neighbors
    # Returns True if consistent, False otherwise
    pass

def is_topological_disk(sphere, volume, threshold):
    # Check if intersection of sphere and iso-surface is a topological disk
    # Returns True if topological disk, False otherwise
    pass
```

**Edge Cases and Pitfalls:**
*   **Disjoint Components:** The algorithm must explicitly check for unvisited boundary particles after the main loop to ensure all disjoint components are extracted [S1].
*   **Topological Ambiguity:** The normal consistency constraint is critical for handling complex topologies where standard Delaunay checks might fail [S1].
*   **Sharp Features:** Failure to prioritize feature positions may result in loss of sharp features in the output mesh [S1].

## Subtopics

### [3D reconstruction based on edge growth](8730-3d-reconstruction-based-on-edge-growth.md)
A variant that reconstructs surfaces by propagating from seed triangles through a queue of "growth edges". It solves topological ambiguity and hole-generation by explicitly tracking connectivity across cube boundaries, using specific topological configurations for triangle and quadrangle edges.

### [Region Growing Segmentation](9712-region-growing-segmentation.md)
A semi-automatic technique that extracts connected regions of interest from a 3D volume by iteratively adding neighboring voxels to a user-selected seed point based on statistical similarity criteria. It distinguishes itself by relying on local homogeneity rather than global cut optimization.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The algorithm generates high-quality curvature-adaptive semi-regular meshes and preserves sharp features [S1]. It is robust for large-scale volumetric datasets with complex topology and geometry [S1].
*   **Strengths:** Generates evenly-sized and quasi-equilateral meshes; adapts to local surface properties (curvature); preserves sharp features; extracts all disjoint components; robust against topological ambiguity via normal consistency constraints [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-quality, curvature-adaptive meshes are required, especially for complex topologies or when sharp features must be preserved [S1].
*   **When not to use:** Not given in the sources.

## Background
The algorithm was proposed by Jian Xi and Ye Duan in 2008 [S1]. It builds on the concept of the marching triangles algorithm, which generates higher quality evenly-sized and quasi-equilateral meshes by iteratively joining new triangles to the current region boundary [S1]. The original marching triangles algorithm did not adapt well to local surface properties, requiring many small triangles for surfaces with large curvature variations [S1]. This new algorithm improves upon it by using a prioritized seed initialization scheme and normal consistency constraints [S1].

## Sources
[S1] A novel region-growing based iso-surface extraction algorithm, https://doi.org/10.1016/j.cag.2008.09.007, fetched 2026-10-27
