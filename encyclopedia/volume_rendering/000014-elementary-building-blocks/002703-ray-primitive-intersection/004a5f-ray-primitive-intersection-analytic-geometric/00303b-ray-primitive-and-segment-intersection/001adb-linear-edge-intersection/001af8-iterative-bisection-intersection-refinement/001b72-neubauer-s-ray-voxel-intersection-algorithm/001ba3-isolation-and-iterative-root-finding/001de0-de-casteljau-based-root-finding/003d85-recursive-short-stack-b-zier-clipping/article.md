# Recursive Short-Stack Bézier Clipping

Also called: Modified Bézier Clipping, Optimized Bézier Clipping

## Overview
Recursive Short-Stack Bézier Clipping is a GPU-optimized algorithm for finding the intersection of a ray with a Bézier curve (or surface parameterized as a curve in the clipping dimension). It solves the problem of efficiently locating roots of a polynomial function defined by Bézier control points while minimizing register usage and avoiding expensive convex hull intersection tests. It distinguishes itself from standard iterative Bézier Clipping by using a fixed-size stack to limit recursion depth and by aggressively splitting the control polygon when sign changes exceed one, thereby reducing the number of expensive geometric tests required [S1].

## How it works
**Inputs:**
- A set of Bézier control points $P_0, P_1, \dots, P_n$ defining the curve segment.
- A convergence threshold $\epsilon$ (typically $10^{-4}$) [S1].
- A maximum stack size $S_{max}$ (typically 3) [S1].

**Outputs:**
- The parameter $t \in [0, 1]$ where the curve intersects the axis (zero-crossing), or an indication that no intersection exists within the segment.

**Data Structures:**
- **Stack:** A fixed-size array of control point sets (or indices to them) with capacity $S_{max}$. The stack stores sub-segments of the control polygon that require further processing.
- **Control Points:** Represented as values along the axis of interest (e.g., the signed distance from the ray to the surface, or the polynomial value).

**Algorithm Steps:**
1. **Initialization:** Push the initial set of control points onto the stack.
2. **Loop:** While the stack is not empty:
    a. Pop a set of control points $C = \{P_0, \dots, P_n\}$ from the stack.
    b. **Sign Change Count:** Count the number of sign changes in the sequence of control point values along the axis. Let this count be $k$.
    c. **Case $k = 0$:** The convex hull does not intersect the axis. Discard this segment; no intersection exists.
    d. **Case $k > 1$:** The control polygon is likely to contain multiple roots or is too wide. Split the control polygon into two halves using de Casteljau's algorithm (or equivalent subdivision). Push both resulting sub-segments onto the stack. Do not perform the expensive intersection test.
    e. **Case $k = 1$:** The convex hull intersects the axis exactly once. Perform the expensive intersection test between the convex hull and the axis to find the root $t$.
        i. If the root $t$ is found and the segment width is greater than $\epsilon$, split the segment at $t$ and push the two new sub-segments onto the stack for refinement.
        ii. If the segment width is $\le \epsilon$, return $t$ as the intersection point.

**Formulas:**
- **Sign Change Count:** For a sequence $v_0, v_1, \dots, v_n$, the number of sign changes is the count of indices $i$ such that $v_i \cdot v_{i+1} < 0$ (ignoring zeros or handling them by skipping).
- **Subdivision (de Casteljau):** To split a Bézier curve at $t=0.5$, the new control points $Q_i$ are computed as:
  $$ Q_i = \sum_{j=0}^{i} \binom{i}{j} (0.5)^j (0.5)^{i-j} P_j $$
  Or recursively via de Casteljau's algorithm.

