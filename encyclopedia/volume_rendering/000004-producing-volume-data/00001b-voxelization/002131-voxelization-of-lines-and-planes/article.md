# Voxelization of Lines and Planes
Also called: Slab-based Voxelization, N-separating Voxelization, 6-separating voxelization, 26-separating voxelization, 6-Separating Plane Voxelization, 26-Separating Plane Voxelization

## Overview
This method generates a discrete, topologically correct representation of 2D lines and 3D planes by defining a thickness parameter based on the surface normal and grid resolution. It guarantees **N-separability** (no N-holes) and **N-minimality** (no redundant voxels) for $N \in \{4, 6, 8, 26\}$ [S2]. Unlike general mesh voxelization which may produce artifacts at edges and vertices, this method provides a rigorous geometric criterion for inclusion, ensuring that connected ray paths (6-connected or 26-connected) cannot penetrate the discretized surface [S1, unofficial][S2]. It builds on [Voxelization](12-voxelization.md) by focusing specifically on the topological integrity of planar primitives.

## How it works
**Inputs:**
*   Primitive definition:
    *   2D Line: $A_p x + B_p y + D_p = 0$
    *   3D Plane: $A_p X + B_p Y + C_p Z + D_p = 0$
*   Grid parameters: Voxel width $L$, grid resolution.
*   Separability type $N$: 4, 6, 8, or 26.

**Outputs:**
*   A set of voxels (or pixels) $\tilde{S}$ marked as "black" (occupied) or "white" (empty).

**Data Structures:**
*   A 3D boolean grid or sparse structure indexed by integer coordinates $(x, y, z)$.
*   For 2D lines, a 2D grid indexed by $(x, y)$.

**Parameters:**
*   $L$: Voxel width (typically 1.0 in normalized grid space, or physical size).
*   $t_N$: The thickness threshold for separability $N$.
*   $\alpha$: Angle between the voxel/pixel diagonal and the surface normal.
*   $\beta$: Angle between the voxel face normal and the surface normal.

**Steps:**

1.  **Normalize the Plane/Line Equation:**
    Ensure the normal vector $(A_p, B_p, C_p)$ is normalized such that its magnitude is 1.
    $$ \sqrt{A_p^2 + B_p^2 + C_p^2} = 1 $$
    [S2]

2.  **Calculate Thickness Threshold ($t_N$):**
    Compute the slab thickness based on the desired separability and the orientation of the surface relative to the grid.
    *   For **26-separability** (3D):
        $$ t_{26} = \frac{L}{2 \sqrt{3} \cos \alpha} $$
        where $\alpha$ is the angle between the voxel diagonal $K$ and the plane normal $N$. [S2]
    *   For **6-separability** (3D):
        $$ t_{6} = \frac{L}{2 \cos \beta} $$
        where $\beta$ is the angle between the plane normal and the normal to the voxel face. [S2]
    *   For **8-separability** (2D):
        $$ t_{8} = \frac{L}{2 \sqrt{2} \cos \alpha} $$
        where $\alpha$ is the angle between the pixel diagonal and the line normal. [S2]
    *   For **4-separability** (2D):
        $$ t_{4} = \frac{L}{2 \cos \beta} $$
        where $\beta$ is the angle between the line normal and the pixel edge normal. [S2]

3.  **Define Slab Boundaries:**
    Construct two parallel planes (or lines in 2D) offset from the original surface by $\pm t_N$.
    *   3D Planes $G$ and $H$:
        $$ A_p X + B_p Y + C_p Z + D_p \pm t_N = 0 $$
    *   2D Lines $G$ and $H$:
        $$ A_p x + B_p y + D_p \pm t_N = 0 $$
    [S2]

4.  **Evaluate Voxels/Pixels:**
    For each voxel center $(x, y, z)$ or pixel center $(x, y)$, calculate the signed distance value $val$ from the primitive equation.
    *   3D: $val = A_p x + B_p y + C_p z + D_p$
    *   2D: $val = A_p x + B_p y + D_p$
    [S2]

5.  **Classify and Mark:**
    *   If $-t_N \leq val \leq t_N$, mark the voxel/pixel as **black** (occupied, value 1).
    *   Otherwise, mark it as **white** (empty, value 0).
    [S2]

