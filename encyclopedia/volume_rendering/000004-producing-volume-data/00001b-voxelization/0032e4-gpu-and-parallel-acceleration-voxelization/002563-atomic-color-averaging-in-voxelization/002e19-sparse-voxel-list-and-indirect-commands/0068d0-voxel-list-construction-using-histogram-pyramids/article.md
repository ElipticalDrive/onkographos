# Voxel List Construction Using Histogram Pyramids

Also called: Histogram pyramid traversal, Parallel voxel list initialization, Voxel List Construction via Histogram Pyramid Traversal

## Overview
This method initializes a sparse voxel list on the GPU by mapping linear indices of non-zero voxels back to their spatial coordinates [S1, unofficial]. It solves the parallel coordinate reconstruction problem by using a histogram pyramid to perform a deterministic traversal that maps each linear index to a specific voxel location [S1, unofficial].

## How it works
**Inputs:**
- Input volume data $V$ of size $m \times m \times m$ (or $m \times m$ in 2D) [S1, unofficial].
- A binary classification mask $B$ where $B_{ijk} = 1$ if $V_{ijk} \neq 0$, else $0$ [S1, unofficial].

**Outputs:**
- A voxel list $L$ containing $n$ elements, where $n$ is the count of non-zero voxels [S1, unofficial].
- Each element in $L$ stores the Morton code (z-order) of the corresponding voxel's spatial location [S1, unofficial].

**Data Structures:**
1.  **Histogram Pyramid $H$:** A hierarchical structure storing partial sums of the binary mask $B$ [S1, unofficial].
    -   **Levels:** $0$ to $\lceil \log_2 m \rceil$ [S1, unofficial].
    -   **Level 0 (Leaves):** Contains $m^2$ (or $m^3$) entries, each holding the value of $B_{ijk}$ (0 or 1) [S1, unofficial].
    -   **Level $k > 0$:** Contains entries representing the sum of children from level $k-1$ [S1, unofficial].
    -   **Layout:** Stored as a flat array hierarchy where parent nodes sum their children [S1, unofficial].
2.  **Voxel List $L$:** An array of size $n$ storing Morton codes [S1, unofficial].

**Algorithm Steps:**

1.  **Binary Classification:**
    Generate the binary mask $B$ from input volume $V$. $B_{ijk} = 1$ if $V_{ijk} \neq 0$, else $0$ [S1, unofficial].

2.  **Construct Histogram Pyramid:**
    Perform a parallel reduction on $B$ to build $H$ [S1, unofficial].
    -   Initialize level 0 with $B$.
    -   For each level $k$ from 1 to $\lceil \log_2 m \rceil$:
        -   Each node at level $k$ sums the values of its children from level $k-1$ [S1, unofficial].
        -   In 2D, a node sums four children; in 3D, eight children [S1, unofficial].
    -   Read the root value to determine $n$, the total number of non-zero voxels [S1, unofficial].

3.  **Parallel Traversal Kernel:**
    Launch a GPU kernel with $n$ work-items (threads) [S1, unofficial]. Each thread $t$ (where $t \in [0, n-1]$) is responsible for finding the spatial coordinate of the $t$-th non-zero voxel [S1, unofficial].

    **Traversal Logic for Thread $t$ (Target Index $i = t$):**
    -   Start at the root of $H$.
    -   Maintain a current coordinate interval $[min, max]$ (initially covering the entire volume bounds) [S1, unofficial].
    -   Iterate from level $k = \lceil \log_2 m \rceil$ down to 1:
        -   Iterate over the children of the current node in Morton order (z-order) [S1, unofficial].
        -   Accumulate the sum $S$ of the children's values.
        -   Check if $S > i$.
        -   If $S > i$:
            -   The target voxel is within the subtree of the current child that caused the sum to exceed $i$ [S1, unofficial].
            -   Move down to that child node.
            -   Update the coordinate interval $[min, max]$ to the bounds of this child [S1, unofficial].
            -   Update $i$ to $i - (\text{sum of previous siblings})$ [S1, unofficial].
        -   If $S \le i$:
            -   Subtract $S$ from $i$ (since the target is not in this subtree).
            -   Continue to the next sibling [S1, unofficial].
    -   **Leaf Level:** When reaching the leaf level (level 0), the traversal stops when the index $i$ is just exceeded by the summation of values [S1, unofficial]. The final coordinate interval identifies the specific voxel $(x, y, z)$ [S1, unofficial].

4.  **Morton Code Conversion:**
    -   Convert the final spatial coordinates $(x, y, z)$ obtained from the traversal into a Morton code [S1, unofficial].
    -   Store this Morton code in the voxel list $L[t]$ [S1, unofficial].
    -   The transformation uses standard bit-interleaving [G].

