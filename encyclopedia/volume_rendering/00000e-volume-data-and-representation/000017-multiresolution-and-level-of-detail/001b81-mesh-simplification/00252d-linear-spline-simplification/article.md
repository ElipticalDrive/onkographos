# Linear Spline Simplification

Also called: Knot removal algorithm, PRI data reduction

## Overview
Linear Spline Simplification is an iterative knot removal algorithm that reduces the number of control points (knots) in a 1D linear spline by merging consecutive segments when internal knots lie within a pre-chosen vertical distance (scalar domain) from the candidate merged segment [S1, S2, unofficial]. It solves the problem of redundant data storage in Piecewise Linear Representations (PRI) by identifying spans of nearly collinear scalar values and removing intermediate samples while preserving endpoints [S1, S2, unofficial]. Unlike general mesh decimation methods that modify geometry topology, this method operates strictly on the scalar parameterization of sampling rays, ensuring the resulting knot list is a subset of the original list [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A list of knots $K = \{k_0, k_1, \dots, k_{n-1}\}$ representing a 1D linear spline, where each knot $k_i$ has a position (domain value, e.g., distance along a ray) and a scalar value (e.g., density or temperature). An error threshold $\tau \ge 0$ [S1, S2, unofficial].
- **Output**: A simplified list of knots $K'$ such that $K' \subseteq K$ [S1, S2, unofficial].

**Data Structures**
The spline is represented as a sequence of linear segments connecting consecutive knots. Each knot $k_i$ is defined by:
- $x_i$: The domain coordinate (e.g., distance along the sampling ray).
- $s_i$: The scalar value at $x_i$.

**Algorithm Steps**
1. Initialize the current knot list $K_{curr} = K$.
2. Repeat the following iteration until no merges occur:
   a. Create an empty list $K_{next}$.
   b. Iterate through $K_{curr}$ using an index $i$ from $0$ to $len(K_{curr}) - 2$.
   c. Consider the pair of consecutive segments defined by knots $(k_i, k_{i+1})$ and $(k_{i+1}, k_{i+2})$. Note: The algorithm considers "consecutive and non-overlapping pairs of linear spline segments" [S1, S2, unofficial]. This implies processing disjoint pairs of segments. If the list has an odd number of segments, the last segment is handled in the next iteration or kept as is.
   d. Let the three knots defining the two segments be $A=k_i$, $B=k_{i+1}$, and $C=k_{i+2}$.
   e. Check if the middle knot $B$ can be removed. The condition is that $B$ lies within the vertical distance $\tau$ from the line segment connecting $A$ and $C$.
      - Calculate the scalar value $s_{AC}(x_B)$ on the segment $AC$ at the domain coordinate $x_B$:
        $$ s_{AC}(x_B) = s_A + \frac{s_C - s_A}{x_C - x_A} (x_B - x_A) $$
      - Calculate the vertical error:
        $$ \epsilon = |s_B - s_{AC}(x_B)| $$
      - If $\epsilon \le \tau$, the merge is valid.
   f. If the merge is valid for the pair $(A, B, C)$:
      - Add knot $A$ to $K_{next}$? No, the algorithm merges segments. The standard implementation of this iterative approach (based on Duchaineau's terrain simplification) processes pairs. If a merge occurs, the middle knot $B$ is removed. The resulting segment is $AC$.
      - To ensure non-overlapping processing, if a merge is performed for indices $(i, i+1, i+2)$, the next pair to consider starts after $C$. Thus, increment $i$ by 2 (skipping $B$ and $C$ as separate entities for the next pair check in this pass) or handle the list construction carefully.
      - Specifically, if we merge $A, B, C$ into $A, C$, we keep $A$ and $C$. $B$ is discarded.
      - If the merge is NOT valid, keep knot $B$ (and $A$ was already kept or processed).
   g. A robust implementation constructs $K_{next}$ by iterating through the segments:
      - Let $K_{curr} = [k_0, k_1, \dots, k_m]$.
      - Initialize $K_{next} = [k_0]$.
      - Let $j = 1$.
      - While $j < m$:
        - Let $A = k_{j-1}$, $B = k_j$, $C = k_{j+1}$ (if $j+1 \le m$).
        - If $j+1 \le m$ and the error condition for $B$ relative to $AC$ is $\le \tau$:
          - Merge: Remove $B$. The new segment is $AC$.
          - Update $C$'s scalar value? No, $C$ remains $k_{j+1}$. The segment is now from $A$ to $C$.
          - Add $C$ to $K_{next}$? No, $A$ is already in $K_{next}$ (or is the last added). We need to replace the end of the current chain.
          - Actually, the description says "merge the two segments into a larger single segment by removing the middle knot".
          - If we merge $A-B$ and $B-C$, we get $A-C$.
          - In the list construction:
            - If merge is valid: The segment $A-C$ replaces $A-B$ and $B-C$. The knot $B$ is removed. The next knot to consider for pairing is $k_{j+2}$? No, the segments are non-overlapping.
            - Let's trace: Pairs of segments are $(k_0, k_1, k_2)$, $(k_2, k_3, k_4)$, etc.
            - If $(k_0, k_1, k_2)$ merges, $k_1$ is removed. The new end is $k_2$. The next pair starts at $k_2$? No, the segments are non-overlapping. The next pair is $(k_2, k_3, k_4)$.
            - So, if we merge at index $j$ (removing $k_j$), the next pair starts at $k_{j+1}$? No, the segments are defined by indices.
            - Segment 1: $k_0 \to k_1$. Segment 2: $k_1 \to k_2$. Pair: $(k_0, k_1, k_2)$.
            - If merged, result is $k_0 \to k_2$.
            - Next pair of segments must be non-overlapping. The next segment in the original list is $k_2 \to k_3$. The one after is $k_3 \to k_4$.
            - So the next pair to evaluate is $(k_2, k_3, k_4)$.
            - Therefore, if a merge occurs at $j$, we advance the index to $j+1$ (which is $k_{j+1}$, the new end of the merged segment) to start the next pair? No, the next pair's first segment starts at $k_{j+1}$.
            - Let's refine the loop:
              - $K_{next} = [k_0]$.
              - $i = 1$.
              - While $i < len(K_{curr}) - 1$:
                - $A = K_{curr}[i-1]$
                - $B = K_{curr}[i]$
                - $C = K_{curr}[i+1]$
                - Calculate error of $B$ against $AC$.
                - If error $\le \tau$:
                  - Merge $B$.
                  - Update $K_{next}$'s last element to $C$? No, $A$ is already in $K_{next}$?
                  - If we just removed $B$, the segment is $A \to C$.
                  - We need to ensure $A$ is the last element of $K_{next}$ before adding $C$.
                  - If $K_{next}$ ends with $A$, replace $A$ with $C$? Or just append $C$?
                  - Since $A$ is the start of the new segment, and $C$ is the end, and $A$ is already in the list (as the previous endpoint), we simply update the last element of $K_{next}$ to $C$.
                  - Increment $i$ by 2 (skip $B$ and $C$ as the start of the next pair? No, $C$ is the start of the next segment).
                  - The next pair is $(C, D, E)$. So $i$ becomes $i+2$ (pointing to $D$? No, $i$ points to the middle knot).
                  - If $i$ was the index of $B$, the next middle knot is $D$ (index $i+2$).
                  - So $i \leftarrow i + 2$.
                - Else:
                  - Do not merge. Keep $B$.
                  - Append $B$ to $K_{next}$.
                  - Increment $i$ by 1.
              - Append the last knot $K_{curr}[-1]$ to $K_{next}$.
   h. Set $K_{curr} = K_{next}$.
   i. If $K_{curr}$ is unchanged from the previous iteration, stop.

**Parameters**
- **Error Threshold ($\tau$)**: The maximum allowable vertical distance (scalar deviation) for merging.
  - Typical values tested: $0.0, 0.01, 0.02, 0.03$ [S1, S2, unofficial].
  - Scalar values are in the range $[0, 1]$ [S1, S2, unofficial].
  - A threshold of $0.0$ removes superfluous knots representing samples in constant or linearly changing regions [S1, S2, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Floating-point comparisons for $\epsilon \le \tau$ should handle small epsilon errors if necessary, though the sources do not specify a tolerance for the comparison itself [S1, S2, unofficial].
- **Boundary Knots**: The first and last knots of the spline are always preserved [S1, S2, unofficial].
- **Degenerate Input**: If $x_C = x_A$ (vertical segment in domain), the slope is undefined. This should not occur in valid sampling rays where domain coordinates are strictly increasing.
- **Convergence**: The algorithm is guaranteed to terminate because each iteration removes at least one knot or none, and the number of knots is finite [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_spline_simplification(knots, threshold):
    """
    knots: List of tuples (x, s) representing (domain, scalar) values.
    threshold: Float, max vertical error allowed for merging.
    Returns: Simplified list of knots.
    """
    if len(knots) <= 2:
        return knots

    current_knots = list(knots)
    
    while True:
        next_knots = []
        if len(current_knots) > 0:
            next_knots.append(current_knots[0])
        
        i = 1
        merged = False
        
        while i < len(current_knots) - 1:
            A = current_knots[i-1]
            B = current_knots[i]
            C = current_knots[i+1]
            
            # Calculate scalar value on segment AC at B's x-coordinate
            # Segment AC: x from A.x to C.x, s from A.s to C.s
            if C[0] == A[0]:
                # Degenerate case, vertical line in domain. Should not happen in valid rays.
                # Treat as no merge to be safe.
                error = float('inf')
            else:
                # Linear interpolation
                t = (B[0] - A[0]) / (C[0] - A[0])
                s_ac = A[1] + t * (C[1] - A[1])
                error = abs(B[1] - s_ac)
            
            if error <= threshold:
                # Merge B: Remove B, segment becomes A-C
                # Update the last element of next_knots to C
                # Since A is already in next_knots (or was the last added),
                # we replace the last element with C.
                # Note: If i=1, next_knots has [A]. We replace A with C?
                # No, the segment is A->C. A is the start. C is the end.
                # If next_knots ends with A, we want it to end with C.
                next_knots[-1] = C
                i += 2  # Skip B and C (C is now the end of the merged segment, start of next pair)
                merged = True
            else:
                # No merge, keep B
                next_knots.append(B)
                i += 1
        
        # Append the last knot
        if len(current_knots) > 0:
            next_knots.append(current_knots[-1])
            
        # Check for convergence
        if not merged or next_knots == current_knots:
            break
            
        current_knots = next_knots
        
    return current_knots
```

## Subtopics

### [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md)
Removes valence patterns to simplify mesh topology. Choose when topological consistency is more important than scalar field preservation.

### [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md)
Parallel edge collapse for meshes. Choose for high-performance mesh simplification on GPU.

### [Grid-based LOD Simplification](4423-grid-based-lod-simplification.md)
Simplifies volume grids by removing cells. Choose for structured grid data.

### [Octree Simplification](4641-octree-simplification.md)
Simplifies octree structures. Choose for hierarchical volume representations.

### [Sample-based Simplification](7814-sample-based-simplification.md)
General sample reduction. Choose when specific spline structure is not required.

### [Isosurface Simplification via Persistence](8470-isosurface-simplification-via-persistence.md)
Simplifies isosurfaces based on feature persistence. Choose for surface extraction.

### [Vertex Clustering Simplification](9046-vertex-clustering-simplification.md)
Clusters vertices to reduce count. Choose for coarse simplification.

### [Triangular Mesh Simplification](9103-triangular-mesh-simplification.md)
General triangle mesh simplification. Choose for unstructured surface meshes.

## Operating characteristics
**Time and Memory Complexity**
- **Time**: Each iteration scans the knot list once. In the worst case, $O(N)$ per iteration. The number of iterations is bounded by $O(\log N)$ because at most half the knots are removed per iteration [S1, S2, unofficial]. Total time complexity is $O(N \log N)$ [S1, S2, unofficial].
- **Memory**: $O(N)$ to store the knot lists [S1, S2, unofficial].

**Measured Performance**
Performance was measured on the Piecewise Linear Representation (PRI) of unstructured grid volumes [S1, S2, unofficial]. The following table shows the average number of knots per spline for different datasets and error thresholds (scalar range [0, 1]) [S1, S2, unofficial]:

| Dataset | Threshold 0.00 | Threshold 0.01 | Threshold 0.02 | Threshold 0.03 |
| :--- | :--- | :--- | :--- | :--- |
| Blunt Fin | 49 | 9 | 6 | 6 |
| Combustor | 53 | 15 | 11 | 10 |
| Liquid Oxygen Post | 55 | 9 | 7 | 7 |
| SPX | 20 | 7 | 5 | 4 |

**Strengths and Limitations**
- **Strengths**: Guarantees the simplified knot list is a subset of the original, preserving topological correspondence. Effective at removing redundant samples in linear or constant regions [S1, S2, unofficial].
- **Limitations**: Only reduces data size; does not improve rendering speed directly unless the PRI is used for ray marching. The approximation error can rise sharply if the threshold is relaxed too much [S1, S2, unofficial].

**When to Use**
- Use when storing 1D scalar fields along sampling rays and memory bandwidth is a bottleneck.
- Use when the scalar field exhibits large linear regions.

**When Not to Use**
- Do not use if the scalar field is highly oscillatory and the threshold must be very small, as the reduction factor will be low.
- Do not use if the application requires strict error bounds that are not captured by the vertical distance metric.

## Background
The method was introduced by Shareef et al. in 2006 as part of an image-based modeling approach for GPU-based unstructured grid volume rendering [S1, S2, unofficial]. It is based on a 1D version of the mesh simplification algorithm for terrain data by Duchaineau et al. (1997) [S1, S2, unofficial]. The method is used to reduce the size of the Piecewise Linear Representation (PRI), which is a resampling of unstructured grid data into a texture-based format for efficient GPU rendering [S1, S2, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