**Parameters:**
- Convergence threshold: $10^{-4}$ [S1].
- Stack size: 3 [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision errors may cause sign changes to be detected incorrectly near zero. Robust handling of zero values is required [G].
- **Degenerate Input:** If all control points are zero, the curve lies on the axis. The algorithm should handle this by returning the midpoint or a valid range [G].
- **Stack Overflow:** The fixed stack size limits recursion depth. If the stack fills up, the algorithm may need to discard segments or increase the stack size, but the method assumes a stack size of 3 is sufficient for typical cases [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_short_stack_bezier_clipping(control_points, epsilon=1e-4, stack_size=3):
    """
    Finds the root of a Bézier curve defined by control_points.
    control_points: list of scalar values representing control points along the axis.
    epsilon: convergence threshold.
    stack_size: maximum size of the stack.
    Returns: t (float) or None if no intersection found.
    """
    stack = [control_points]
    
    while stack:
        if len(stack) == 0:
            break
            
        current_points = stack.pop()
        n = len(current_points) - 1
        
        # Count sign changes
        sign_changes = 0
        for i in range(n):
            if current_points[i] * current_points[i+1] < 0:
                sign_changes += 1
        
        if sign_changes == 0:
            # No intersection
            continue
        elif sign_changes > 1:
            # Split the polygon
            mid = n // 2
            # Subdivide into two halves
            left_points = de_casteljau_subdivide(current_points, 0.5)[0]
            right_points = de_casteljau_subdivide(current_points, 0.5)[1]
            
            # Push only if stack has space (or handle overflow)
            if len(stack) < stack_size:
                stack.append(left_points)
                stack.append(right_points)
            else:
                # Stack full, push one and discard the other? 
                # The source implies stack size 3 is sufficient, so we assume it fits.
                # If not, we might need to prioritize or drop. 
                # For simplicity, we assume it fits as per source claim.
                pass 
        elif sign_changes == 1:
            # Perform expensive intersection test
            t = convex_hull_axis_intersection(current_points)
            if t is not None:
                # Check convergence
                if is_converged(current_points, t, epsilon):
                    return t
                else:
                    # Split at t and push sub-segments
                    left_points, right_points = de_casteljau_subdivide(current_points, t)
                    if len(stack) < stack_size:
                        stack.append(left_points)
                        stack.append(right_points)
                    else:
                        pass # Handle stack overflow if necessary
        
    return None

def de_casteljau_subdivide(points, t):
    """
    Subdivides a Bézier curve at parameter t.
    Returns (left_control_points, right_control_points).
    """
    n = len(points) - 1
    left = [points[0]]
    right = [points[n]]
    
    # Compute intermediate points
    current_level = points[:]
    for i in range(1, n + 1):
        next_level = []
        for j in range(n - i + 1):
            p = (1 - t) * current_level[j] + t * current_level[j + 1]
            next_level.append(p)
            if i == n:
                # The last point is the intersection point
                pass
            if j == 0:
                left.append(p)
            if j == n - i:
                right.append(p)
        current_level = next_level
        
    return left, right

def convex_hull_axis_intersection(points):
    """
    Finds the intersection of the convex hull of the control points with the axis.
    This is the "expensive" test.
    Returns t in [0, 1] or None.
    """
    # For a Bézier curve, the convex hull is the polygon formed by the control points.
    # The intersection with the axis can be found by checking the edges of the convex hull.
    # However, for Bézier clipping, it often involves solving for the root within the hull.
    # A simple approximation is to use the linear interpolation between the points with opposite signs.
    # More robustly, it might involve solving the polynomial equation.
    # Given the source context, this is likely a geometric test.
    # We assume a simple linear interpolation for the sake of implementation completeness.
    for i in range(len(points) - 1):
        if points[i] * points[i+1] < 0:
            # Linear interpolation
            t = points[i] / (points[i] - points[i+1])
            return t
    return None

def is_converged(points, t, epsilon):
    """
    Checks if the segment width is less than epsilon.
    """
    # The width of the segment at t can be approximated by the difference in y-values
    # or the distance between the control points.
    # A common metric is the difference between the max and min control point values.
    width = max(points) - min(points)
    return width < epsilon
```

## Subtopics

### de Casteljau-based root finding
The underlying method for subdividing Bézier curves used in this algorithm. It provides the mechanism for splitting the control polygon without index operations. [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md)

## Operating characteristics
- **Time Complexity:** The algorithm reduces the number of iterations significantly compared to standard Bézier Clipping. Pabst et al.'s method used 16.4 iterations on average (max 29), while this method uses 2.3 iterations on average (max 8) [S1].
- **Memory Complexity:** Uses a fixed-size stack of size 3, minimizing register usage on the GPU [S1].
- **Performance:** Achieves approximately double the overall speedup of standard Bézier Clipping [S1].
- **Strengths:** Efficient on GPU due to fixed stack size and reduced intersection tests. Robust against degenerate cases where the convex hull barely intersects the axis.
- **Limitations:** The fixed stack size may require careful handling if the recursion depth exceeds the limit, though the source claims a size of 3 is sufficient.

## Background
The method was introduced by Bernadon et al. in their work on GPU-based fast ray casting for a large number of metaballs [S1]. It builds on the iterative Bézier Clipping method by Pabst et al. [PSS∗06] and the recursive algorithm by Nishita et al. [NSK90]. The key innovation is the modification of the splitting strategy to occur early when sign changes exceed one, addressing a flaw pointed out by Campagna et al. [CSS97] where the convex hull barely intersects the axis. The method was tested on an NVIDIA GeForce 8800 Ultra graphics card using Wyvill et al.'s degree-six polynomial for metaballs [WMW86].

## Sources
[S1] GPU‐based Fast Ray Casting for a Large Number of Metaballs, https://doi.org/10.1111/j.1467-8659.2008.01132.x, fetched 2026-11-01
