# Linear Spline Simplification

Also called: Knot removal algorithm, PRI data reduction

## Overview
Linear Spline Simplification is an iterative knot removal algorithm that reduces the number of control points (knots) in a 1D linear spline by merging consecutive segments when internal knots lie within a pre-chosen vertical distance (scalar domain) from the candidate merged segment [S1, unofficial][S2, unofficial]. It solves the problem of redundant data storage in Piecewise Linear Representations (PRI) by identifying spans of nearly collinear scalar values and removing intermediate samples while preserving endpoints [S1, unofficial][S2, unofficial]. The resulting list of knots is a subset of the original list [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A list of knots $K = \{k_0, k_1, \dots, k_{n-1}\}$ representing a 1D linear spline, where each knot $k_i$ has a domain coordinate $x_i$ and a scalar value $s_i$ [G]. An error threshold $\tau \ge 0$ [S1, unofficial][S2, unofficial].
- **Output**: A simplified list of knots $K'$ such that $K' \subseteq K$ [S1, unofficial][S2, unofficial].

**Data Structures**
The spline is represented as a sequence of linear segments connecting consecutive knots. Each knot $k_i$ is defined by a tuple $(x_i, s_i)$ [G].

**Algorithm Steps**
1. Initialize the current knot list $K_{curr} = K$ [G].
2. Repeat the following iteration until no merges occur:
   a. Create an empty list $K_{next}$ and append the first knot $k_0$ to it [G].
   b. Iterate through $K_{curr}$ using an index $i$ from $1$ to $len(K_{curr}) - 2$ [G].
   c. Consider the pair of consecutive segments defined by knots $(k_{i-1}, k_i)$ and $(k_i, k_{i+1})$ [S1, unofficial][S2, unofficial]. The algorithm considers "consecutive and non-overlapping pairs of linear spline segments" [S1, unofficial][S2, unofficial].
   d. Let $A = k_{i-1}$, $B = k_i$, and $C = k_{i+1}$ [G].
   e. Check if the middle knot $B$ can be removed. The condition is that $B$ lies within the vertical distance $\tau$ from the line segment connecting $A$ and $C$ [S1, unofficial][S2, unofficial].
      - Calculate the scalar value $s_{AC}(x_B)$ on the segment $AC$ at the domain coordinate $x_B$:
        $$ s_{AC}(x_B) = s_A + \frac{s_C - s_A}{x_C - x_A} (x_B - x_A) $$
      - Calculate the vertical error:
        $$ \epsilon = |s_B - s_{AC}(x_B)| $$
      - If $\epsilon \le \tau$, the merge is valid [S1, unofficial][S2, unofficial].
   f. If the merge is valid for the pair $(A, B, C)$:
      - Remove knot $B$. The resulting segment is $AC$ [S1, unofficial][S2, unofficial].
      - To ensure non-overlapping processing, if a merge is performed, the next pair to consider starts after $C$. Thus, increment $i$ by 2 [S1, unofficial][S2, unofficial].
      - Update the last element of $K_{next}$ to $C$ [G].
   g. If the merge is NOT valid, keep knot $B$. Append $B$ to $K_{next}$ and increment $i$ by 1 [S1, unofficial][S2, unofficial].
   h. After the loop, append the last knot $K_{curr}[-1]$ to $K_{next}$ only if it was not already added as $C$ in the final merge step [G].
   i. Set $K_{curr} = K_{next}$.
   j. If $K_{curr}$ is unchanged from the previous iteration, stop [S1, unofficial][S2, unofficial].

**Parameters**
- **Error Threshold ($\tau$)**: The maximum allowable vertical distance (scalar deviation) for merging. Typical values tested: $0.0, 0.01, 0.02, 0.03$ [S1, unofficial][S2, unofficial]. Scalar values are in the range $[0, 1]$ [S1, unofficial][S2, unofficial]. A threshold of $0.0$ removes superfluous knots representing samples in constant or linearly changing regions [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls**
- **Precision**: Floating-point comparisons for $\epsilon \le \tau$ should handle small epsilon errors if necessary, though the sources do not specify a tolerance for the comparison itself [S1, unofficial][S2, unofficial].
- **Boundary Knots**: The first and last knots of the spline are always preserved [S1, unofficial][S2, unofficial].
- **Degenerate Input**: If $x_C = x_A$ (vertical segment in domain), the slope is undefined. This should not occur in valid sampling rays where domain coordinates are strictly increasing [G].
- **Convergence**: The algorithm is guaranteed to terminate because each iteration removes at least one knot or none, and the number of knots is finite [S1, unofficial][S2, unofficial].

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
        next_knots = [current_knots[0]]
        i = 1
        merged = False
        
        while i < len(current_knots) - 1:
            A = current_knots[i-1]
            B = current_knots[i]
            C = current_knots[i+1]
            
            if C[0] == A[0]:
                error = float('inf')
            else:
                t = (B[0] - A[0]) / (C[0] - A[0])
                s_ac = A[1] + t * (C[1] - A[1])
                error = abs(B[1] - s_ac)
            
            if error <= threshold:
                next_knots[-1] = C
                i += 2
                merged = True
            else:
                next_knots.append(B)
                i += 1
        
        if next_knots[-1] != current_knots[-1]:
            next_knots.append(current_knots[-1])
            
        if not merged or next_knots == current_knots:
            break
            
        current_knots = next_knots
        
    return current_knots
```

## Operating characteristics
**Time and Memory Complexity**
- **Time**: Each iteration scans the knot list once. In the worst case, $O(N)$ per iteration. The number of iterations is bounded by $O(\log N)$ because at most half the knots are removed per iteration [S1, unofficial][S2, unofficial]. Total time complexity is $O(N \log N)$ [S1, unofficial][S2, unofficial].
- **Memory**: $O(N)$ to store the knot lists [S1, unofficial][S2, unofficial].

**Measured Performance**
Performance was measured on the Piecewise Linear Representation (PRI) of unstructured grid volumes [S1, unofficial][S2, unofficial]. The following table shows the average number of knots per spline for different datasets and error thresholds (scalar range $[0, 1]$) [S1, unofficial][S2, unofficial]:

| Dataset | Threshold 0.00 | Threshold 0.01 | Threshold 0.02 | Threshold 0.03 |
| :--- | :--- | :--- | :--- | :--- |
| Blunt Fin | 49 | 9 | 6 | 6 |
| Combustor | 53 | 15 | 11 | 10 |
| Liquid Oxygen Post | 55 | 9 | 7 | 7 |
| SPX | 20 | 7 | 5 | 4 |

**Strengths and Limitations**
- **Strengths**: Guarantees the simplified knot list is a subset of the original, preserving topological correspondence. Effective at removing redundant samples in linear or constant regions [S1, unofficial][S2, unofficial].
- **Limitations**: Only reduces data size; does not improve rendering speed directly unless the PRI is used for ray marching [G]. The approximation error can rise sharply if the threshold is relaxed too much [S1, unofficial][S2, unofficial].

**When to Use**
- Use when storing 1D scalar fields along sampling rays and memory bandwidth is a bottleneck [G].
- Use when the scalar field exhibits large linear regions [S1, unofficial][S2, unofficial].

**When Not to Use**
- Do not use if the scalar field is highly oscillatory and the threshold must be very small, as the reduction factor will be low [G].
- Do not use if the application requires strict error bounds that are not captured by the vertical distance metric [G].

## Background
The method was introduced by Shareef et al. in 2006 as part of an image-based modeling approach for GPU-based unstructured grid volume rendering [S1, unofficial][S2, unofficial]. It is based on a 1D version of the mesh simplification algorithm for terrain data by Duchaineau et al. (1997) [S1, unofficial][S2, unofficial]. The method is used to reduce the size of the Piecewise Linear Representation (PRI), which is a resampling of unstructured grid data into a texture-based format for efficient GPU rendering [S1, unofficial][S2, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