5.  **Sorting:**
    -   Sort the elements of the list in Morton order (z-order) to exploit sorting in the look-up phase for binary search [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_histogram_pyramid(binary_mask, m):
    """
    Builds a histogram pyramid from a binary mask of size m x m.
    Returns the pyramid levels as a list of arrays.
    """
    levels = [binary_mask.flatten()]
    while len(levels[-1]) > 1:
        prev = levels[-1]
        curr = [0] * (len(prev) // 4)
        for i in range(len(curr)):
            idx = i * 4
            curr[i] = prev[idx] + prev[idx+1] + prev[idx+2] + prev[idx+3]
        levels.append(curr)
    return levels

def voxel_list_construction(input_volume, m):
    # 1. Binary Classification
    binary_mask = (input_volume != 0).astype(int)

    # 2. Construct Histogram Pyramid
    levels = build_histogram_pyramid(binary_mask, m)
    n = levels[-1][0]
    voxel_list = [0] * n

    # 3. Parallel Traversal (Simulated sequentially)
    for t in range(n):
        remaining = t
        current_node_idx = 0
        current_level_idx = len(levels) - 1
        min_x, max_x = 0, m - 1
        min_y, max_y = 0, m - 1

        while current_level_idx > 0:
            level = levels[current_level_idx]
            found = False
            # Iterate over children in Morton order
            for j in range(current_node_idx, len(level)):
                val = level[j]
                if remaining < val:
                    current_node_idx = j
                    found = True
                    break
                remaining -= val
            if not found:
                raise ValueError("Target index out of bounds")

            # Update coordinate intervals based on Morton order quadrant [G]
            offset = current_node_idx % 4
            dx = (max_x - min_x + 1) // 2
            dy = (max_y - min_y + 1) // 2
            if offset == 0: # Bottom-Left
                max_x -= dx; max_y += dy
            elif offset == 1: # Bottom-Right
                min_x += dx; max_y += dy
            elif offset == 2: # Top-Left
                max_x -= dx; min_y -= dy
            else: # Top-Right
                min_x += dx; min_y -= dy

            current_level_idx -= 1

        # Leaf level reached
        x = min_x
        y = min_y
        # 4. Morton Code Conversion [G]
        morton = interleave_bits(x, y)
        voxel_list[t] = morton

    # 5. Sorting
    voxel_list.sort()
    return voxel_list

def interleave_bits(x, y):
    # Standard bit-interleaving for Morton code [G]
    x = (x | (x << 16)) & 0x0000FFFF0000FFFF
    x = (x | (x << 8)) & 0x00FF00FF00FF00FF
    x = (x | (x << 4)) & 0x0F0F0F0F0F0F0F0F
    x = (x | (x << 2)) & 0x3333333333333333
    x = (x | (x << 1)) & 0x5555555555555555
    y = (y | (y << 16)) & 0x0000FFFF0000FFFF
    y = (y | (y << 8)) & 0x00FF00FF00FF00FF
    y = (y | (y << 4)) & 0x0F0F0F0F0F0F0F0F
    y = (y | (y << 2)) & 0x3333333333333333
    y = (y | (y << 1)) & 0x5555555555555555
    return x | (y << 1)
```

**Edge Cases and Pitfalls:**
-   **Empty Volume:** If $n=0$, the kernel launches with 0 work-items. The histogram pyramid root is 0 [S1, unofficial].
-   **Precision:** The histogram values are integers (counts), so no floating-point precision issues arise during summation [S1, unofficial].
-   **Boundary Conditions:** The traversal correctly handles the final element by stopping when the index is just exceeded by the leaf summation [S1, unofficial].
-   **Morton Order:** The traversal must strictly follow Morton order (z-order) to ensure the spatial locality required for the binary search property [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(n)$ for the parallel traversal, where $n$ is the number of non-zero voxels. The histogram construction is $O(m^2)$ (or $O(m^3)$) for a volume of side length $m$. The traversal depth is $\log(m)$ [S1, unofficial].
-   **Memory Complexity:** $O(m^2)$ (or $O(m^3)$) to store the histogram pyramid. The voxel list requires $O(n)$ space [S1, unofficial].
-   **Performance:** The method is designed for GPU parallel execution. It avoids the contention of atomic operations. The source notes that the traversal takes $\log(m)$ steps per element [S1, unofficial].
-   **Strengths:** Deterministic, efficient for sparse data, enables fast binary search lookups after sorting [S1, unofficial].
-   **Limitations:** Requires storing the full histogram pyramid. The initial construction cost is non-trivial for large volumes [S1, unofficial].

## Background
This method was introduced by Ziegler for GPU volume dataset data structures [S1, unofficial]. It builds upon the concept of histogram pyramids for parallel reduction and applies it to the specific problem of coordinate reconstruction in voxel lists [S1, unofficial]. The use of Morton codes for sorting and lookup is a standard technique in spatial indexing, adapted here to facilitate the traversal [S1, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
