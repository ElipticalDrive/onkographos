# Block Subdivision

Also called: Block-based volume partitioning, Disjoint block processing

## Overview
Block subdivision partitions a volumetric particle set into disjoint spatial blocks to enable sequential, memory-bounded surface extraction while maintaining a seamless global result [S1]. It differs from global hashing approaches by limiting the active memory footprint to a single block's cache, reducing overhead to approximately 12% for typical configurations, and enabling independent multi-threaded rendering of each block [S1].

## How it works
**Inputs:**
*   List of particles, each with position and radius of influence $R_c$.
*   Block dimensions in cubes: $W \times H \times D$.
*   Cube size: $S$.
*   Isosurface threshold: $T$.
*   Margin width in cubes: $M$ (determined by $R_c$ and $S$).

**Outputs:**
*   Vertex and index buffers for the isosurface mesh.

**Data Structures:**
*   **Block List:** A collection of disjoint blocks covering the volume.
*   **Particle Cache:** For each block, a list of particles within distance $R_c$ of the block boundary, plus all particles inside the block. This cache accelerates field value calculations [S1].

**Parameters:**
*   $R_c$: Radius of influence of a particle. Determines the margin size and particle inclusion range [S1].
*   $S$: Size of a cube (voxel edge length) [S1].
*   $T$: Threshold value for surface generation [S1].
*   $M$: Margin width in cubes. Calculated such that the physical margin covers distance $R_c$. Typically $M \approx \lceil R_c / S \rceil$ [S1].

**Steps:**
1.  **Partition Volume:** Divide the global volume containing particles into a grid of disjoint blocks of size $W \times H \times D$ cubes.
2.  **Build Particle Lists:** For each block, identify all particles located within the block. Additionally, identify all particles located outside the block but within a distance $R_c$ from the block's boundary. Insert these external particles into the block's particle list to ensure field consistency at the boundaries [S1].
3.  **Expand Geometry:** Conceptually expand the block's processing volume by a margin of $M$ cubes in all directions. This expanded volume encloses the region where particles from the margin might influence the interior field values [S1].
4.  **Process Blocks Sequentially:** Iterate through each block in the list:
    a.  **Initialize Cache:** Load the block's particle list into a lookup cache [S1].
    b.  **Marching Slices:** Execute the marching slices algorithm over the expanded volume (original block + margin).
    c.  **Field Calculation:** For each cube in the expanded volume, calculate the field value $f(r)$ based on nearby particles in the cache. The field function is defined as:
        $$
        f(r) = \begin{cases} 
        (r/\sqrt{2}R_c)^4 - (r/\sqrt{2}R_c)^2 + 0.25 & \text{if } r < R_c \\
        0 & \text{otherwise}
        \end{cases}
        $$
        where $r$ is the distance from the sample point to the particle [S1].
    d.  **Boundary Instrumentation:** To guarantee the marching slices algorithm can traverse the closed isosurface, force field values to zero at sample points on the outer boundary of the margin. This is achieved by instrumenting the lookup cache to exclude particles when sampling points lie exactly on the margin boundary [S1].
    e.  **Mesh Generation:** Generate vertices and indices for cubes where the isosurface intersects. Do not output mesh triangles for cubes located in the margin region [S1].
    f.  **Buffer Output:** Write the generated vertices and indices to the user-specified buffers [S1].
