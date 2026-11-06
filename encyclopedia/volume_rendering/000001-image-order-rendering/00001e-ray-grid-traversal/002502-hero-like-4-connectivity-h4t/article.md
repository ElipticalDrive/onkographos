# Hero-like 4-Connectivity (H4T)

## Overview
Hero-like 4-Connectivity (H4T) is a ray traversal algorithm for uniform grids that guarantees strict 4-connectivity by explicitly resolving boundary crossing ambiguities via stable sorting of intersection distances. It solves the problem of diagonal jumps (8-connectivity errors) and associated normal inaccuracies that occur when a ray crosses an edge or corner in standard Digital Differential Analyzer (DDA) approaches [S1, unofficial]. Unlike its sibling [Hierarchical Planes 4-Connectivity (HP4T)](3444-hierarchical-planes-4-connectivity-hp4t.md), which uses plane intersections and clamping, H4T relies on a 2x2 cell test and stable sorting to determine the first cell entered, achieving 100% correctness in randomized testing [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, grid dimensions $N_x, N_y, N_z$, voxel size $v$, and grid bounds.
**Outputs:** Sequence of visited voxel indices $(i, j, k)$ and traversal intervals.

**Data Structures:**
*   `t_values`: Array of floating-point distances to the next boundary for the relevant axes.
*   `axis_ids`: Array of integers mapping each `t_value` to its axis (0 for x, 1 for y).
*   `grid_position`: Current voxel coordinates $(i, j, k)$.

**Algorithm Steps:**
1.  **Initialization:** Determine the starting voxel containing the ray origin. Compute initial distances to the first boundary in the x and y axes (and z if applicable, though H4T emphasizes the 2D plane logic for connectivity resolution). Let $t_x$ be the distance to the next x-boundary and $t_y$ be the distance to the next y-boundary.
2.  **2x2 Cell Test:** Identify the candidate cells the ray passes through by evaluating the intersection distances. This involves checking which of the four cells sharing the current edge/corner is entered first.
3.  **Association:** Associate each distance value $t$ with an axis identifier.
    *   If the distance corresponds to an x-boundary, assign `axis_id = 0`.
    *   If the distance corresponds to a y-boundary, assign `axis_id = 1`.
4.  **Stable Sorting:** Sort the pairs $(t, \text{axis\_id})$ using a stable sorting algorithm.
    *   Primary key: `axis_id` (ascending).
    *   Secondary key: $t$ (ascending).
    *   *Note:* A stable sort preserves the relative order of elements with equal keys. By sorting first by axis, ties in $t$ between x and y boundaries are resolved by prioritizing the x-axis (axis 0) over the y-axis (axis 1). This ensures that if the ray hits an edge where $t_x = t_y$, the x-plane is processed first, maintaining 4-connectivity [S1, unofficial].
5.  **Traversal Update:**
    *   Select the first entry in the sorted list as the next boundary to cross.
    *   Update the `grid_position` index for the selected axis.
    *   Update the corresponding $t$ value for that axis by adding the step size $\Delta t$.
    *   *Precision Note:* The `grid_position` may contain fractional values during intermediate calculations and must be floored before reading the grid data [S1, unofficial].
6.  **Iteration:** Repeat steps 2-5 until the ray exits the grid bounds.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def h4t_traverse(ray_origin, ray_direction, grid_bounds, voxel_size):
    # 1. Initialize starting voxel and boundary distances
    start_voxel = floor(ray_origin / voxel_size)
    # Calculate distances to next boundaries for x and y
    # t_x: distance to next x-plane
    # t_y: distance to next y-plane
    t_x = compute_distance_to_boundary(ray_origin.x, ray_direction.x, start_voxel.x, voxel_size)
    t_y = compute_distance_to_boundary(ray_origin.y, ray_direction.y, start_voxel.y, voxel_size)
    
    # 2. Prepare for sorting
    # List of tuples: (distance, axis_id)
    # axis_id: 0 for x, 1 for y
    candidates = []
    
    # Add current candidates
    candidates.append((t_x, 0))
    candidates.append((t_y, 1))
    
    # 3. Stable Sort
    # Sort by axis_id first, then by distance.
    # Python's sort is stable. We sort by (axis_id, distance).
    candidates.sort(key=lambda item: (item[1], item[0]))
    
    # 4. Select next boundary
    next_t, next_axis = candidates[0]
    
    # 5. Update state
    if next_axis == 0:
        # Crossed x-boundary
        current_voxel.x += step_x(ray_direction.x)
        t_x += delta_t_x(ray_direction.x, voxel_size)
    else:
        # Crossed y-boundary
        current_voxel.y += step_y(ray_direction.y)
        t_y += delta_t_y(ray_direction.y, voxel_size)
        
    # 6. Re-evaluate candidates for next step
    # In a full implementation, this loop continues, updating candidates
    # based on the new voxel position and remaining distances.
    # The core H4T logic is the sorting step to resolve ties.
    
    return current_voxel
```

**Edge Cases and Pitfalls:**
*   **Ties:** When $t_x = t_y$, standard DDA might arbitrarily pick one, leading to 8-connectivity. H4T forces x-first via sorting [S1, unofficial].
*   **Fractional Positions:** The method may produce fractional grid positions during traversal; these must be floored before texture/voxel lookup [S1, unofficial].
*   **Performance:** The sorting step adds computational overhead compared to simple comparisons. The source notes the C code was omitted due to length, implying complexity [S1, unofficial].

## Subtopics

### [Hierarchical Planes 4-Connectivity (HP4T)](3444-hierarchical-planes-4-connectivity-hp4t.md)
A hierarchical variant that replaces stable sorting with direct plane intersection and clamping. Chosen when hierarchical grid structures (like Octrees) are used and higher precision `total_t` calculation is needed, though it trades the 100% correctness of H4T for hierarchical efficiency.

## Operating characteristics
*   **Correctness:** Achieved 100% correctness for both position and normal in 100,000,000 randomized test cases and 2,883,496 exact test cases [S1, unofficial].
*   **Accuracy:** Superior to HP4T (99.99% position/normal correctness) and HP8T (97.66% position/99.29% normal correctness) [S1, unofficial].
*   **Complexity:** The stable sorting step introduces overhead. The source does not provide explicit Big-O notation but notes the implementation complexity is high enough that the source code was not published [S1, unofficial].
*   **Use Case:** Best suited for applications where geometric correctness and normal accuracy are critical and performance overhead of sorting is acceptable.

## Background
H4T was developed as part of a project investigating ray traversal algorithms for hierarchical grids. It was discovered late in the project cycle. The method was designed to address the connectivity errors inherent in previous approaches like HP4T and HP8T. The research was conducted by a student at KTH Royal Institute of Technology, as documented in a thesis [S1, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
