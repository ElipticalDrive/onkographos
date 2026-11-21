# Hero-like 4-Connectivity (H4T)

## Overview
Hero-like 4-Connectivity (H4T) is a ray traversal algorithm for uniform grids that guarantees strict 4-connectivity by resolving boundary crossing ambiguities through a stable sort of candidate cell entries. It differs from standard DDA and its sibling [Coherent Grid Traversal](2987-coherent-grid-traversal.md) by explicitly testing a 2x2 cell neighborhood and using axis-priority sorting to prevent diagonal jumps, ensuring that rays crossing grid lines do so one axis at a time [S1, unofficial].

## How it works
H4T determines the sequence of voxels visited by a ray by identifying the entry point into a 2x2 block of cells and then advancing through them while maintaining connectivity. The method relies on a stable sorting mechanism to break ties between axis crossings.

**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, grid dimensions, voxel size $v$.
*   **Output:** A sequence of voxel indices $(i, j, k)$ visited by the ray.

**Data Structures:**
*   **Candidate List:** A list of tuples containing $(t, \text{axis\_id})$, where $t$ is the distance to the next boundary and $\text{axis\_id}$ identifies the axis ($0$ for $x$, $1$ for $y$). Note: The source specifies axis numbers 0 and 1, implying a 2D projection or specific handling for the primary axes in the 2x2 test context [S1, unofficial].

**Steps:**
1.  **2x2 Cell Test:** Identify the candidate cells the ray passes through in the current 2x2 block. This involves determining which of the four adjacent cells the ray enters first based on the ray's position relative to the grid boundaries [S1, unofficial].
2.  **Associate Axis IDs:** For each candidate boundary crossing distance $t$, associate it with an axis identifier. The source specifies storing $0$ for the $x$-axis and $1$ for the $y$-axis [S1, unofficial].
3.  **Stable Sort:** Sort the candidate list using a stable sorting algorithm. The sort key is primarily the axis identifier, and secondarily the $t$-value. This ensures that if $t$-values for the $x$ and $y$ axes are equal (a tie), the $x$-axis ($0$) is processed before the $y$-axis ($1$) [S1, unofficial].
4.  **Advance:** Select the first entry in the sorted list as the next cell. Update the ray's position and repeat the process for the next block. The grid position must be floored before reading the grid data, as the method may produce fractional position values during intermediate calculations [S1, unofficial].

**Pseudocode:**
The source notes that the C code is too long to include and describes the logic conceptually. The following pseudocode implements the described stable sort logic for the 2x2 test resolution.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def h4t_traversal(ray_origin, ray_direction, voxel_size, grid_bounds):
    # Determine initial voxel
    current_voxel = [int(ray_origin[i] // voxel_size) for i in range(3)]
    
    # Calculate distances to next boundaries for x and y (primary axes for 2x2 test)
    # Note: Standard DDA calculates t_max for all axes. H4T focuses on resolving 
    # the tie between x and y planes specifically.
    
    # Calculate t_max for x and y axes
    t_max_x = (current_voxel[0] + (1 if ray_direction[0] > 0 else 0)) * voxel_size - ray_origin[0]
    t_max_x /= ray_direction[0] if ray_direction[0] != 0 else float('inf')
    
    t_max_y = (current_voxel[1] + (1 if ray_direction[1] > 0 else 0)) * voxel_size - ray_origin[1]
    t_max_y /= ray_direction[1] if ray_direction[1] != 0 else float('inf')
    
    # Create candidate list with axis IDs: 0 for x, 1 for y
    candidates = []
    if ray_direction[0] != 0:
        candidates.append((t_max_x, 0))
    if ray_direction[1] != 0:
        candidates.append((t_max_y, 1))
        
    # Stable sort: first by axis_id, then by t-value
    # Python's sort is stable. We sort by (axis_id, t_value)
    candidates.sort(key=lambda x: (x[1], x[0]))
    
    # The first candidate is the next cell to enter
    # This ensures x-axis crossings are prioritized over y-axis crossings on ties
    next_t, next_axis = candidates[0]
    
    # Advance in the chosen axis
    step = 1 if ray_direction[next_axis] > 0 else -1
    current_voxel[next_axis] += step
    
    # Update t_max for the advanced axis
    if next_axis == 0:
        t_max_x += voxel_size / abs(ray_direction[0])
    else:
        t_max_y += voxel_size / abs(ray_direction[1])
        
    # Re-evaluate candidates for the next step
    # In a full implementation, this loop continues, updating candidates 
    # based on the new voxel position and re-sorting or merging the sorted lists.
    # The source emphasizes the sorting mechanism for tie-breaking.
    
    yield current_voxel
```

**Edge Cases and Pitfalls:**
*   **Ties:** The critical pitfall is the handling of equal $t$-values for different axes. Without stable sorting by axis ID, the traversal may jump diagonally, violating 4-connectivity. The source explicitly states that sorting by axis number first preserves 4-connectivity [S1, unofficial].
*   **Fractional Positions:** The method may result in fractional grid positions during calculation. The grid must be accessed using floored coordinates [S1, unofficial].
*   **Code Length:** The source notes that the full implementation is complex and lengthy, suggesting careful handling of the candidate list management [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Correctness:** H4T achieved 100% correctness for both position and normal in randomized testing (100,000,000 cases) and specific test cases (2,883,496 cases), outperforming HP4T and HP8T which had error rates of 99.99% and 97.66% respectively [S1, unofficial].
*   **Complexity:** The source does not provide explicit time or memory complexity metrics. However, the requirement for stable sorting of candidates suggests a slight overhead compared to simple min-selection, but this is negligible for the 2x2 test scope.
*   **Strengths:** Highest accuracy among tested methods; guarantees 4-connectivity.
*   **Limitations:** Implementation complexity is high; the source code was not published due to length [S1, unofficial].

## Background
H4T was developed as part of a project investigating ray traversal algorithms for hierarchical grids. It was discovered late in the project and is described as working "very differently" from previous methods like HP4T and HP8T. The method was tested using a C program compiled with GCC [S1, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
