# Sphere-Center Method

Also called: Sphere-center grid

## Overview
The Sphere-Center Method is an optimization for glyph-based particle visualization within the Coherent Grid Traversal (CGT) framework that eliminates data duplication and redundant intersection tests by storing each particle in exactly one grid cell [S1, S2]. It achieves this by traversing an enlarged frustum that accounts for the maximum particle radius, ensuring that any sphere intersecting the original view volume has its center contained within the traversal bounds [S1, S2]. Unlike standard grid approaches that require mailboxes or reference lists to handle primitives overlapping multiple cells, this method guarantees single storage and single intersection per particle [S1, S2].

## How it works
The method exploits the property that particle radii are either uniform or fall within a well-defined range [S2]. By using the maximum radius across all particles, the algorithm constructs a frustum large enough to contain the centers of all potentially intersecting spheres, thereby allowing a simple "center-in-cell" storage scheme [S1, S2].

### Inputs and Outputs
*   **Inputs:**
    *   Particle dataset: A collection of spheres defined by center coordinates $(x, y, z)$ and radius $r$.
    *   Ray packet frustum $F$: Defined by bounding planes and near/far clipping planes.
    *   Grid resolution: Determines the cell size in world units.
*   **Outputs:**
    *   A grid structure where each sphere is stored in exactly one cell.
    *   An enlarged frustum $F_r$ used for traversal.

### Data Structures
*   **Grid:** A multi-level grid structure (as defined in [Coherent Grid Traversal](2987-coherent-grid-traversal.md)).
*   **Cell Content:** Each cell stores a list of particle indices or direct particle data. Unlike standard grids, no duplicate entries or mailbox structures are required [S1, S2].
*   **Particle Storage:** Particles are stored directly in the finest level of the grid. The conversion from floating-point center coordinates to grid cell indices is performed via truncation [S1].

### Parameters
*   $r_{max}$: The maximum radius across all particles in the dataset. This value is determined once during initialization and cached for subsequent build and traversal steps [S1].
*   $dU$: The direction vector component of the normal to a bounding plane of the frustum [S2].

### Steps

1.  **Initialization:**
    *   Compute $r_{max}$ by iterating over all particles and finding the maximum radius [S1, S2].
    *   Cache $r_{max}$ for use in grid construction and traversal [S1].

2.  **Grid Construction:**
    *   For each particle with center $C = (x, y, z)$, compute its grid cell coordinates by truncating the floating-point values of its center in the grid coordinate space to integers [S1].
    *   Store the particle in the list of the identified cell [S1, S2].
    *   This process replaces expensive primitive/cell or bounding box/cell overlap tests with a single float-to-int truncation per particle [S1].

3.  **Frustum Enlargement:**
    *   Compute the shift distance $s$ for the side bounding planes of the frustum. The distance $s$ is calculated based on $r_{max}$ and the plane normal direction $dU$ [S2]:
        $$s = r_{max} \sqrt{1 + dU^2}$$
    *   Derivation: The condition that a sphere of radius $r_{max}$ intersects a bounding plane defined by normal direction $dU$ is equivalent to the sphere's center being within distance $r_{max}$ of the plane. Geometrically, this requires shifting the plane by $s$ such that the projection of the shift vector onto the normal equals $r_{max}$. The scalar projection formula yields $r_{max} = \frac{\langle -dU, 1 \rangle \cdot \langle 0, s \rangle}{|\langle -dU, 1 \rangle|}$, which solves to $s = r_{max} \sqrt{1 + dU^2}$ [S1, S2].
    *   Adjust the bounding planes: Add $s$ to the $u_0$ parameter for the maximum bounding plane and subtract $s$ for the minimum bounding plane [S1, S2].
    *   Adjust the near and far planes: Shift the near plane by $r_{max}$ in the $-\mathbf{K}$ direction and the far plane by $r_{max}$ in the $+\mathbf{K}$ direction, where $\mathbf{K}$ is the viewing direction [S1, S2].
    *   The resulting structure is the enlarged frustum $F_r$ [S1, S2].