6.  **Handle Degeneracies:**
    Use infinitesimal perturbation to ensure that if the surface passes exactly through a corner, edge, or face, the intersection is treated as occurring inside a voxel, never exactly on the boundary. This prevents ambiguous classification. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_plane(plane_coeffs, grid_bounds, voxel_size, separability):
    """
    Voxelize a 3D plane with guaranteed N-separability.
    
    Args:
        plane_coeffs: Tuple (Ap, Bp, Cp, Dp) for plane Ap*X + Bp*Y + Cp*Z + Dp = 0
        grid_bounds: Tuple (min_x, min_y, min_z, max_x, max_y, max_z)
        voxel_size: Float L, the width of a voxel
        separability: Integer N, either 6 or 26
        
    Returns:
        A 3D boolean array or sparse set of occupied voxel coordinates.
    """
    Ap, Bp, Cp, Dp = plane_coeffs
    
    # Step 1: Normalize the normal vector
    norm = math.sqrt(Ap**2 + Bp**2 + Cp**2)
    if norm == 0:
        raise ValueError("Plane normal cannot be zero")
    Ap, Bp, Cp = Ap/norm, Bp/norm, Cp/norm
    
    # Step 2: Calculate thickness threshold t_N
    # Determine angle alpha (between normal and voxel diagonal) or beta (between normal and face normal)
    # For 26-separability, we use the voxel diagonal. The max projection of a diagonal of length L*sqrt(3)
    # onto the normal determines the worst-case gap.
    # The distance from center to the farthest corner is (L/2)*sqrt(3).
    # The condition for 26-separation requires t_26 >= (L/2)*sqrt(3) * cos(alpha)? 
    # Source formula: t_26 = L / (2 * sqrt(3) * cos(alpha))
    # Note: cos(alpha) = |N . K| / (|N| |K|). Since N is unit, |K|=sqrt(3).
    # Let's compute cos(alpha) explicitly from the normal components.
    # The diagonal vector K can be (1,1,1) in local grid coords.
    # cos_alpha = abs(Ap*1 + Bp*1 + Cp*1) / sqrt(3)
    cos_alpha = abs(Ap + Bp + Cp) / math.sqrt(3)
    
    if separability == 26:
        if cos_alpha == 0:
            t_N = float('inf') # Degenerate case, plane parallel to diagonal? Unlikely for unit normal.
        else:
            t_N = voxel_size / (2 * math.sqrt(3) * cos_alpha)
    elif separability == 6:
        # For 6-separability, we consider the face normal.
        # The distance from center to the farthest face is L/2.
        # The condition involves beta, the angle between plane normal and face normal.
        # The worst case is when the plane is parallel to a face (beta=90), but then t_6 would be infinite?
        # Actually, for 6-separation, we just need to cover the gap created by the "staircase" effect along axes.
        # Source formula: t_6 = L / (2 * cos(beta))
        # cos(beta) is the max component of the normal vector (since face normals are (1,0,0), (0,1,0), (0,0,1)).
        cos_beta = max(abs(Ap), abs(Bp), abs(Cp))
        if cos_beta == 0:
            t_N = float('inf')
        else:
            t_N = voxel_size / (2 * cos_beta)
    else:
        raise ValueError("Only 6 and 26 separability are supported for 3D planes in this implementation")

    # Step 3 & 4 & 5: Iterate and classify
    occupied_voxels = []
    
    # Convert bounds to integer grid indices
    min_ix = math.floor(grid_bounds[0] / voxel_size)
    max_ix = math.ceil(grid_bounds[3] / voxel_size)
    min_iy = math.floor(grid_bounds[1] / voxel_size)
    max_iy = math.ceil(grid_bounds[4] / voxel_size)
    min_iz = math.floor(grid_bounds[2] / voxel_size)
    max_iz = math.ceil(grid_bounds[5] / voxel_size)
    
    for x in range(min_ix, max_ix):
        for y in range(min_iy, max_iy):
            for z in range(min_iz, max_iz):
                # Center of the voxel in world space
                cx = (x + 0.5) * voxel_size
                cy = (y + 0.5) * voxel_size
                cz = (z + 0.5) * voxel_size
                
                # Signed distance value
                val = Ap * cx + Bp * cy + Cp * cz + Dp
                
                # Check if within slab
                if -t_N <= val <= t_N:
                    occupied_voxels.append((x, y, z))
                    
    return occupied_voxels

