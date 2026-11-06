# Linear Spline Simplification

Also called: Knot removal algorithm, PRI data reduction

## Overview
Linear Spline Simplification is an iterative algorithm that reduces the number of knots in a piecewise linear spline by merging consecutive segments when internal knots lie within a specified vertical distance (scalar error) from the candidate merged segment [S1, unofficial]. It is used in the context of Piecewise Rational Interpolation (PRI) for unstructured grid volume rendering to identify spans of nearly collinear scalar values, thereby removing redundant internal samples while preserving endpoints [S1, unofficial]. Unlike general mesh decimation methods that may introduce new vertices, this method ensures the resulting knot list is a strict subset of the original list [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A piecewise linear spline defined by a sequence of knots $K = \{k_0, k_1, \dots, k_n\}$, where each knot $k_i$ has a scalar domain value $s_i$. An error threshold $\epsilon \ge 0$.
*   **Output:** A simplified spline with a subset of the original knots.

**Data Structures:**
The spline is represented as a list of knots. Each knot is a pair $(x_i, s_i)$, where $x_i$ is the position in the parametric domain (e.g., arc length or index) and $s_i$ is the scalar value. The sources do not specify a complex bit-packing scheme for the knots themselves, but note that in the PRI implementation, knots are stored in a 2D "knots texture" and referenced by an "index texture" [S1, unofficial]. For the simplification algorithm itself, a simple list or array is sufficient.

**Algorithm Steps:**
1.  **Initialization:** Let the current list of knots be $K_{current} = K_{original}$.
2.  **Iteration:** Repeat the following until no merges occur in a full pass:
    a.  Create a new list $K_{next}$ initialized with the first knot of $K_{current}$.
    b.  Iterate through $K_{current}$ using an index $i$ from $0$ to $n-2$ (where $n$ is the number of knots in $K_{current}$).
    c.  Consider the pair of consecutive segments defined by knots $k_i, k_{i+1}, k_{i+2}$.
    d.  Define the candidate merged segment connecting $k_i$ and $k_{i+2}$.
    e.  Check if the middle knot $k_{i+1}$ and any other knots lying strictly between $k_i$ and $k_{i+2}$ in the original spline (if any were skipped in previous iterations, though the algorithm processes consecutive pairs in the current list) are within the vertical distance $\epsilon$ from the candidate segment.
        *   Specifically, for each knot $k_j$ such that $i < j < i+2$ (in the current list context, this is just $j=i+1$), calculate the vertical distance $d_j$ from $k_j$ to the line segment $\overline{k_i k_{i+2}}$.
        *   The vertical distance is the absolute difference in scalar values between the actual scalar $s_j$ and the scalar value $s_{proj}$ interpolated linearly on the segment $\overline{k_i k_{i+2}}$ at position $x_j$.
        *   $s_{proj} = s_i + \frac{s_{i+2} - s_i}{x_{i+2} - x_i} (x_j - x_i)$.
        *   $d_j = |s_j - s_{proj}|$.
    f.  If $d_j \le \epsilon$ for all such $j$, merge the segments:
        *   Do not add $k_{i+1}$ to $K_{next}$.
        *   Skip $k_{i+1}$ and proceed to consider the next pair starting from $k_{i+2}$ (effectively removing $k_{i+1}$).
        *   Update the current index to $i+2$ (or handle the loop to skip the next element).
    g.  If the condition is not met, keep $k_{i+1}$:
        *   Add $k_{i+1}$ to $K_{next}$.
        *   Proceed to the next pair.
    h.  Add the last knot of $K_{current}$ to $K_{next}$.
    i.  Set $K_{current} = K_{next}$.
3.  **Termination:** The algorithm stops when a full iteration results in no merges (i.e., $K_{next}$ is identical to $K_{current}$).

**Parameters:**
*   **Error Threshold ($\epsilon$):** The maximum allowable vertical scalar deviation. Typical values tested in the source are $0.0, 0.01, 0.02, 0.03$. Scalar values are in the range $[0, 1]$ [S1, unofficial].
*   **Note:** An error threshold of $0.0$ removes superfluous knots that lie in constant or linearly changing regions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_spline_simplification(knots, epsilon):
    """
    Simplify a piecewise linear spline by removing knots within epsilon vertical distance.
    
    Args:
        knots: List of tuples (x, s) representing the spline knots.
        epsilon: Maximum allowed vertical scalar deviation.
        
    Returns:
        List of simplified knots.
    """
    if len(knots) < 3:
        return knots

    current_knots = list(knots)
    changed = True

    while changed:
        changed = False
        next_knots = [current_knots[0]]
        i = 0
        n = len(current_knots)

        while i < n - 2:
            k_i = current_knots[i]
            k_mid = current_knots[i+1]
            k_next = current_knots[i+2]

            # Calculate the projected scalar value at k_mid's x-position
            # on the segment connecting k_i and k_next
            x_i, s_i = k_i
            x_mid, s_mid = k_mid
            x_next, s_next = k_next

            # Avoid division by zero if x positions are identical
            if x_next == x_i:
                projected_s = s_i
            else:
                projected_s = s_i + (s_next - s_i) * (x_mid - x_i) / (x_next - x_i)

            # Calculate vertical distance
            vertical_dist = abs(s_mid - projected_s)

            if vertical_dist <= epsilon:
                # Merge: skip the middle knot
                # We do not add k_mid to next_knots
                # The next iteration will consider k_next as the new 'i' relative to the next pair
                # But since we are building next_knots, we just continue to i+2
                # However, we must ensure we don't skip k_next if it was needed for the next pair
                # The standard approach is to advance i by 2, effectively removing k_mid
                i += 2
                changed = True
            else:
                # Keep the middle knot
                next_knots.append(k_mid)
                i += 1

        # Add the last knot
        next_knots.append(current_knots[-1])
        
        current_knots = next_knots

    return current_knots
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors in calculating the projected scalar value can affect the merge decision, especially for very small $\epsilon$ [G].
*   **Collinear Points:** If three consecutive knots are perfectly collinear, the vertical distance is 0.0, so they will be merged if $\epsilon \ge 0.0$ [S1, unofficial].
*   **Boundary Conditions:** The algorithm processes consecutive non-overlapping pairs. The implementation above handles this by advancing the index. Care must be taken to ensure that skipping a knot does not prevent a valid merge in the next step that involves the skipped knot's neighbor [G]. The iterative nature ensures convergence.

## Subtopics

### PRI Data Reduction
This method is specifically applied to Piecewise Rational Interpolation (PRI) data structures to reduce the number of knots in the spline representation, which is stored in texture formats for GPU rendering [S1, unofficial].

### Comparison with Mesh Simplification
While inspired by Duchandeau's terrain mesh simplification [S1, unofficial], this method operates on 1D splines and guarantees the output knots are a subset of the input, unlike some mesh decimation techniques that may introduce new vertices.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The algorithm is iterative. In each iteration, it scans the list of knots. In the worst case, it removes half the knots per iteration, leading to $O(N \log N)$ or $O(N^2)$ depending on the data distribution, where $N$ is the number of knots [S1, unofficial].
*   **Memory:** The algorithm requires $O(N)$ memory to store the knots. The PRI implementation uses 2D textures for storage, which adds overhead for texture management [S1, unofficial].

**Measured Performance:**
*   **Blunt Fin:** Average knots per spline reduced from 49 (threshold 0.0) to 6 (threshold 0.03) [S1, unofficial].
*   **Combustor:** Average knots per spline reduced from 53 (threshold 0.0) to 10 (threshold 0.03) [S1, unofficial].
*   **Liquid Oxygen Post:** Average knots per spline reduced from 55 (threshold 0.0) to 7 (threshold 0.03) [S1, unofficial].
*   **SPX:** Average knots per spline reduced from 20 (threshold 0.0) to 4 (threshold 0.03) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple to implement; guarantees subset property; effective at removing redundant samples in linear regions; reduces memory bandwidth for texture lookups in GPU rendering [S1, unofficial].
*   **Limitations:** The resulting spline is an approximation, not an interpolation (except at the remaining knots); the error is bounded by $\epsilon$ but not minimized globally; the iterative nature can be slow for large datasets if not optimized [S1, unofficial].

## Background
The Linear Spline Simplification algorithm is based on a 1D version of the mesh simplification algorithm for terrain data by Duchandeau et al. (1997) [S1, unofficial]. It was applied in the context of GPU-based unstructured grid volume rendering using Piecewise Rational Interpolation (PRI) [S1, unofficial]. The method is used to reduce the complexity of the PRI data structure, which is represented as 2D textures (index and knots textures) for efficient GPU access [S1, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
