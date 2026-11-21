# Amanatides-Woo Grid Picking

Also called: Amanatides-Woo DDA for Grid Picking, DDA over the level map, pure Amanatides-Woo DDA

## Overview
Amanatides-Woo Grid Picking is a method to determine the grid cell and intersection face ('top' or 'side') intersected by a ray in a 2D grid-based environment, replacing expensive raycasting against detailed terrain meshes [S1, unofficial]. It builds on [Amanatides–Woo grid traversal](5003-amanatides-woo-grid-traversal.md) by applying the traversal logic to a logical height map rather than a visual mesh, ensuring that decorative geometry does not influence picking results [S1, unofficial]. Unlike brute-force raycasting against triangle meshes, this method is exact, deterministic, and significantly faster, with costs proportional only to the number of cells crossed by the ray [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `grid`: The 2D logical grid structure (dimensions and bounds).
    *   `heightAt(cell)`: A function returning the floor Y height for a given cell coordinate. It returns the drawn floor Y for normal cells, `CHASM_DEPTH` for void cells, and `min(level, cutLevel)` if a cut plane is applied [S1, unofficial].
    *   `origin`: The ray origin $(x_0, y_0, z_0)$.
    *   `dir`: The ray direction vector $(d_x, d_y, d_z)$, assumed normalized or handled via $t$ scaling.
    *   `cutLevel`: A parameter limiting the maximum height considered for hits (default $\infty$) [S1, unofficial].
*   **Outputs:**
    *   A result object containing:
        *   `cell`: The grid cell coordinates $(x, z)$ hit by the ray, or `null` if no cell is hit.
        *   `point`: The 3D intersection point.
        *   `face`: The type of face hit, either `'top'` or `'side'`. If `cell` is `null`, `face` is undefined or null.
    *   Snapping data (`corner`, `edge`, `edgeDistance`) is computed from the `point` using standard world-to-corner/edge logic [S1, unofficial].

**Data Structures:**
*   **Grid:** A 2D array or hash map of cells. Each cell has logical integer coordinates $(x, z)$ and an associated height value accessible via `heightAt`.
*   **Ray:** Defined by origin and direction.
*   **DDA State:**
    *   `tMaxX`, `tMaxZ`: The ray parameter $t$ at which the ray crosses the next grid boundary in the X and Z axes, respectively.
    *   `tDeltaX`, `tDeltaZ`: The ray parameter $t$ required to move from one grid boundary to the next in the X and Z axes.
    *   `stepX`, `stepZ`: Integers ($+1$ or $-1$) indicating the direction of traversal along the X and Z axes.

**Algorithm Steps:**

1.  **Clip the Ray:**
    Restrict the ray to the grid's bounding box.
    *   Clip $x$ and $z$ to the grid's minimum and maximum bounds.
    *   Clip $y$ to the range $[CHASM\_DEPTH, Y_{max}]$, where $Y_{max}$ is the highest top surface level in the grid [S1, unofficial].
    *   If the ray does not intersect this box, return `null` for the cell and the intersection with the $y=0$ plane for the point (if it exists) [S1, unofficial].

2.  **Initialize DDA Variables:**
    Calculate the initial step parameters for the X and Z axes [S1, unofficial]:
    $$ tDeltaX = \frac{gridSizeX}{|d_x|} $$
    $$ tDeltaZ = \frac{gridSizeZ}{|d_z|} $$
    Determine the initial crossing points `tMaxX` and `tMaxZ` based on the ray's origin and direction. If $d_x > 0$, the ray moves in the positive X direction; otherwise, it moves in the negative direction. The same logic applies to Z.
    *   If $d_x = 0$, set `tDeltaX` and `tMaxX` to $\infty$ (no X traversal).
    *   If $d_z = 0$, set `tDeltaZ` and `tMaxZ` to $\infty$ (no Z traversal).
    *   Set `stepX` and `stepZ` to $+1$ or $-1$ based on the sign of $d_x$ and $d_z$ [S1, unofficial].

3.  **Traverse Cells:**
    Iterate through the grid cells starting from the cell containing the ray origin. In each iteration:
    *   Identify the current cell $(x, z)$ and its height $h = heightAt(x, z)$ [S1, unofficial].
    *   Determine the next boundary crossing along the ray. Compare `tMaxX` and `tMaxZ` to find the smaller positive $t$. Let this be $t_{next}$ and the corresponding axis be $A$ (X or Z).
    *   Check for intersection with the current cell's top surface before reaching $t_{next}$.
        *   Calculate the ray's Y position at $t_{next}$: $y_{next} = y_0 + t_{next} \cdot d_y$.
        *   Calculate the ray's Y position at the current step's entry point (or origin).
        *   Check if the ray drops below the cell's top surface $h$ within the current column segment. Specifically, if the ray enters the cell above $h$ and exits below $h$ (or vice versa, depending on direction), a hit occurs [S1, unofficial].
        *   More precisely: If the ray's Y coordinate at the current cell's entry point is greater than $h$ and the Y coordinate at $t_{next}$ is less than $h$, the ray intersects the top face. The intersection point is calculated by solving for $t$ where $y(t) = h$.
        *   If the ray enters the cell below or at $h$, it hits the side face at the entry point of the cell [S1, unofficial].
    *   If a hit is detected:
        *   If the hit is on the 'top' face, return `{ cell: (x, z), point: intersection\_point, face: 'top' }`.
        *   If the hit is on the 'side' face, return `{ cell: (x, z), point: entry\_point, face: 'side' }`. Side hits return the raised cell [S1, unofficial].
    *   If no hit is detected, advance the DDA state:
        *   If $t_{next} == tMaxX$, update $x \leftarrow x + stepX$, and $tMaxX \leftarrow tMaxX + tDeltaX$.
        *   If $t_{next} == tMaxZ$, update $z \leftarrow z + stepZ$, and $tMaxZ \leftarrow tMaxZ + tDeltaZ$.
    *   Handle void cells: If the current cell is a void (height is `CHASM_DEPTH`), apply specific void rules. Typically, if the ray enters a hole, it returns the void cell or the owner of the wall it meets, depending on the implementation's void handling logic [S1, unofficial].
    *   If the ray exits the grid bounds without hitting a cell, return `{ cell: null, point: intersection\_with\_y=0 }` [S1, unofficial].

4.  **Compute Snapping Data:**
    Using the returned `point`, compute `corner`, `edge`, and `edgeDistance` using the same logic as the previous system (e.g., `worldToCorner`, `worldToEdge`) [S1, unofficial]. If `cell` is `null`, use the ray's intersection with the $y=0$ plane for this computation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pick_cell(grid, height_at, origin, direction, cut_level=float('inf')):
    # 1. Clip ray to grid bounding box
    # Determine grid bounds: x_min, x_max, z_min, z_max, y_min (chasm_depth), y_max
    # Clip ray to these bounds. If no intersection, return null cell and y=0 point.
    
    # 2. Initialize DDA variables
    t_delta_x = float('inf')
    t_delta_z = float('inf')
    t_max_x = float('inf')
    t_max_z = float('inf')
    step_x = 1
    step_z = 1
    
    if abs(direction.x) > 1e-6:
        t_delta_x = grid.cell_size_x / abs(direction.x)
        # Calculate initial t_max_x based on origin and direction
        # If direction.x > 0, t_max_x is distance to next positive boundary
        # If direction.x < 0, t_max_x is distance to next negative boundary
        # This involves floor/ceil operations on origin coordinates
        if direction.x > 0:
            step_x = 1
            t_max_x = (floor(origin.x / grid.cell_size_x) + 1) * grid.cell_size_x - origin.x
            t_max_x /= direction.x
        else:
            step_x = -1
            t_max_x = (ceil(origin.x / grid.cell_size_x) - 1) * grid.cell_size_x - origin.x
            t_max_x /= direction.x
    else:
        step_x = 0
        
    if abs(direction.z) > 1e-6:
        t_delta_z = grid.cell_size_z / abs(direction.z)
        if direction.z > 0:
            step_z = 1
            t_max_z = (floor(origin.z / grid.cell_size_z) + 1) * grid.cell_size_z - origin.z
            t_max_z /= direction.z
        else:
            step_z = -1
            t_max_z = (ceil(origin.z / grid.cell_size_z) - 1) * grid.cell_size_z - origin.z
            t_max_z /= direction.z
    else:
        step_z = 0
        
    # Current cell
    curr_x = floor(origin.x / grid.cell_size_x)
    curr_z = floor(origin.z / grid.cell_size_z)
    
    # 3. Traverse
    while True:
        # Check bounds
        if not grid.contains(curr_x, curr_z):
            # Ray exited grid
            # Return null cell, point at y=0 if ray intersects y=0 plane
            point_y0 = origin + direction * (-origin.y / direction.y) if direction.y != 0 else None
            return {"cell": None, "point": point_y0}
            
        # Get height of current cell
        h = height_at(curr_x, curr_z)
        # Apply cut level
        if h > cut_level:
            h = cut_level
            
        # Determine next boundary crossing
        if t_max_x < t_max_z:
            t_next = t_max_x
            axis = 'x'
        else:
            t_next = t_max_z
            axis = 'z'
            
        # Check for top hit within current column
        # Ray Y at t_next
        y_next = origin.y + t_next * direction.y
        # Ray Y at current entry (approximated by origin.y for first step, or previous t)
        # For simplicity, assume we track current t
        # If direction.y is negative, ray is going down
        if direction.y < 0:
            # Check if ray crosses h between current t and t_next
            # Current Y is origin.y + current_t * direction.y
            # We need to check if h is between current_y and y_next
            # Since direction.y < 0, y_next < current_y
            if y_next < h <= origin.y + current_t * direction.y:
                # Top hit
                t_hit = (h - origin.y) / direction.y
                point = origin + direction * t_hit
                return {"cell": (curr_x, curr_z), "point": point, "face": "top"}
        elif direction.y > 0:
            # Ray going up
            if origin.y + current_t * direction.y < h <= y_next:
                t_hit = (h - origin.y) / direction.y
                point = origin + direction * t_hit
                return {"cell": (curr_x, curr_z), "point": point, "face": "top"}
        else:
            # Horizontal ray, no top hit possible unless exactly at h
            pass
            
        # Check for side hit
        # If ray entered below h, it hits the side at entry
        # Entry Y is origin.y + current_t * direction.y
        entry_y = origin.y + current_t * direction.y
        if entry_y <= h:
            # Side hit at entry point
            return {"cell": (curr_x, curr_z), "point": origin + direction * current_t, "face": "side"}
            
        # Advance DDA
        if axis == 'x':
            curr_x += step_x
            t_max_x += t_delta_x
        else:
            curr_z += step_z
            t_max_z += t_delta_z
            
        current_t = t_next

# Helper for worldToCorner/worldToEdge would be called after picking
```

**Parameters:**
*   `cutLevel`: Default $\infty$. Limits the maximum height considered for hits [S1, unofficial].
*   `heightAt(cell)`: Returns the floor Y. For void cells, returns `CHASM_DEPTH` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Rounded Corners:** Visual mesh rounded corners differ from logical columns by at most ~0.1 cell near grid corners. The DDA picks stay logical by design, ignoring visual rounding [S1, unofficial].
*   **Layer Restriction:** The raycaster must be restricted to a specific layer (`PICK_LAYER`) to ensure decorative meshes (ground chunks, cliffs, backdrop, dice, effects) do not interfere with picking. Tokens, props, walls, and lights remain pickable [S1, unofficial].
*   **Void Cells:** Specific void rules apply. If the ray enters a hole, it returns the void cell or the owner of the wall it meets [S1, unofficial].
*   **Off-Grid:** If no cell is hit, the method returns `null` for the cell but still provides the ray's intersection with the $y=0$ plane for edge snapping [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of cells crossed by the ray [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Under 0.05 ms per pick on a 100x100 grid [S1, unofficial]. This is significantly faster than brute-force raycasting against a 205k-triangle ground mesh, which measured about 10.6 ms per pick [S1, unofficial].
*   **Strengths:** Exact, cheap, testable, and independent of visual mesh detail. No GPU cost; identical on every backend and tier [S1, unofficial].
*   **Limitations:** Logical grid only; does not account for visual rounding or decorative geometry. Requires a logical height map.
*   **When to Use:** When picking in a 3D grid-based game environment where performance is critical and visual mesh detail should not affect picking results [S1, unofficial].
*   **When Not to Use:** When precise visual intersection with complex, non-grid-aligned geometry is required.

## Background
Amanatides-Woo Grid Picking was introduced in the 'Node' project (Thirdfold/client) to replace the previous method of picking cells via raycasting against terrain boxes or a table plane at $y=0$ [S1, unofficial]. The previous method was expensive, with brute-force raycasting against a 205k-triangle ground mesh measuring about 10.6 ms per pick, compared to 0.24 ms for 1,728-instance terrain [S1, unofficial]. The new method uses a logical DDA over the level map, ensuring that decorative geometry does not decide what a click means [S1, unofficial]. It is used for cell picking in a 3D grid-based game environment, replacing `TerrainLayer` and `WorldGround`'s play-area plane [S1, unofficial].

## Sources
[S1] [M69] Pick cells with a DDA over the level map, https://github.com/Thirdfold/client/issues/246, fetched 2026-10-25 (unofficial)
