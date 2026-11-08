# Stack-Based Representation (SBR)
Also called: Stacks
## Overview
Stack-Based Representation (SBR) is a sparse volumetric data structure that decomposes a voxel grid into vertical columns, or "stacks," where each stack is compacted using run-length encoding along the vertical axis [S1, unofficial]. It solves the problem of storing layered volumetric data by exploiting vertical coherence, storing only the transitions in attribute values (such as density or color) rather than individual voxels [S1, unofficial]. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Hash Grids](936-hash-grids.md) that rely on spatial hashing or uniform subdivision, SBR is inherently anisotropic, optimizing specifically for datasets with strong vertical layering, and serves as the foundational block for hierarchical variants like [QuadStack](1113-quadstack.md) [S1, 1081-stack-based-representation-sbr.md, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid $V$ of resolution $w \times h \times d$, where each voxel $v_{x,y,z}$ stores one or more attributes (e.g., color, material, density) [S1, unofficial].
*   **Output:** A set $S$ of vertical stacks, where each stack $S_{x,y}$ corresponds to a column at position $(x,y)$ [S1, unofficial].

**Data Structures:**
Each stack $S_{x,y}$ is represented as a sequence of intervals $i_1, i_2, \dots, i_n$ sorted by height [S1, unofficial].
*   **Interval ($i_k$):** A tuple $\langle a_k, h_k \rangle$.
    *   $a_k$: The attribute value shared by all voxels in the interval.
    *   $h_k$: The cumulative height (z-coordinate) of the top voxel of the interval [S1, unofficial].
*   **Implicit Geometry:** The interval $i_k$ covers voxels from $z = 1 + h_{k-1}$ to $z = h_k$. For the first interval ($k=1$), $h_0$ is assumed to be 0 [S1, unofficial].

**Formulas:**
The stack is the union of voxels in the column:
$$S_{x,y} \cong V_{x,y} = \bigcup_{i=1}^d v_{x,y,i}$$

An interval $i_k$ represents the space of voxels with identical attribute value $a_k$:
$$i_k \cong \bigcup_{i=1+h_{k-1}}^{h_k} v_{x,y,i}$$

Intervals are strictly sorted by height:
$$h_k < h_{k+1} \quad \text{for any given } k \text{ such that } 1 \le k \le n$$

**Algorithm:**
1.  Iterate through each column $(x,y)$ in the grid $w \times h$.
2.  For each column, iterate through $z$ from 1 to $d$.
3.  Compare the attribute value of the current voxel $v_{x,y,z}$ with the previous voxel $v_{x,y,z-1}$ (or a default null value for $z=1$).
4.  If the attribute value changes:
    *   Finalize the current interval with the height $z-1$.
    *   Start a new interval with the new attribute value.
5.  After processing $z=d$, finalize the last interval with height $d$.
6.  Store the sequence of intervals for the stack $S_{x,y}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sbr(voxel_grid, width, height, depth):
    stacks = {}
    for x in range(width):
        for y in range(height):
            intervals = []
            current_attr = None
            current_height = 0
            
            for z in range(1, depth + 1):
                # Get attribute value for voxel (x, y, z)
                attr = voxel_grid.get_attribute(x, y, z)
                
                # Check if attribute changes from previous voxel
                # For z=1, previous is implicitly different (or null)
                if attr != current_attr:
                    # If we had a previous interval, record its end height
                    if current_attr is not None:
                        intervals.append({
                            'attr': current_attr,
                            'height': current_height
                        })
                    
                    # Start new interval
                    current_attr = attr
                    current_height = z
            
            # Finalize the last interval
            if current_attr is not None:
                intervals.append({
                    'attr': current_attr,
                    'height': depth
                })
            
            stacks[(x, y)] = intervals
    return stacks
```

**Edge Cases and Pitfalls:**
*   **Precision:** Interval heights $h_k$ must be stored with sufficient bits to represent the maximum depth $d$ [S1, unofficial].
*   **Boundaries:** The algorithm assumes a 1-based index for $z$ in the mathematical definition, but implementation may use 0-based indexing; consistency must be maintained [S1, unofficial].
*   **Degenerate Input:** If a column is entirely empty or has a single uniform attribute, the stack contains a single interval [S1, unofficial].
*   **Parallelism:** Construction is embarrassingly parallel as stack construction does not require information about neighboring stacks [S1, unofficial].

## Subtopics

### [QuadStack](1113-quadstack.md)
Organizes similar stacks into a quadtree to compress regions with constant layer topology, achieving higher compression rates than flat SBR by exploiting attribute uniformity across neighboring columns.

### [Group of Stacks (gstack)](1169-group-of-stacks-gstack.md)
Represents a rectangular region of vertical stacks sharing identical attribute sequences and interval counts, compressing common attributes while storing varying geometry as non-intersecting heightfields.

## Operating characteristics
*   **Time Complexity:** Construction is $O(n)$, where $n = w \times h \times d$, because each voxel is processed exactly once [S1, unofficial].
*   **Memory Complexity:** Depends on the number of layer transitions. In the worst case (alternating attributes every voxel), memory usage equals or exceeds the raw voxel grid. In the best case (single uniform attribute per column), memory is proportional to the number of columns $w \times h$ [S1, unofficial].
*   **Strengths:** Embarrassingly parallel construction; highly efficient for layered datasets with strong vertical coherence; simple data structure [S1, unofficial].
*   **Limitations:** Poor compression for isotropic or horizontally layered data; no inherent spatial locality for neighboring stacks in the flat representation [S1, unofficial].
*   **When to use:** Layered volumetric datasets (e.g., geological strata, atmospheric data) where vertical coherence is high [S1, unofficial].
*   **When not to use:** Isotropic volumes or data with complex 3D structures without vertical bias [S1, unofficial].

## Background
The Stack-Based Representation was introduced in the context of efficient compression and direct rendering of layered datasets. It was developed by researchers at the Czech Technical University in Prague (DCGI) and presented in the paper "QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets" [S1, unofficial]. The method exploits the observation that while stack heights may vary, attribute values often remain consistent across neighboring columns, a principle further exploited by its child structures [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
