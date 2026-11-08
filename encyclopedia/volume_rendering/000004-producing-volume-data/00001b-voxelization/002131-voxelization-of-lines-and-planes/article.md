# Voxelization of Lines and Planes

Also called: Slab-based Voxelization, N-separating Voxelization, 6-separating voxelization, 26-separating voxelization, 26-Separating Plane Voxelization

## Overview
Voxelization of Lines and Planes is a method for generating discrete, topologically correct representations of 2D lines and 3D planes within a voxel grid. It solves the problem of ensuring that the resulting voxel set is both **N-separating** (preventing N-connected ray paths from penetrating the surface) and **N-minimal** (containing no redundant voxels). Unlike general mesh voxelization methods that may produce artifacts at edges or require heavy computation, this method uses a geometric slab defined by the surface normal and grid resolution to deterministically include or exclude voxels based on their center's signed distance to the plane [S1, unofficial][S2]. It is distinct from [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by relying on analytic distance criteria rather than projection, and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by explicitly handling continuous surface geometry [S2].

## How it works
The method operates by defining a "slab" of thickness $2t$ centered on the input geometric primitive (line or plane). Voxels whose centers fall within this slab are marked as occupied. The thickness $t$ is calculated based on the desired separability (4, 6, 8, or 26) and the grid resolution.

### Inputs and Outputs
*   **Inputs**:
    *   Geometric primitive: A 2D line defined by $A_p x + B_p y + D_p = 0$ or a 3D plane defined by $A_p x + B_p y + C_p z + D_p = 0$.
    *   Normal vector $(A_p, B_p, C_p)$: Must be normalized such that $\sqrt{A_p^2 + B_p^2 + C_p^2} = 1$ [S2].
    *   Grid resolution: Voxel width $L$ (typically 1 in normalized space, or physical size) [S2].
    *   Separability type $N$: Either 6 or 26 for 3D planes; 4 or 8 for 2D lines [S2].
*   **Outputs**:
    *   A set of voxel coordinates $\tilde{S}$ representing the discrete surface.

### Data Structures
*   **Voxel Set $\tilde{S}$**: A collection of integer grid indices $(x, y, z)$ (or $(x, y)$ for 2D).
*   **Plane Parameters**: Floating-point values $A_p, B_p, C_p, D_p$.

### Algorithm Steps

1.  **Normalize Normal Vector**: Ensure the normal vector $\mathbf{n} = (A_p, B_p, C_p)$ satisfies $\|\mathbf{n}\| = 1$. Note: Source [S2] contains a typo in the normalization formula ($\frac{A_p^2 + B_p^2 + C_p^2}{2} = 1$), but the standard interpretation and context confirm unit length [S2].
2.  **Calculate Thickness $t$**: Compute the slab half-thickness $t$ based on $N$ and $L$.
    *   For 3D Planes:
        *   $t_{26} = \frac{L}{2 \sqrt{3} \cos \alpha}$, where $\alpha$ is the angle between the voxel diagonal and the plane normal [S2].
        *   $t_{6} = \frac{L}{2 \cos \beta}$, where $\beta$ is the angle between the plane normal and the normal to the voxel face [S2].
    *   For 2D Lines:
        *   $t_{8} = \frac{L}{2 \cos \alpha}$, where $\alpha$ is the angle between the pixel diagonal and the line normal [S2].
        *   $t_{4} = \frac{L}{2 \cos \beta}$, where $\beta$ is the angle between the line normal and the pixel axis [S2].
3.  **Iterate Voxels**: For each voxel center $(x, y, z)$ in the bounding volume of interest:
    *   Calculate the signed distance value $val = A_p x + B_p y + C_p z + D_p$.
    *   Check if $-t \leq val \leq t$.
4.  **Assign Occupancy**:
    *   If the condition holds, mark the voxel as occupied (1/black).
    *   Otherwise, mark as empty (0/white) [S2].
