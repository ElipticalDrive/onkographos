# Stack-Based Representation (SBR)

Also called: Stacks

## Overview
Stack-Based Representation (SBR) is a sparse volume data structure that decomposes a voxel grid into vertical columns, compressing each column via run-length encoding of attribute values along the depth axis [S1, unofficial]. It solves the problem of storing layered volumetric data by exploiting vertical coherence, separating geometric layer boundaries from attribute values [S1, unofficial]. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Hash Grids](936-hash-grids.md) that rely on spatial hashing or tree traversal for arbitrary sparsity, SBR is specialized for datasets with strong vertical anisotropy, such as terrain or layered scenes, achieving linear construction complexity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid $V$ of resolution $w \times h \times d$, where each voxel $v_{x,y,z}$ stores one or more attributes (e.g., color, density, material) [S1, unofficial].
*   **Output:** A set $S$ of vertical stacks, one for each $(x,y)$ position in the grid [S1, unofficial].

**Data Structures:**
Each stack $S_{x,y}$ is a sequence of intervals $i_1, i_2, \dots, i_n$ sorted by height [S1, unofficial].
*   **Interval $i_k$:** A tuple $\langle a_k, h_k \rangle$ [S1, unofficial].
    *   $a_k$: The attribute value (e.g., material ID, color) shared by all voxels in the interval [S1, unofficial].
    *   $h_k$: The cumulative height (z-coordinate) of the top voxel of this interval [S1, unofficial].
*   **Implicit Geometry:** The interval $i_k$ covers voxels from $z = 1 + h_{k-1}$ to $z = h_k$ [S1, unofficial]. For the first interval ($k=1$), $h_0$ is assumed to be 0 [S1, unofficial].

**Formulas:**
The stack $S_{x,y}$ is the union of all voxels in the column:
$$S_{x,y} \cong V_{x,y} = \bigcup_{i=1}^d v_{x,y,i}$$
An interval $i_k$ represents the space of voxels with identical attribute $a_k$:
$$i_k \cong \bigcup_{i=1+h_{k-1}}^{h_k} v_{x,y,i}$$
Intervals are strictly sorted by height:
$$h_k < h_{k+1} \quad \text{for any given } k \text{ such that } 1 \le k \le n$$

**Algorithm:**
1.  Iterate through every voxel $v_{x,y,z}$ in the grid $V$ for $x \in [0, w-1]$, $y \in [0, h-1]$, $z \in [0, d-1]$ [S1, unofficial].
2.  For each column $(x,y)$, maintain a current attribute `curr_attr` and current height `curr_h` [G]. Initialize `curr_attr` to the attribute of the first voxel $v_{x,y,0}$ and `curr_h` to 0 [G].
3.  For each subsequent voxel $v_{x,y,z}$ (where $z > 0$):
    *   Let `new_attr` be the attribute of $v_{x,y,z}$ [S1, unofficial].
    *   If `new_attr` == `curr_attr`:
        *   Increment `curr_h` by 1 [G].
    *   Else:
        *   Append a new interval $\langle \text{curr\_attr}, \text{curr\_h} \rangle$ to the stack $S_{x,y}$ [S1, unofficial].
        *   Update `curr_attr` = `new_attr` [G].
        *   Update `curr_h` = $z$ [G].
4.  After processing all $z$ for a column, append the final interval $\langle \text{curr\_attr}, \text{curr\_h} \rangle$ to $S_{x,y}$ [G].
5.  Store the resulting stack $S_{x,y}$ in the set $S$ [S1, unofficial].

**Pitfalls:**
*   **Precision:** Height values $h_k$ must be stored with sufficient bits to represent the maximum depth $d$ [S1, unofficial].
*   **Boundary Conditions:** The first interval always starts at $z=1$ (or $z=0$ in 0-indexed logic) with $h_0=0$ [S1, unofficial].
*   **Degenerate Input:** If a column contains only one attribute value, the stack contains a single interval with height $d$ [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sbr(grid, width, height, depth):
    """
    Builds the Stack-Based Representation for a voxel grid.
    grid: 3D array [width][height][depth] storing attribute values.
    Returns: A dictionary mapping (x, y) to a list of intervals.
    """
    stacks = {}
    
    for x in range(width):
        for y in range(height):
            stack_intervals = []
            if depth == 0:
                stacks[(x, y)] = stack_intervals
                continue
                
            # Initialize with the first voxel
            current_attr = grid[x][y][0]
            current_h = 0
            
            for z in range(1, depth):
                new_attr = grid[x][y][z]
                
                if new_attr == current_attr:
                    # Extend the current interval
                    current_h = z
                else:
                    # Finalize the previous interval and start a new one
                    stack_intervals.append((current_attr, current_h))
                    current_attr = new_attr
                    current_h = z
            
            # Append the final interval for this column
            stack_intervals.append((current_attr, current_h))
            stacks[(x, y)] = stack_intervals
            
    return stacks
```

## Subtopics

### [QuadStack](1113-quadstack.md)
Extends SBR by organizing stacks into a quadtree to compress regions with identical interval sequences, achieving higher compression for layered data.

### [Group of Stacks (gstack)](1169-group-of-stacks-gstack.md)
A building block for QuadStack that groups rectangular regions of stacks sharing identical attributes and interval counts, compressing geometry as heightfields.

## Operating characteristics
*   **Time Complexity:** Construction is $O(n)$, where $n = w \times h \times d$, because each voxel is processed exactly once [S1, unofficial].
*   **Parallelism:** Construction is embarrassingly parallel as stack construction for one column does not depend on neighboring columns [S1, unofficial].
*   **Memory:** Memory usage depends on the number of layers (intervals) per stack. In the worst case (alternating attributes every voxel), it uses more memory than the raw voxel grid due to interval overhead. In best-case layered scenarios, it is highly compact [S1, unofficial].
*   **Strengths:** Optimal for vertically layered data (e.g., terrain, geological strata); simple construction; fast parallelization [S1, unofficial].
*   **Limitations:** Poor compression for isotropic or horizontally layered data; does not inherently handle arbitrary 3D sparsity without hierarchical extensions like QuadStack [S1, unofficial].

## Background
The Stack-Based Representation was introduced as a foundational method for compressing layered volumetric datasets [S1, unofficial]. It was developed to address the inefficiency of standard voxel storage for data with strong vertical coherence, such as terrain models or atmospheric layers [S1, unofficial]. The method serves as the basis for more advanced structures like QuadStack [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
