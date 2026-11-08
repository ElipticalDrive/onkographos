# Ray-Cube Intersection

Also called: box initialization

## Overview
Ray-Cube Intersection computes the precise 3D entry ($t_0$) and exit ($t_1$) points of a ray relative to an axis-aligned cube defined by lower and upper bounds, along with the specific geometric face indices crossed [S1, unofficial]. Unlike standard slab methods that return a boolean or interval, this method explicitly resolves the exact boundary coordinates and face normals required to initialize brick-level traversal [S1, unofficial].

## How it works
The method performs two independent boundary hit calculations: one for the exit point ($t_1$) against the upper bounds, and one for the entry point ($t_0$) against the lower bounds. Both compute an intersection parameter vector $Q_v$, select the extremal axis component, and clamp the resulting 3D position to the exact cube boundary [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `raypos`: The origin of the ray, a 3D vector $(x, y, z)$.
- `raydir`: The direction of the ray, a 3D vector $(dx, dy, dz)$.
- `dir_lower`: The lower corner of the cube, a 3D vector $(min_x, min_y, min_z)$.
- `dir_upper`: The upper corner of the cube, a 3D vector $(max_x, max_y, max_z)$.

**Outputs:**
- `HitEntry`: A structure containing the 3D position $t_0$ and the entry face index `inface`.
- `HitExit`: A structure containing the 3D position $t_1$ and the exit face index `outface`.

### Data Structures
The hit structure is defined as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Hit:
    position: Vector3  # 3D float vector (x, y, z)
    face: int          # Integer index identifying the face
```

### Formulas and Steps

#### 1. Exit Point Calculation ($t_1$)
This step finds where the ray leaves the cube.

1.  **Compute Intersection Parameters ($Q_v$):**
    For each axis $i \in \{0, 1, 2\}$ (corresponding to $x, y, z$), calculate:
    $$ Q_v[i] = \frac{dir\_upper[i] - raypos[i]}{raydir[i]} $$
    [S1, unofficial]

2.  **Identify Minimum Axis:**
    Find the index $i$ of the minimum component in $Q_v$:
    $$ i = \text{mincomponentf3index}(Q_v) $$
    [S1, unofficial]

3.  **Calculate Intersection Distance ($Q$):**
    $$ Q = Q_v[i] $$
    [S1, unofficial]

4.  **Calculate 3D Position ($t_1$):**
    $$ t_1 = raypos + raydir \times Q $$
    [S1, unofficial]

5.  **Determine Exit Face (`outface`):**
    Use a lookup table `outfaces[i][direction]`, where `direction` is determined by the sign of the ray direction component:
    $$ \text{direction} = \begin{cases} 0 & \text{if } raydir[i] < 0 \\ 1 & \text{otherwise} \end{cases} $$
    $$ outface = outfaces[i][\text{direction}] $$
    [S1, unofficial]

6.  **Clamp Position:**
    To ensure numerical precision and exact boundary alignment, set the $i$-th component of $t_1$ to the corresponding bound:
    $$ t_1[i] = dir\_upper[i] $$
    [S1, unofficial]

7.  **Return:**
    Return `Hit` with `position = t1` and `face = outface`.

#### 2. Entry Point Calculation ($t_0$)
This step finds where the ray enters the cube.

1.  **Compute Intersection Parameters ($Q_v$):**
    For each axis $i \in \{0, 1, 2\}$, calculate:
    $$ Q_v[i] = \frac{dir\_lower[i] - raypos[i]}{raydir[i]} $$
    [S1, unofficial]

2.  **Handle Invalid Values:**
    Coalesce any `NaN` values in $Q_v$ to negative infinity ($-\infty$) to handle division by zero or invalid inputs.
    [S1, unofficial]

3.  **Identify Maximum Axis:**
    Find the index $i$ of the maximum component in $Q_v$:
    $$ i = \text{maxcomponentf3index}(Q_v) $$
    [S1, unofficial]

4.  **Calculate Intersection Distance ($Q$):**
    $$ Q = Q_v[i] $$
    [S1, unofficial]

5.  **Calculate 3D Position ($t_0$):**
    $$ t_0 = raypos + raydir \times Q $$
    [S1, unofficial]

6.  **Determine Entry Face (`inface`):**
    Use a lookup table `infaces[i][direction]`, where `direction` is determined by the sign of the ray direction component:
    $$ \text{direction} = \begin{cases} 0 & \text{if } raydir[i] < 0 \\ 1 & \text{otherwise} \end{cases} $$
    $$ inface = infaces[i][\text{direction}] $$
    [S1, unofficial]

7.  **Clamp Position:**
    To ensure numerical precision and exact boundary alignment, set the $i$-th component of $t_0$ to the corresponding bound:
    $$ t_0[i] = dir\_lower[i] $$
    [S1, unofficial]

8.  **Return:**
    Return `Hit` with `position = t0` and `face = inface`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Tuple, Optional

Vector3 = Tuple[float, float, float]
Hit = Tuple[Vector3, int]  # (position, face_index)

def safe_div(a: float, b: float) -> float:
    """Division handling zero denominator per [S1, unofficial]."""
    if b == 0.0:
        return -math.inf if a < 0 else math.inf
    return a / b

def min_component_index(v: Vector3) -> int:
    min_val, min_idx = v[0], 0
    if v[1] < min_val: min_val, min_idx = v[1], 1
    if v[2] < min_val: min_val, min_idx = v[2], 2
    return min_idx

def max_component_index(v: Vector3) -> int:
    max_val, max_idx = v[0], 0
    if v[1] > max_val: max_val, max_idx = v[1], 1
    if v[2] > max_val: max_val, max_idx = v[2], 2
    return max_idx

def ray_cube_exit(raypos: Vector3, raydir: Vector3, dir_upper: Vector3) -> Optional[Hit]:
    """Calculates the exit point (t1) of a ray into a cube [S1, unofficial]."""
    # 1. Compute Qv
    Qv = [safe_div(dir_upper[0] - raypos[0], raydir[0]),
          safe_div(dir_upper[1] - raypos[1], raydir[1]),
          safe_div(dir_upper[2] - raypos[2], raydir[2])]
    
    # 2. Check for negative components (ray outside or pointing away) [S1, unofficial]
    if any(q < 0 for q in Qv):
        return None
        
    # 3. Identify minimum axis
    i = min_component_index(Qv)
    Q = Qv[i]
    
    # 4. Calculate t1
    t1 = (raypos[0] + raydir[0] * Q,
          raypos[1] + raydir[1] * Q,
          raypos[2] + raydir[2] * Q)
    
    # 5. Determine outface
    direction = 0 if raydir[i] < 0 else 1
    # Note: The sources do not give the exact face index mapping for outfaces[i][direction] [G]
    outface = 0  # Placeholder index; replace with implementation-specific lookup table
    
    # 6. Clamp t1
    t1_clamped = list(t1)
    t1_clamped[i] = dir_upper[i]
    t1_final = tuple(t1_clamped)
    
    return (t1_final, outface)

def ray_cube_entry(raypos: Vector3, raydir: Vector3, dir_lower: Vector3) -> Optional[Hit]:
    """Calculates the entry point (t0) of a ray into a cube [S1, unofficial]."""
    # 1. Compute Qv
    Qv = [safe_div(dir_lower[0] - raypos[0], raydir[0]),
          safe_div(dir_lower[1] - raypos[1], raydir[1]),
          safe_div(dir_lower[2] - raypos[2], raydir[2])]
    
    # 2. Coalesce NaN to -inf [S1, unofficial]
    Qv = [-math.inf if math.isnan(v) else v for v in Qv]
    
    # 3. Check for negative components [S1, unofficial]
    if any(q < 0 for q in Qv):
        return None
        
    # 4. Identify maximum axis
    i = max_component_index(Qv)
    Q = Qv[i]
    
    # 5. Calculate t0
    t0 = (raypos[0] + raydir[0] * Q,
          raypos[1] + raydir[1] * Q,
          raypos[2] + raydir[2] * Q)
    
    # 6. Determine inface
    direction = 0 if raydir[i] < 0 else 1
    # Note: The sources do not give the exact face index mapping for infaces[i][direction] [G]
    inface = 0  # Placeholder index; replace with implementation-specific lookup table
    
    # 7. Clamp t0
    t0_clamped = list(t0)
    t0_clamped[i] = dir_lower[i]
    t0_final = tuple(t0_clamped)
    
    return (t0_final, inface)
```

### Parameters
- `dir_lower`, `dir_upper`: The bounding box of the cube [S2, unofficial].
- `raypos`, `raydir`: The ray definition [S1, unofficial].

### Edge Cases and Pitfalls
- **Negative $Q_v$ Components:** If any component of $Q_v$ is negative, the ray originates outside the cube or points away from the relevant face, yielding no valid entry/exit within the box bounds defined by the logic. The function returns a null hit [S1, unofficial].
- **All Negative $Q_v$:** If all components of $Q_v$ are negative, the ray does not intersect the cube in the forward direction. Return a null hit [S1, unofficial].
- **Division by Zero:** If `raydir[i]` is 0, the division results in infinity or NaN. The source [S1, unofficial] specifies coalescing NaNs to negative infinity for the entry calculation to handle this gracefully.
- **Precision:** Clamping the intersection point to the exact boundary (`dir_lower` or `dir_upper`) is critical to avoid self-intersection artifacts in subsequent traversal steps [S1, unofficial].

## Subtopics

### [Ray-Cube Intersection (Brick/Slab)](3620-ray-cube-intersection-brick-slab.md)
This method performs the primary hit resolution for a ray entering a 4×4×4 voxel brick, finding the entry point ($t_0$) of the first solid voxel the ray crosses within a specified time interval $[t_{lo}, t_{hi}]$. It builds on [Ray-Cube Intersection](2466-ray-cube-intersection.md) by extending the boundary check into a full traversal of the brick's interior voxels using the Amanatides-Woo Digital Differential Analyzer (DDA). Unlike the parent method which only identifies the cube entry face, this method returns the time of the entry face of the first solid voxel found, or -1 if the ray misses all solid voxels in the brick.

## Operating characteristics
- **Time Complexity:** $O(1)$ for both entry and exit calculations, involving a constant number of arithmetic operations and comparisons [G].
- **Memory Complexity:** $O(1)$, requiring only a few temporary vectors and scalars [G].
- **Strengths:** Provides exact boundary intersection points and face indices, which are essential for robust voxel traversal [G]. The clamping step ensures numerical stability [G].
- **Limitations:** Does not perform traversal within the volume; it only determines the entry/exit boundaries. Requires careful handling of division by zero and NaN values [G].
- **When to Use:** When initializing ray traversal for voxel-based volume rendering, specifically when the entry/exit points and face normals are needed to set up the DDA state [G].
- **When Not to Use:** When only a boolean intersection test is required (use [Ray-box intersection](776-ray-box-intersection.md) or a simpler AABB test). When traversing the interior of the volume (use [Ray-Cube Intersection (Brick/Slab)](3620-ray-cube-intersection-brick-slab.md)) [G].

## Background
The method originates from implementations in voxel rendering libraries, specifically the `landscapes` repository [S1, unofficial] and the `r3d` repository [S2, unofficial]. The `landscapes` source provides the detailed algorithm for calculating $t_0$ and $t_1$ with face identification, while `r3d` provides the context of initializing a polyhedron structure as an axis-aligned cube [S2, unofficial]. The approach adapts standard ray-primitive intersection logic for explicit face and coordinate output in voxel traversal pipelines [G].

## Sources
[S1] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
[S2] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