5.  **Handle Degeneracies**: Use infinitesimal perturbation to ensure that if the surface passes exactly through a corner, edge, or face, the intersection is treated as occurring strictly inside a voxel, preventing ambiguous boundary cases [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_plane_3d(Ap, Bp, Cp, Dp, L, separability_type):
    """
    Voxelizes a 3D plane for 6-separating or 26-separating output.
    
    Args:
        Ap, Bp, Cp, Dp: Plane equation coefficients Ax + By + Cz + D = 0.
        L: Voxel width (grid spacing).
        separability_type: '6' or '26'.
        
    Returns:
        A list of voxel coordinates (x, y, z) that are occupied.
    """
    # 1. Normalize normal vector
    norm = math.sqrt(Ap**2 + Bp**2 + Cp**2)
    if norm == 0:
        raise ValueError("Normal vector cannot be zero")
    Ap, Bp, Cp = Ap/norm, Bp/norm, Cp/norm
    
    # 2. Calculate thickness t
    # Angle alpha: between voxel diagonal and plane normal
    # Diagonal vector is (1, 1, 1) normalized -> (1/sqrt(3), 1/sqrt(3), 1/sqrt(3))
    # cos(alpha) = dot(normal, diagonal_unit)
    cos_alpha = abs(Ap/smath.sqrt(3) + Bp/smath.sqrt(3) + Cp/smath.sqrt(3))
    
    # Angle beta: between plane normal and face normal (e.g., x-axis)
    # cos(beta) = abs(dot(normal, (1,0,0))) = abs(Ap)
    # However, the source defines t_6 using beta as angle between normal and normal to face.
    # To guarantee 6-separability for ANY orientation, we must consider the worst-case face alignment.
    # The source formula is t_6 = L / (2 * cos(beta)). 
    # In the proof, beta is the angle between the normal N and the normal to the face of the voxel.
    # For a general plane, we need to ensure separation for all 6 faces.
    # The source implies t_6 is derived specifically for the 6-neighbor connectivity.
    # We use the provided formula structure.
    
    if separability_type == '26':
        if cos_alpha == 0:
            t = L / 2 # Fallback if normal is perpendicular to diagonal (impossible for unit normal in 3D grid unless degenerate)
        else:
            t = L / (2 * math.sqrt(3) * cos_alpha)
    elif separability_type == '6':
        # For 6-separability, the thickness depends on the angle with the face normals.
        # The source gives t_6 = L / (2 * cos(beta)).
        # To be robust, we take the maximum required thickness over the 3 axes?
        # Or simply use the formula as stated for the specific orientation.
        # Given the source text "t_6 = L / (2 cos beta)", and beta is angle between N and face normal.
        # We calculate cos_beta for the primary face normal? 
        # Actually, for 6-connectivity, the critical direction is the face normal.
        # We will use the absolute value of the normal components as cosines of angles with axes.
        # The tightest bound requires the maximum of these? 
        # Source Theorem 4 uses t_6. Let's assume the caller provides the specific beta or we compute it.
        # For a general implementation, we might need to check all 6 directions.
        # However, the source formula is specific. We will implement the direct formula.
        # Note: Source [S2] defines t_6 = L / (2 cos beta). 
        # Let's assume beta is the angle with the normal of the face that the ray might traverse.
        # For a robust 6-separating slab, we typically use t = L/2 * max(|nx|, |ny|, |nz|)? 
        # No, the source is explicit. We follow the source.
        # We will calculate cos_beta as the maximum absolute component of the normal, 
        # representing the worst-case alignment with a face normal? 
        # Actually, the proof for 6-separability relies on the distance to the face.
        # Let's stick to the variable name beta from the source.
        cos_beta = max(abs(Ap), abs(Bp), abs(Cp)) # Approximation for worst-case face angle
        if cos_beta == 0:
            t = L / 2
        else:
            t = L / (2 * cos_beta)
    else:
        raise ValueError("Separability type must be '6' or '26'")

    # 3. Determine Bounding Box
    # We need to iterate over voxels that could possibly intersect the slab.
    # The slab extends from -t to t in signed distance.
    # We can estimate the bounding box by projecting the slab onto the axes.
    # However, for simplicity in this generic method, we assume a known bounding box 
    # or iterate over a region of interest.
    # Here we return the logic for a single voxel check.
    
    def is_occupied(x, y, z):
        val = Ap * x + Bp * y + Cp * z + Dp
        return -t <= val <= t

    return is_occupied

def voxelize_line_2d(Ap, Bp, Dp, L, separability_type):
    """
    Voxelizes a 2D line for 4-separating or 8-separating output.
    """
    # 1. Normalize
    norm = math.sqrt(Ap**2 + Bp**2)
    if norm == 0:
        raise ValueError("Normal vector cannot be zero")
    Ap, Bp = Ap/norm, Bp/norm
    
    # 2. Calculate thickness
    if separability_type == '8':
        # Angle alpha between pixel diagonal and line normal
        # Diagonal (1,1) -> (1/sqrt(2), 1/sqrt(2))
        cos_alpha = abs(Ap/smath.sqrt(2) + Bp/smath.sqrt(2))
        if cos_alpha == 0:
            t = L / 2
        else:
            t = L / (2 * math.sqrt(2) * cos_alpha) # Note: Source says t_8 = L / (2 cos alpha) where alpha is angle with diagonal?
            # Re-reading Source [S2]: "t_8 = L / (2 cos alpha), where alpha is angle between normal and pixel diagonal."
            # Wait, the formula in the text block says: t_8 = L / (2 cos alpha).
            # But in the derivation, it relates to the diagonal length.
            # Let's use the explicit formula provided: t_8 = L / (2 cos alpha).
            # If alpha is the angle with the diagonal, cos(alpha) is the dot product.
            pass 
        # Correction: The source text says: "t_8 = L / (2 cos alpha)".
        # And "alpha is angle between normal and pixel diagonal".
        # So we just use that.
        t = L / (2 * cos_alpha)
        
    elif separability_type == '4':
        # Angle beta between normal and pixel axis (face normal equivalent)
        cos_beta = max(abs(Ap), abs(Bp))
        if cos_beta == 0:
            t = L / 2
        else:
            t = L / (2 * cos_beta)
            
    def is_occupied(x, y):
        val = Ap * x + Bp * y + Dp
        return -t <= val <= t
        
    return is_occupied
```

### Parameters
*   $L$: Voxel width. Typical value: 1.0 (normalized) or physical dimension [S2].
*   $N$: Adjacency type. Values: $\{6, 26\}$ for 3D, $\{4, 8\}$ for 2D [S2].
*   $\alpha$: Angle between the normal and the diagonal (for 26/8-separability) [S2].
*   $\beta$: Angle between the normal and the face/axis normal (for 6/4-separability) [S2].

### Edge Cases and Pitfalls
*   **Degeneracies**: If the surface passes exactly through a voxel corner, edge, or face, standard intersection tests may fail or produce inconsistent results. The method requires **infinitesimal perturbation** to ensure the intersection is treated as occurring strictly inside a voxel [S1, unofficial].
*   **26-Holes**: If a voxel is intersected by the plane but excluded due to an incorrect thickness calculation, a 26-hole (penetration path) is created. The proof shows that if such a voxel $P$ exists with neighbors $Q$ and $R$ on opposite sides not included, the distance from $P$'s center to the plane must be $\leq t_{26}$, meaning $P$ should have been included [S2].
*   **Performance**: The method described is a non-optimized scan conversion. An efficient incremental algorithm is needed for real-time performance, as the current approach recalculates distances for all voxels [S2].

## Subtopics

### [4-Separating Line Voxelization](2502-4-separating-line-voxelization.md)
A method to generate a discrete representation of a 2D line that is 4-separating (no 4-holes) and 4-minimal (no redundant voxels). Choose this when strict 4-connectivity separation is required for 2D surfaces.

## Operating characteristics
*   **Time Complexity**: $O(V)$, where $V$ is the number of voxels in the bounding volume. The method requires iterating over all candidate voxels and performing a constant-time distance check.
*   **Memory Complexity**: $O(V)$ to store the resulting voxel set.
*   **Performance**: Measured in VPM (Voxels per millisecond). Performance decreases as the number of voxels increases due to linked list maintenance costs in the implementation [S2]. For example, the 'Connector' scene (larger polygons) had higher VPM than 'Pump' (smaller polygons) [S2].
*   **Strengths**: Guarantees topological correctness (separability and minimality). Eliminates common artifacts at edges and vertices. Visually indistinguishable from less strict methods for display purposes [S2].
*   **Limitations**: 26-separating voxelization produces significantly more Vertex and Edge Voxels than 6-separating voxelization, though Body Voxels are similar. Requires careful handling of degeneracies [S1, unofficial][S2].
*   **When to Use**: When topological correctness is critical (e.g., ray casting, collision detection) and the input is a planar surface or line.
*   **When Not to Use**: When performance is the primary concern and topological gaps are acceptable (use 6-separating or rasterization). Not suitable for complex curved surfaces without tessellation [S2].

## Background
The method originates from the need for topologically correct voxelization of polygon meshes. Early work focused on 2D polygons or 2D projections. Cohen-Or and Kaufman [COK97] provided in-depth analysis of separability. The specific slab-based method for 6- and 26-separating voxelization was presented to guarantee both separability and minimality, addressing deficiencies in existing algorithms that failed to handle edges and vertices correctly [S2]. It is used in surface graphics, ray tracing, and volume rendering pipelines where accurate discretization of surfaces is required [S1, unofficial][S2].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
[S2] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