5.  **Repeat:** Continue until all blocks are processed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_subdivision(particles, block_dims, cube_size, threshold, radius_influence):
    """
    particles: List of Particle objects with pos and radius.
    block_dims: Tuple (W, H, D) in number of cubes.
    cube_size: Float S.
    threshold: Float T.
    radius_influence: Float Rc.
    """
    margin_cubes = int(math.ceil(radius_influence / cube_size))
    blocks = create_block_grid(particles, block_dims, cube_size)
    
    for block in blocks:
        # Step 2: Build particle list for this block
        block_particles = []
        for p in particles:
            if is_within_block(p.pos, block.bounds):
                block_particles.append(p)
            elif distance_to_block_boundary(p.pos, block.bounds) <= radius_influence:
                block_particles.append(p)
        
        # Step 3: Define expanded bounds
        expanded_bounds = expand_bounds(block.bounds, margin_cubes * cube_size)
        
        # Step 4: Process block
        vertices = []
        indices = []
        
        # Iterate over all cubes in the expanded volume
        for cube in cubes_in_volume(expanded_bounds, cube_size):
            # Check if cube is in margin
            is_margin = is_in_margin(cube, block.bounds, margin_cubes)
            
            # Calculate field values for corners
            field_values = []
            for corner in cube.corners:
                # Instrumentation: Force field to 0 on outer margin boundary
                if is_on_outer_margin_boundary(corner, expanded_bounds):
                    field_val = 0.0
                else:
                    field_val = 0.0
                    for p in block_particles:
                        r = distance(corner, p.pos)
                        if r < radius_influence:
                            # Field function from [S1]
                            norm_r = r / (math.sqrt(2) * radius_influence)
                            field_val += (norm_r**4 - norm_r**2 + 0.25)
                field_values.append(field_val)
            
            # Marching Slices logic (simplified)
            if not is_margin:
                # Generate mesh for this cube if it intersects isosurface
                mesh = marching_slices_cube(field_values, threshold, cube.pos, cube_size)
                vertices.extend(mesh.vertices)
                indices.extend(mesh.indices)
            else:
                # Margin cubes: only need sign check for traversal, no output
                pass
                
        write_to_buffers(vertices, indices)
```

**Edge Cases and Pitfalls:**
*   **Closed Isosurface Assumption:** The marching slices algorithm assumes a closed surface. If particles in the margin are not handled correctly, traversal may fail. Forcing field values to zero at the outer margin boundary resolves this [S1].
*   **Seams:** Varying cube size $S$ between blocks for Level of Detail (LOD) control can cause visible seams. Care must be taken to ensure compatibility at block boundaries [S1].
*   **Precision:** Field values are summed from multiple particles. Precision issues may arise if many particles contribute to a single sample point, though the specific numerical stability is not detailed in the sources [S1].

## Operating characteristics
*   **Time Complexity:** The overhead of blocking is approximately 12% of total work for randomly distributed particles in a typical 100x100x100 block with a 5-cube margin [S1]. This is significantly lower than hashing approaches, which require hashing for both corner retrieval and particle lookups [S1].
*   **Memory Complexity:** Memory footprint is limited to a single block's particle cache and corner values, making it suitable for large volumes that do not fit in memory [S1].
*   **Performance:** For a 100x100x100 block with a 5-cube margin, 25% of the cubes in the expanded 110x110x110 volume are margin cubes. Traversing margin cubes costs approximately half that of normal cubes since only the sign of the field is needed and no vertices are generated [S1].
*   **Strengths:**
    *   Low memory usage [S1].
    *   Enables multi-threaded rendering (one thread per block) [S1].
    *   Efficient for static scenes: blocks with no moving particles can be redrawn by re-outputting old buffers without polygonization [S1].
    *   Supports visibility culling and LOD control [S1].
*   **Limitations:**
    *   Requires sequential processing of blocks if memory is strictly limited, though parallelization is possible across blocks [S1].
    *   Margin overhead increases with larger $R_c$ relative to block size [S1].
*   **When to use:** Surface modeling with moving particles, large unbounded volumes, hardware with limited memory per thread/buffer, or when visibility culling is beneficial [S1].
*   **When not to use:** When global hashing is more efficient for the specific particle distribution, or when strict real-time constraints prevent the overhead of block management (though the overhead is small) [S1].

## Background
Block subdivision was introduced in the context of real-time particle isosurface extraction [S1]. It addresses the challenge of rendering large volumes of particles without exceeding memory limits, by partitioning the problem into manageable, independent chunks [S1]. The method builds upon the marching slices algorithm, which processes the volume slice by slice to further reduce memory usage [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
