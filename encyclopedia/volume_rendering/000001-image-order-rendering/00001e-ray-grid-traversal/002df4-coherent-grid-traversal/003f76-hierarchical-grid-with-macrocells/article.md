# Hierarchical Grid with Macrocells

Also called: Macrocell hierarchy, Two-level hierarchy

## Overview
A hierarchical grid structure that divides densely populated regions of space more finely than empty regions by imposing a coarser grid over the previous level, where each macrocell corresponds to an $M \times M \times M$ block of cells in the underlying level [S1]. It sets itself apart from the original Coherent Grid Traversal (CGT) by replacing simple Boolean empty/non-empty flags with parameter range culling, storing minimum and maximum values for multiple particle properties to enable more aggressive early-out tests [S1].

## How it works
**Inputs:** Particle dataset (position, radius, up to four additional properties); macrocell resolution $M$; grid dimensions.
**Outputs:** Hierarchical grid structure (macrocells containing bounds for underlying cells).

**Data Structures:**
1.  **Macrocell:** Stores minimum and maximum values for eight parameters: position $x, y, z$, radius, and up to four additional particle properties. This replaces the Boolean empty/non-empty flag [S1].
2.  **Underlying Grid:** Standard CGT grid cells, organized into blocks of size $M \times M \times M$ corresponding to macrocells [S1].

**Construction Algorithm (Sort-Middle):**
1.  Distribute construction tasks for disjoint sections of the grid statically among all system threads [S1].
2.  Each thread performs a coarse parallel bucket sort of particles by their cell location, inserting them into its local set of buckets [S1].
3.  Regions for each bucket are disjoint, avoiding write conflicts and the need for mutexes [S1].
4.  Each thread inserts its particles into the appropriate grid cells within its assigned disjoint region [S1].
5.  Aggregate the grid data to form macrocells. For each macrocell, compute the min/max of the eight parameters across all particles contained in its underlying $M \times M \times M$ block of cells [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_hierarchical_grid(particles, M, num_threads):
    # 1. Static distribution of grid sections to threads
    # Each thread t gets a disjoint set of grid cells to manage
    thread_tasks = distribute_grid_sections(num_threads)
    
    # 2. Parallel bucket sort per thread
    # Particles are sorted by cell location into local buckets
    # Buckets correspond to disjoint regions, avoiding write conflicts
    local_buckets = [Bucket() for _ in range(num_threads)]
    
    for particle in particles:
        cell_id = get_cell_id(particle.position)
        thread_id = get_thread_for_cell(cell_id, thread_tasks)
        local_buckets[thread_id].insert(particle)
        
    # 3. Insert into grid cells
    # Since regions are disjoint, no synchronization needed
    grid_cells = initialize_grid()
    for t in range(num_threads):
        for particle in local_buckets[t].particles:
            cell_id = get_cell_id(particle.position)
            grid_cells[cell_id].add_particle(particle)
            
    # 4. Build Macrocells
    # Each macrocell covers M x M x M cells
    macrocells = []
    for macrocell_coords in get_macrocell_coords(grid_dimensions, M):
        mc = Macrocell()
        # Initialize min/max for 8 parameters
        mc.min_vals = [infinity] * 8
        mc.max_vals = [-infinity] * 8
        
        # Iterate over underlying cells in this macrocell
        for cell in get_cells_in_macrocell(macrocell_coords, M):
            if cell.has_particles():
                for particle in cell.particles:
                    # Update min/max for position, radius, and 4 properties
                    for i in range(8):
                        val = get_parameter(particle, i)
                        mc.min_vals[i] = min(mc.min_vals[i], val)
                        mc.max_vals[i] = max(mc.max_vals[i], val)
        macrocells.append(mc)
        
    return macrocells, grid_cells
```

**Parameters:**
*   **Macrocell resolution ($M$):** The size of the macrocell block in cells. Typical value: $6 \times 6 \times 6$ [S1].
*   **Lambda ($\lambda$):** A parameter affecting performance in modified algorithms. Value $\lambda = 1$ provides best performance for tested datasets [S1].

**Edge Cases and Pitfalls:**
*   **Write Conflicts:** Avoided by using disjoint regions for each thread during construction [S1].
*   **Parameter Tuning:** Performance varies widely with $\lambda$ for large particle counts; $\lambda=1$ is recommended [S1].
*   **Frustum Overlap:** Ray packet size impacts performance; larger packets overlap more cells, increasing intersection cost per step [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Construction is $O(N)$ with parallel bucket sort, where $N$ is the number of particles, due to disjoint thread regions [S1]. Traversal benefits from parameter range culling, potentially reducing intersection tests compared to Boolean flags [S1].
*   **Memory Complexity:** Higher than original CGT due to storing 8 min/max values per macrocell instead of a single Boolean flag [S1].
*   **Performance:** $\lambda = 1$ and $M = 6 \times 6 \times 6$ yield reasonable performance for tested datasets [S1]. Performance is sensitive to $\lambda$ for very large particle counts [S1].
*   **Strengths:** Efficient traversal via parameter range culling; parallel construction without synchronization [S1].
*   **Limitations:** Increased memory usage per macrocell; sensitivity to parameter tuning for large datasets [S1].

## Background
Originated from the work of Parker et al. [17] and adapted for particle-based simulation data visualization by Ize et al. [9] and others [S1]. It builds on the original CGT algorithm but modifies the macrocell data structure to support more sophisticated culling [S1].

## Sources
[S1] A coherent grid traversal approach to visualizing particle-based simulation data, https://ieeexplore.ieee.org/abstract/document/4293019/, fetched 2026-10-25
