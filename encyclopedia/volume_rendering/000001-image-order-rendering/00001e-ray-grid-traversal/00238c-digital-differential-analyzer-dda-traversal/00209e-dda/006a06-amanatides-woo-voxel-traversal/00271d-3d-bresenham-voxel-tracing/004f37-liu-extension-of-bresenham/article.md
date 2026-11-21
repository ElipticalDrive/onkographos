# Liu Extension of Bresenham

Also called: Bresenham Extension, LZY04, Liu Extension, Liu et al. Algorithm

## Overview
The Liu Extension of Bresenham is a voxel traversal algorithm that extends the classic 2D Bresenham line algorithm to 3D grid traversal by detecting and visiting diagonal neighbors when the ray intersects a cell corner [S1][S2, unofficial]. Unlike standard Bresenham which visits only one neighbor per step, this method checks if the error exceeds a specific threshold to decide whether to visit both $(x+1, y)$ and $(x+1, y+1)$ in 2D, or the corresponding diagonal paths in 3D [S1][S2, unofficial]. It is distinguished from its siblings by its reliance on a geometric intersection check against a derived threshold rather than simple integer error accumulation, offering performance advantages even on modern hardware where floating-point operations are efficient [S1][S2, unofficial].

## How it works
**Inputs:** Ray origin $(x_0, y_0, z_0)$, ray direction $(dx, dy, dz)$, and the starting voxel $(x, y, z)$. The direction components $dx, dy, dz$ are assumed to be positive (symmetric handling for negative directions is required but not detailed in the primary source text) [S1][S2, unofficial].
**Outputs:** A sequence of voxel coordinates visited by the ray.

**Data Structures:**
*   Current voxel coordinates: $(x, y, z)$ (integers).
*   Error accumulator: $e$ (floating-point or fixed-point).
*   Slope: $k = dx/dy$ (for 2D) or analogous slopes for 3D axes [S1][S2, unofficial].

**Steps:**
1.  **Initialization:** Determine the driving axis. In the standard 2D case described, the x-axis is the driving axis. Initialize the error $e$ based on the initial position [S1][S2, unofficial].
2.  **Intersection and Threshold Comparison:** At each step, determine the intersection point of the ray with the two walls of the current cell (specifically the walls perpendicular to the non-driving axes) [S1]. Compute the error $e$, defined as the difference between the chosen integer cell boundary and the real coordinate of the line intersection [S1][S2, unofficial].
3.  **Threshold Check:** Compare the error $e$ against a threshold. The source text provides conflicting formulas for the threshold:
    *   Source [S1] states: $\text{threshold} = \frac{1}{2} - \frac{dy}{2dx}$.
    *   Source [S2, unofficial] states: $\text{threshold} = \frac{1}{2} - \frac{1}{2dx \times dy}$ (garbled text "21 2dx dy" interpreted as $\frac{1}{2} - \frac{1}{2dx dy}$ or similar, but later text says $2^{1/2} dx dy$ which is likely a transcription error for the fraction).
    *   *Implementation Note:* The standard Bresenham threshold is $1/2$. The Liu extension modifies this to account for the diagonal intersection. Given the garbled nature of [S2, unofficial], the formula from [S1] ($\frac{1}{2} - \frac{dy}{2dx}$) is structurally more consistent with a slope-based adjustment, though exact verification requires the original [LZY04] paper. If $e > \text{threshold}$, the ray is considered to pass through a corner/edge, requiring the visit of diagonal neighbors.
4.  **Neighbor Selection:**
    *   **2D Case:** If $e > \text{threshold}$, visit both $(x+1, y)$ and $(x+1, y+1)$ [S1][S2, unofficial]. Otherwise, visit only $(x+1, y)$ (assuming y is the secondary axis) [S1][S2, unofficial].
    *   **3D Case:** Identify two possible paths from cell $(x, y, z)$ to $(x+1, y+1, z+1)$ [S1][S2, unofficial]. The specific logic for selecting between these paths or visiting both is not fully detailed in the provided text, but it implies a similar diagonal check in 3D space.
5.  **Integer Adjustment:** Thereafter, follow the standard Bresenham approach, adjusting the algorithm to integer arithmetic by updating the error $e$ using integer additions/subtractions based on the slope $k = dx/dy$ [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def liu_bresenham_2d(x0, y0, x1, y1):
    """
    Liu Extension of Bresenham for 2D grid traversal.
    Assumes dx > 0, dy > 0.
    """
    dx = x1 - x0
    dy = y1 - y0
    
    # Slope calculation
    k = dx / dy
    
    # Threshold calculation based on [S1]
    # Note: [S2, unofficial] provides conflicting/garbled formulas.
    # Using [S1] formula: threshold = 1/2 - dy/(2*dx)
    threshold = 0.5 - (dy / (2.0 * dx))
    
    x, y = x0, y0
    yield (x, y)
    
    # Error initialization [G]
    # Standard Bresenham initializes error based on midpoint.
    # Liu extension checks intersection with walls.
    # We assume e starts at 0 or a value representing initial offset.
    e = 0.0 
    
    for i in range(dx):
        # Determine intersection error e
        # In standard Bresenham, e accumulates k.
        # Liu checks if the line crosses a corner.
        # The text says "determines the intersection point... and compare error e".
        # We update e by adding the slope component for the step.
        e += k 
        
        # Check threshold
        if e > threshold:
            # Visit diagonal neighbors: (x+1, y) and (x+1, y+1)
            yield (x + 1, y)
            yield (x + 1, y + 1)
            # Adjust error for integer arithmetic [G]
            # Standard Bresenham subtracts 1 from error if it exceeds 1.
            # Here, we might need to adjust e to reflect the diagonal step.
            e -= 1.0 
        else:
            # Visit only (x+1, y)
            yield (x + 1, y)
        
        x += 1
        
    # Note: The 3D extension is described as "quite similar" but specific
    # integer adjustment steps for 3D are not provided in the text.
```

**Edge Cases and Pitfalls:**
*   **Threshold Formula Disagreement:** Sources [S1] and [S2, unofficial] provide different or garbled formulas for the threshold. Implementers must verify the correct threshold against the original [LZY04] paper.
*   **Negative Directions:** The algorithm assumes positive direction components. Handling negative $dx, dy, dz$ requires mirroring the coordinate system or adjusting the neighbor selection logic.
*   **3D Complexity:** The 3D extension is described vaguely ("identify two possible paths"). Implementers must derive the specific 3D diagonal traversal logic, likely involving checks against multiple thresholds for the three axes.

## Operating characteristics
*   **Performance:** The Liu algorithm is reported to be faster than Zalik et al.'s code-based approach and even faster than standard Bresenham on modern processors, where floating-point arithmetic is nearly as efficient as integer arithmetic [S1][S2, unofficial].
*   **Complexity:** Time complexity is linear with respect to the number of voxels traversed. Memory complexity is constant, storing only current coordinates and error terms.
*   **Strengths:** Efficient diagonal traversal without complex branching; leverages floating-point precision for threshold checks.
*   **Limitations:** Ambiguity in the 3D extension details; potential precision issues if the threshold formula is misinterpreted.

## Background
The Liu Extension of Bresenham was proposed by Liu et al. in [LZY04]. It is an efficient extension of Bresenham's algorithm designed for ray tracing and grid traversal. It addresses the limitation of standard Bresenham by explicitly handling diagonal cell visits when the ray intersects a cell corner, improving traversal accuracy and efficiency in certain scenarios [S1][S2, unofficial].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Grids in Ray Tracing: A Comprehensive Survey, https://www.scribd.com/document/349616186/GG-400, fetched 2026-10-25 (unofficial)
