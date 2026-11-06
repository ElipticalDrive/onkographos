# Sphere Assisted Raymarching

Also called: Shape Assisted Raymarching, Shape Marching, Sphere Assisted

## Overview
Sphere Assisted Raymarching is a technique for accelerating ray traversal in voxel grids by storing the radius of the largest empty sphere centered at each voxel in a separate shape texture [S1, S2, unofficial]. This decouples shape information from color information [S1, unofficial]. The technique allows rays to take large steps through empty space, significantly reducing the number of voxel lookups compared to plane marching [S1, S2, unofficial]. It sets itself apart from its parent method by using the exact Euclidean distance (sphere radius) rather than an L1 approximation to determine step sizes, providing tighter bounds on empty space [G].

## How it works
Sphere Assisted Raymarching stores the Euclidean distance to the nearest solid voxel for every empty voxel [S1, S2, unofficial].

**Inputs and Outputs**
*   **Input:** Ray origin $p_0$, ray direction $d$ (normalized), maximum check distance $maxDistToCheck$, voxel grid size $voxelGridSize$ [G].
*   **Output:** Color of the first solid voxel hit, or transparent if no voxel is hit within $maxDistToCheck$ [G].

**Data Structures**
Two 3D textures are required [S1, unofficial]:
1.  **`voxels`**: A 3D texture storing normalized float color values, where the alpha channel indicates solidity [S1, unofficial].
2.  **`shape`**: A separate 3D texture storing non-negative integers, where the red channel stores the radius of the largest empty sphere centered at that voxel [S1, S2, unofficial].
    *   **Solid Voxel Marker**: The value `255` represents a solid voxel [S1, S2, unofficial].
    *   **Empty Voxel Value**: The value represents the distance to the nearest solid voxel [S2, unofficial].

**Algorithm Steps**
1.  Initialize ray parameter $t = 0$ [S1, unofficial].
2.  Loop while $t \le maxDistToCheck$:
    a.  Calculate current position $p = p_0 + d \cdot t$ [S1, unofficial].
    b.  Convert to grid coordinates $p_{grid} = p / voxelGridSize$ [G].
    c.  Sample the shape texture at $p_{grid}$ to get `shapeValue` [S1, S2, unofficial].
    d.  If `shapeValue` == 255, the ray has hit a solid voxel. Return the color from the `voxels` texture at $p_{grid}$ [S1, S2, unofficial].
    e.  Calculate the distance to the next axis-aligned plane boundaries along $d$ using $\delta_{plane} = \min_i \frac{\mathbb{I}(d_i > 0) - \text{fract}(p_{grid,i})}{d_i}$, where $\mathbb{I}$ is the indicator function and $\text{fract}(x) = x - \lfloor x \rfloor$ [S1, unofficial].
    f.  Update $t \leftarrow t + \max(\delta_{plane} + \text{shapeValue}, \epsilon)$ [S1, unofficial].
3.  If the loop terminates without hitting a solid voxel, return transparent color [S1, unofficial].

**Pseudocode**
The following pseudocode implements the method as described in [S1, unofficial]. It uses standard Python syntax and explicit vector operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_assisted_raymarch(p0, d, maxDistToCheck, voxelGridSize, shape_texture, voxels_texture):
    t = 0.0
    epsilon = 1e-6  # Small constant to guarantee progress [S1, unofficial]
    
    while t <= maxDistToCheck:
        # Calculate current world position
        p = [p0[i] + d[i] * t for i in range(3)]
        
        # Convert to grid coordinates for sampling and plane calculation
        p_grid = [p[i] / voxelGridSize for i in range(3)]
        
        # Sample shape data
        shape_value = shape_texture.sample_level(p_grid, 0).r
        
        # Check for solid voxel
        if shape_value == 255:
            return voxels_texture.sample_level(p_grid, 0)
        
        # Calculate deltas to next plane boundaries
        # step(0, d) returns 1.0 where d > 0, else 0.0
        # fract(p_grid) returns the fractional part of grid coordinates
        step_mask = [1.0 if d[i] > 0.0 else 0.0 for i in range(3)]
        fract_p = [p_grid[i] - math.floor(p_grid[i]) for i in range(3)]
        deltas = [(step_mask[i] - fract_p[i]) / d[i] for i in range(3)]
        
        # Find the minimum delta to the nearest plane boundary
        min_delta = min(deltas)
        
        # Step forward by the maximum of the plane distance and the sphere radius
        t += max(min_delta + shape_value, epsilon)
        
    return [0.0, 0.0, 0.0, 0.0]
```

**Parameters**
*   `255`: Constant representing a solid voxel in the shape texture [S1, S2, unofficial].
*   `epsilon`: Small constant added to the step size to guarantee progress and avoid infinite loops due to floating-point precision issues [S1, unofficial].
*   `maxDistToCheck`: Maximum distance the ray will travel [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Upfront Cost**: Generating the shape data (distance field) has a significant upfront computational cost [S2, unofficial].
*   **Memory Usage**: Requires two textures (color and shape), increasing memory usage compared to single-texture methods [S1, unofficial].
*   **Precision**: The shape data uses integer formats to store distances, which may limit precision for very large voxel grids or fine details [S1, unofficial].

## Subtopics

### [Cube Assisted Raymarching](4940-cube-assisted-raymarching.md)
A variant that stores the size of the largest possible cube around each empty cell instead of a sphere. It is faster to compute (order of magnitude faster) but slightly suboptimal for step size compared to sphere-assisted. It uses a summed area table (SAT) to compute the area of arbitrary boxes in $O(1)$ time, allowing binary search for the maximum cube size in $O(\log n)$ per voxel, reducing upfront complexity from $O(n^2)$ to $O(n \log n)$.

## Operating characteristics
*   **Time Complexity**: The raymarching step itself is efficient, reducing voxel lookups in empty space [S1, unofficial]. However, the upfront cost to generate the shape data is $O(n^2)$ for sphere-assisted, where $n$ is the number of voxels [S2, unofficial].
*   **Memory Complexity**: Requires two 3D textures (color and shape), doubling memory usage compared to methods using a single texture [S1, unofficial].
*   **Performance**: Reduces the number of voxel lookups, which can improve performance in sparse scenes [S1, unofficial]. However, it is not always a performance win due to the overhead of texture sampling and the upfront generation cost [S1, S2, unofficial].
*   **Strengths**: Large steps through empty space; reduces ray traversal steps [S1, S2, unofficial].
*   **Limitations**: High upfront computational cost for shape generation; higher memory usage; integer precision limits [S1, S2, unofficial].
*   **When to Use**: When the scene is sparse and the cost of ray traversal is high relative to the upfront shape generation cost [S1, unofficial].
*   **When Not to Use**: When memory is constrained, or when the scene is dense and the upfront cost outweighs the traversal savings [S1, S2, unofficial].

## Background
Sphere Assisted Raymarching was introduced by Caleb Leak in his blog posts on raymarching voxel rendering [S1, S2, unofficial]. It was developed as an optimization over plane marching to handle large empty spaces more efficiently [S1, S2, unofficial].

## Sources
[S1] Raymarching Voxel Rendering | Caleb Leak, https://calebleak.com/posts/raymarching-voxel-rendering/, fetched 2026-10-24 (unofficial)
[S2] Raymarching Voxel Rendering, https://medium.com/@calebleak/raymarching-voxel-rendering-58018201d9d6, fetched 2026-10-25 (unofficial)