4.  **Traversal:**
    *   Traverse the grid using the enlarged frustum $F_r$ instead of the original frustum $F$ [S1, S2].
    *   For each cell visited, intersect the rays in the packet against all spheres stored in that cell [S1, S2].
    *   Because the frustum is enlarged by $r_{max}$, any sphere intersecting the original frustum $F$ is guaranteed to have its center inside $F_r$. Thus, intersecting only spheres whose centers are in the visited cells covers all potential intersections [S1, S2].

5.  **Early Ray Termination:**
    *   Adjust early ray termination criteria to account for spheres whose centers lie in the next slice but whose geometry extends into the current slice due to the enlarged frustum [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_center_method(particles, ray_packet_frustum, grid_resolution):
    """
    Implements the Sphere-Center Method for particle visualization.
    
    Args:
        particles: List of objects with attributes .center (vec3) and .radius (float).
        ray_packet_frustum: Object defining the frustum with planes and near/far bounds.
        grid_resolution: Float representing the size of one grid cell.
        
    Returns:
        grid: A dictionary or array mapping cell coordinates to lists of particle indices.
        enlarged_frustum: The modified frustum for traversal.
    """
    
    # Step 1: Initialization
    # Find the maximum radius across all particles
    r_max = 0.0
    for p in particles:
        if p.radius > r_max:
            r_max = p.radius
            
    # Step 2: Grid Construction
    # Initialize empty grid structure
    grid = {}
    
    for i, p in enumerate(particles):
        # Convert center to grid coordinates by truncation
        # Note: Assumes grid origin is at (0,0,0) and axes are aligned.
        # In a general case, transform center to grid space first.
        gx = int(p.center.x / grid_resolution)
        gy = int(p.center.y / grid_resolution)
        gz = int(p.center.z / grid_resolution)
        
        cell_key = (gx, gy, gz)
        
        # Store particle index in the cell
        if cell_key not in grid:
            grid[cell_key] = []
        grid[cell_key].append(i)
        
    # Step 3: Frustum Enlargement
    # Compute shift distance s for side planes
    # dU is the direction vector component of the normal to the bounding plane
    # Assuming dU is normalized or defined as per the source's coordinate system
    # The source defines dU as the direction vector of the given plane's normal.
    # For a plane normal n = (nx, ny, nz), if the plane is defined by u = u0 + k*dU,
    # dU represents the lateral direction.
    # The formula s = r_max * sqrt(1 + dU^2) applies to the lateral shift.
    
    # Example: For a plane with normal having lateral component dU
    dU = ray_packet_frustum.plane_normal_lateral_component 
    s = r_max * math.sqrt(1.0 + dU * dU)
    
    # Adjust bounding planes
    # Add s to u0 for max plane, subtract s for min plane
    enlarged_frustum = copy(ray_packet_frustum)
    enlarged_frustum.max_plane_u0 += s
    enlarged_frustum.min_plane_u0 -= s
    
    # Shift near and far planes
    # Shift near plane by r_max in -K direction
    # Shift far plane by r_max in +K direction
    # K is the viewing direction vector
    K = ray_packet_frustum.view_direction
    enlarged_frustum.near_plane -= r_max * K
    enlarged_frustum.far_plane += r_max * K
    
    return grid, enlarged_frustum

def traverse_enlarged_frustum(grid, enlarged_frustum, ray_packet):
    """
    Traverses the grid using the enlarged frustum and intersects particles.
    """
    # Standard CGT traversal logic using enlarged_frustum
    # For each cell intersected by enlarged_frustum:
    for cell in cells_intersected_by(enlarged_frustum, grid):
        # Intersect rays in packet against all particles in this cell
        for particle_idx in grid[cell]:
            particle = particles[particle_idx]
            for ray in ray_packet.rays:
                intersect(ray, particle)
```

### Edge Cases and Pitfalls
*   **Wide Radius Range:** If particle radii span several orders of magnitude, using $r_{max}$ causes the enlarged frustum to be excessively large. This results in traversing many empty cells and intersecting many unnecessary spheres, potentially outweighing the benefits of single storage [S1]. Frustum culling is typically required to mitigate this performance loss [S1].
*   **Non-Uniform Radii:** The method assumes radii are uniform or within a small range. If this assumption is violated, the performance gain diminishes [S1, S2].
*   **Plane Normal Definition:** The calculation of $s$ depends on the definition of $dU$. The sources define $dU$ as the direction vector of the given plane's normal in a reduced 2D context where the normal is $\langle -dU, 1 \rangle$. Implementers must ensure $dU$ is extracted correctly from the frustum plane definitions [S1, S2].

## Subtopics

### Macrocell Hierarchical CGT
Uses a macrocell hierarchy to skip empty space efficiently within the CGT framework. Choose when the scene has large regions of empty space and hierarchical skipping is beneficial. See [Macrocell Hierarchical CGT](5464-macrocell-hierarchical-cgt.md).

### Slice Parallelisation
Parallelizes the processing of slices within the CGT traversal. Choose when hardware supports parallel execution across slices. See [Slice Parallelisation](6292-slice-parallelisation.md).

### Bundle Parallelisation
Parallelizes ray packets across multiple cores. Choose when ray-level parallelism is preferred over slice-level. See [Bundle Parallelisation](6293-bundle-parallelisation.md).

### SIMD Sphere/Frustum Culling
Uses SIMD instructions to efficiently cull spheres against frusta. Choose when CPU SIMD capabilities are available to accelerate culling operations. See [SIMD Sphere/Frustum Culling](10012-simd-sphere-frustum-culling.md).

## Operating characteristics
*   **Time Complexity:** Grid construction is $O(N)$ where $N$ is the number of particles, due to the simple truncation operation [S1]. Traversal complexity depends on the number of cells visited by the enlarged frustum and the number of spheres per cell.
*   **Memory Complexity:** Memory footprint is reduced significantly because each particle is stored exactly once. For the BulletTorso dataset (35M particles), storage dropped from 3.11 GB (with references) to 1.04 GB [S1].
*   **Measured Performance:**
    *   Speed-up over standard grid: 1.02x to 1.27x on various datasets (Cylinder, JP8, Bullet, Thunder, Foam, BulletTorso) [S1].
    *   Grid construction time improvement: Up to 3.07x faster insertion phase (e.g., JP8 dataset: 797.71 ms to 260.13 ms) [S1].
*   **Strengths:**
    *   Eliminates data duplication and mailbox overhead [S1, S2].
    *   Improves locality of reference [S1].
    *   Simplifies grid construction by avoiding overlap tests [S1].
    *   Efficient frustum computation using SIMD (5 operations) [S1, S2].
*   **Limitations:**
    *   Performance degrades if particle radii vary widely [S1].
    *   Requires a priori knowledge of $r_{max}$ [S1].
    *   Enlarged frustum may traverse more cells than necessary if $r_{max}$ is large relative to particle density [S1].
*   **When to Use:** Ideal for particle-based simulation data with uniform or narrowly distributed radii, where interactive performance and low memory footprint are critical [S1, S2].
*   **When Not to Use:** When particle radii vary by several orders of magnitude, unless combined with effective frustum culling [S1].

## Background
The Sphere-Center Method was introduced in the context of interactive ray tracing for particle-based simulation data, specifically for visualizing material point method (MPM) simulations [S2]. It was presented as an optimization to the Coherent Grid Traversal (CGT) algorithm to address classic grid problems: duplicate data, lack of locality of reference, and redundant intersection tests [S1]. The method draws inspiration from offset surfaces used in collision detection [S2]. It was evaluated on datasets such as Cylinder, JP8, Bullet, Thunder, Foam, and BulletTorso, demonstrating improvements in both frame rate and memory usage [S1]. The approach is applicable to other primitive types, such as triangles, though primarily designed for spherical glyphs [S1].

## Sources
[S1] A coherent grid traversal approach to visualizing particle-based simulation data, https://ieeexplore.ieee.org/abstract/document/4293019/, fetched 2026-10-25
[S2] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
