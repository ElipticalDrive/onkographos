# Region-growing Iso-surface Extraction

Also called: Normal consistency constrained marching triangles, Novel region-growing algorithm, Region-growing based iso-surface extraction algorithm

## Overview
Region-growing Iso-surface Extraction is a method that generates high-quality, curvature-adaptive semi-regular meshes while preserving sharp features and extracting all disjoint components of an iso-surface [S1]. It improves upon the original Marching Triangles algorithm by introducing a normal consistency constraint that ensures the intersection of the Delaunay sphere of a new triangle with the iso-surface is a topological disk, providing robustness for complex topologies [S1]. Unlike Marching Cubes, which produces uniform grids, this method adapts triangle size to local curvature, and unlike standard Marching Triangles, it explicitly handles feature preservation and topological robustness via normal checks [S1].

## How it works
The algorithm operates on a volumetric dataset to extract iso-surfaces. It follows a pipeline of five steps, looping through steps 3–5 until all disjoint components are extracted [S1].

**Inputs:**
*   Volumetric scalar field data.
*   Isovalue threshold.

**Outputs:**
*   A triangular mesh representing the iso-surface.

**Data Structures:**
*   **Boundary Particles:** Points sampled on the boundary of the iso-surface. Each particle likely stores position, normal, and classification status (e.g., feature vs. non-feature) [S1].
*   **Seed List:** A prioritized list of initial triangles for region growth [S1].
*   **Front:** The set of boundary triangles currently being expanded [S1].
*   **Visited Set:** A record of boundary particles already processed to ensure completeness [S1].

**Steps:**

1.  **Boundary Particle Sampling and Classification:**
    Sample points on the iso-surface boundary. Classify them to identify sharp features and general surface areas [S1].

2.  **Feature Position Extraction:**
    Identify and extract precise positions of sharp features from the classified particles [S1].

3.  **Prioritized Seed Initialization:**
    Initialize the region-growing process by selecting seed triangles. This step is prioritized to ensure coverage of complex features and disjoint components [S1].

4.  **Front Propagation:**
    Iteratively grow the mesh by adding new triangles to the current front.
    *   **Normal Consistency Constraint:** When considering a new triangle, check if the intersection of its Delaunay sphere with the iso-surface is a topological disk. This is verified using normal consistency constraints, which is a key improvement over the original Marching Triangles algorithm that only used Delaunay face property checks [S1].
    *   If the constraint is satisfied, the triangle is added to the mesh and the front is updated [S1].

5.  **Visited Boundary Particle Identification:**
    Identify and mark boundary particles that have been processed to ensure all disjoint components are eventually extracted and no particles are missed [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_growing_iso_surface_extraction(volume_data, isovalue):
    # Step 1: Boundary particle sampling and classification
    boundary_particles = sample_boundary_particles(volume_data, isovalue)
    classify_particles(boundary_particles)

    # Step 2: Feature position extraction
    feature_positions = extract_feature_positions(boundary_particles)

    # Step 3: Prioritized seed initialization
    seeds = initialize_seeds(boundary_particles, feature_positions)

    # Step 4 & 5: Loop until all components are extracted
    while not all_components_extracted(seeds, boundary_particles):
        # Prioritized seed initialization (re-evaluated or continued)
        current_front = get_front(seeds)
        
        if not current_front:
            break
            
        new_triangles = []
        for triangle in current_front:
            # Check normal consistency constraint
            if is_normal_consistent(triangle, volume_data, isovalue):
                # Ensure Delaunay sphere intersection is a topological disk
                if is_topological_disk(triangle, volume_data, isovalue):
                    new_triangles.append(triangle)
                    mark_visited(triangle.boundary_particles)
        
        update_front(seeds, new_triangles)
        
        # Identify visited particles to ensure completeness
        identify_visited_particles(boundary_particles)

    return generate_mesh(seeds)

def is_normal_consistent(triangle, volume_data, isovalue):
    # Check local iso-surface information via normal consistency
    # This ensures robustness for complex topology
    # [G] Implementation detail: Compare normals of adjacent triangles
    # and check against gradient of volume data at vertices.
    pass

def is_topological_disk(triangle, volume_data, isovalue):
    # Ensure intersection of Delaunay sphere and iso-surface is a topological disk
    # [G] Implementation detail: Compute Delaunay sphere of triangle vertices,
    # sample points on sphere, check sign changes against isovalue.
    pass
```

## Subtopics

### [3D reconstruction based on edge growth](8730-3d-reconstruction-based-on-edge-growth.md)
A 3D reconstruction technique that grows iso-surfaces from seed triangles using growth edges, handling two types of edges: triangle edges and quadrangle edges.

### [Region Growing Segmentation](9712-region-growing-segmentation.md)
A technique to extract a connected region from a 3D volume based on pre-defined connecting criteria, used to isolate a region of interest via a mask volume.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The algorithm is described as very robust for large-scale volumetric datasets with complex topology and geometry [S1]. It generates high-quality curvature-adaptive semi-regular meshes [S1].
*   **Strengths:** Preserves sharp features, extracts all disjoint components, adapts to local curvature, robust for complex topology [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-quality, curvature-adaptive meshes are needed from complex volumetric data, especially when sharp features must be preserved [S1].
*   **When not to use:** When uniform grid-based meshes are sufficient or when real-time performance is critical and the overhead of feature extraction and normal consistency checks is prohibitive [G].

## Background
The algorithm was proposed by Jian Xi and Ye Duan in 2008 [S1]. It is an improvement over the original Marching Triangles algorithm, which does not adapt well to local surface properties and requires many small triangles for high-curvature areas [S1]. The new method introduces a novel normal consistency constraint to address these limitations [S1].

## Sources
[S1] A novel region-growing based iso-surface extraction algorithm, https://doi.org/10.1016/j.cag.2008.09.007, fetched 2026-10-27
