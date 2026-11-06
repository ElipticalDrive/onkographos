# Stack-Based Representation (SBR)

Also called: Stacks

## Overview
Stack-Based Representation (SBR) decomposes a voxel grid $V$ of resolution $w \times h \times d$ into a set $S$ of vertical stacks, compressing each column via run-length encoding of attribute values along the $z$-axis [S1, unofficial]. It solves the storage problem for layered volumetric data by exploiting vertical coherence, separating geometric layer boundaries from attribute values [S1, unofficial]. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Hash Grids](936-hash-grids.md) that rely on spatial hashing or tree traversal for arbitrary sparsity, SBR is specialized for datasets with strong vertical anisotropy, achieving linear construction complexity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid $V$ of resolution $w \times h \times d$, where each voxel $v_{x,y,z}$ stores one or more attributes (e.g., color, density, material) [S1, unofficial].
*   **Output:** A set $S$ of vertical stacks, one for each $(x,y)$ position in the grid [S1, unofficial].

**Data Structures:**
Each stack $S_{x,y}$ is a sequence of intervals $i_1, i_2, \dots, i_n$ sorted by height [S1, unofficial].
*   **Interval $i_k$:** A tuple $\langle a_k, h_k \rangle$ [S1, unofficial].
    *   $a_k$: The attribute value shared by all voxels in the interval [S1, unofficial].
    *   $h_k$: The cumulative height (1-based $z$-coordinate) of the top voxel of this interval [S1, unofficial].
*   **Implicit Geometry:** The interval $i_k$ covers voxels from $z = 1 + h_{k-1}$ to $z = h_k$ [S1, unofficial]. For the first interval ($k=1$), $h_0$ is assumed to be 0 [S1, unofficial].

**Formulas:**
The stack $S_{x,y}$ is the union of all voxels in the column:
$$S_{x,y} \cong V_{x,y} = \bigcup_{i=1}^d v_{x,y,i}$$
An interval $i_k$ represents the space of voxels with identical attribute $a_k$:
$$i_k \cong \bigcup_{i=1+h_{k-1}}^{h_k} v_{x,y,i}$$
Intervals are strictly sorted by height:
$$h_k < h_{k+1} \quad \text{for any given } k \text{ such that } 1 \le k \le n$$

**Algorithm:**
1. Iterate through every column $(x,y)$ for $x \in [0, w-1]$ and $y \in [0, h-1]$ [S1, unofficial].
2. Initialize `curr_attr` to the attribute of the first voxel $v_{x,y,0}$ and `curr_h` to 1 [G].
3. For each subsequent voxel index $z$ from 1 to $d-1$: [S1, unofficial].
    a. Let `new_attr` be the attribute of $v_{x,y,z}$ [S1, unofficial].
    b. If `new_attr` equals `curr_attr`, increment `curr_h` by 1 [G].
    c. Otherwise, append $\langle$ `curr_attr`, `curr_h` $\rangle$ to $S_{x,y}$ [S1, unofficial].
    d. Update `curr_attr` to `new_attr` and `curr_h` to $z + 1$ [G].
4. After the loop, append the final interval $\langle$ `curr_attr`, `curr_h` $\rangle$ to $S_{x,y}$ [G].
5. Store the completed stack $S_{x,y}$ in the set $S$ [S1, unofficial].

**Pitfalls:**
*   **Precision:** Height values $h_k$ must be stored with sufficient bits to represent the maximum depth $d$ [G].
*   **Boundary Conditions:** The first interval always starts at $z=1$ with $h_0=0$ [S1, unofficial].
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
            if depth == 0:
                stacks[(x, y)] = []
                continue
                
            intervals = []
            curr_attr = grid[x][y][0]
            curr_h = 1
            
            for z in range(1, depth):
                new_attr = grid[x][y][z]
                
                if new_attr == curr_attr:
                    curr_h += 1
                else:
                    intervals.append((curr_attr, curr_h))
                    curr_attr = new_attr
                    curr_h = z + 1
            
            intervals.append((curr_attr, curr_h))
            stacks[(x, y)] = intervals
            
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
*   **Memory:** Memory usage depends on the number of intervals per stack [S1, unofficial].
*   **Worst Case:** If attributes alternate every voxel, it uses more memory than the raw voxel grid due to interval overhead [S1, unofficial].
*   **Best Case:** In layered scenarios with uniform attributes, it is highly compact [S1, unofficial].
*   **Strengths:** Optimal for vertically layered data; simple construction; fast parallelization [S1, unofficial].
*   **Limitations:** Poor compression for data with frequent interval height variations; does not inherently handle arbitrary 3D sparsity without hierarchical extensions like QuadStack [S1, unofficial].

## Background
The Stack-Based Representation was introduced as a foundational method for compressing layered volumetric datasets [S1, unofficial]. It serves as the basis for more advanced structures like QuadStack [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