def voxelize_line(line_coeffs, grid_bounds, pixel_size, separability):
    """
    Voxelize a 2D line with guaranteed N-separability.
    
    Args:
        line_coeffs: Tuple (Ap, Bp, Dp) for line Ap*x + Bp*y + Dp = 0
        grid_bounds: Tuple (min_x, min_y, max_x, max_y)
        pixel_size: Float L, the width of a pixel
        separability: Integer N, either 4 or 8
        
    Returns:
        A 2D boolean array or sparse set of occupied pixel coordinates.
    """
    Ap, Bp, Dp = line_coeffs
    
    # Step 1: Normalize
    norm = math.sqrt(Ap**2 + Bp**2)
    if norm == 0:
        raise ValueError("Line normal cannot be zero")
    Ap, Bp = Ap/norm, Bp/norm
    
    # Step 2: Calculate thickness threshold t_N
    if separability == 8:
        # 8-separability uses the pixel diagonal
        # cos_alpha = |Ap*1 + Bp*1| / sqrt(2)
        cos_alpha = abs(Ap + Bp) / math.sqrt(2)
        if cos_alpha == 0:
            t_N = float('inf')
        else:
            t_N = pixel_size / (2 * math.sqrt(2) * cos_alpha)
    elif separability == 4:
        # 4-separability uses the pixel edge
        # cos_beta = max(|Ap|, |Bp|)
        cos_beta = max(abs(Ap), abs(Bp))
        if cos_beta == 0:
            t_N = float('inf')
        else:
            t_N = pixel_size / (2 * cos_beta)
    else:
        raise ValueError("Only 4 and 8 separability are supported for 2D lines in this implementation")

    # Step 3 & 4 & 5: Iterate and classify
    occupied_pixels = []
    
    min_ix = math.floor(grid_bounds[0] / pixel_size)
    max_ix = math.ceil(grid_bounds[2] / pixel_size)
    min_iy = math.floor(grid_bounds[1] / pixel_size)
    max_iy = math.ceil(grid_bounds[3] / pixel_size)
    
    for x in range(min_ix, max_ix):
        for y in range(min_iy, max_iy):
            cx = (x + 0.5) * pixel_size
            cy = (y + 0.5) * pixel_size
            
            val = Ap * cx + Bp * cy + Dp
            
            if -t_N <= val <= t_N:
                occupied_pixels.append((x, y))
                
    return occupied_pixels
```

**Edge Cases and Pitfalls:**
*   **26-Holes:** If a voxel is intersected by the plane but excluded, it creates a 26-hole. The proof demonstrates that if such a voxel $P$ exists with neighbors $Q$ and $R$ on opposite sides not included, the distance from the center $C$ to the plane must be less than or equal to $t_{26}$, meaning $P$ should have been included [S2].
*   **Degeneracies:** Surface passing through a corner, edge, or face must be handled via infinitesimal perturbation to ensure intersection happens inside a voxel, never exactly at the boundary [S1, unofficial].
*   **Performance:** The method iterates over a bounding box. For large grids, this can be slow. An efficient incremental algorithm is needed for faster voxelization, as the current implementation is not optimized [S2].

## Subtopics

### 4-Separating Line Voxelization
A method to generate a discrete representation of a 2D line that is 4-separating (no 4-holes) and 4-minimal (no redundant voxels). It uses a thickness $t_4 = \frac{L}{2 \cos \beta}$, where $\beta$ is the angle between the line normal and the pixel edge normal. If the threshold is not calculated correctly based on the angle, pixels that should be included might be excluded, creating 4-holes [S2].

### 8-Separating Line Voxelization
A method to generate a discrete representation of a 2D line that is 8-separating (no 8-holes) and 8-minimal. It uses a thickness $t_8 = \frac{L}{2 \sqrt{2} \cos \alpha}$, where $\alpha$ is the angle between the line normal and the pixel diagonal. It guarantees that 8-connected paths cannot penetrate the line [S2].

## Operating characteristics
*   **Time Complexity:** $O(N_{voxels})$, where $N_{voxels}$ is the number of voxels in the bounding box of the primitive. The method requires iterating over all candidate voxels/pixels.
*   **Memory Complexity:** $O(N_{occupied})$ for storing the result, or $O(N_{bounding\_box})$ if using a dense grid.
*   **Measured Performance:** VPM (Voxels per millisecond) is a metric reflecting efficiency. Performance decreases as the number of voxels increases due to linked list maintenance costs in non-optimized implementations [S2].
*   **Strengths:** Guarantees topological correctness (separability and minimality). Eliminates common voxelization artifacts at edges and vertices. Visually indistinguishable from less strict methods for display purposes [S2].
*   **Limitations:** 26-separating voxelization produces significantly more Vertex and Edge Voxels than 6-separating voxelization. The method is computationally expensive if not optimized incrementally [S2].
*   **When to use:** When topological integrity is critical, such as in ray tracing against voxelized surfaces or when ensuring no gaps exist in discrete representations.
*   **When not to use:** When performance is paramount and topological gaps are acceptable (e.g., simple visualization).

## Background
The method originates from the need for accurate voxelization of polygon meshes that guarantees both separability and minimality, addressing deficiencies in early scan-conversion algorithms that did not provide mechanisms for topological correctness, especially at edges and vertices [S2]. The concept of 26-separating voxelization was analyzed by Cohen-Or and Kaufman [COK97, COK95], who showed that a full 3D voxel cover is 26-separating, but this method improves efficiency by using 1D intersection targets for effectively 2D inputs [S1, unofficial]. The specific slab-based thickness criterion for N-separability was presented in "An Accurate Method for Voxelizing Polygon Meshes" [S2].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
[S2] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
