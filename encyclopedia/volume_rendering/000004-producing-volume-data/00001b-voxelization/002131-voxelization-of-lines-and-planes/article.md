# Voxelization of Lines and Planes

Also called: Slab-based Voxelization, N-separating Voxelization, 6-separating voxelization, 26-separating voxelization, 6-Separating Plane Voxelization, 26-Separating Plane Voxelization

## Overview
This method generates a discrete, topologically correct representation of 2D lines and 3D planes (and by extension, polygon meshes) that is both **N-separating** and **N-minimal** [S2]. It solves the problem of "holes" in voxelized surfaces where a connected ray path could penetrate the original continuous surface without intersecting any voxel [S1, unofficial]. Unlike standard rasterization which may produce gaps, this method uses a geometric slab defined by the surface normal and grid resolution to guarantee that no N-connected path (6-connected or 26-connected) can pass through the discretization without hitting a voxel [S2]. It sets itself apart from siblings like [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by providing formal topological proofs of separability and minimality rather than relying on heuristic coverage [S2].

## How it works
The method maps continuous world-space coordinates to a discrete grid by evaluating the signed distance of voxel centers to the input primitive. If the distance is within a calculated threshold $t$, the voxel is included.

### Inputs and Outputs
*   **Input**: 
    *   For 3D planes: Plane equation $A_p X + B_p Y + C_p Z + D_p = 0$ with normalized normal $(A_p, B_p, C_p)$ [S2].
    *   For 2D lines: Line equation $A_p x + B_p y + D_p = 0$ with normalized normal $(A_p, B_p)$ [S2].
    *   Grid resolution $L$ (voxel width/size) [S2].
    *   Desired separability type $N \in \{6, 26\}$ for 3D, or $N \in \{4, 8\}$ for 2D [S2].
*   **Output**: A set of voxels $\tilde{S}$ (or pixels $\tilde{S}$ in 2D) where each voxel is marked as "black" (1/occupied) or "white" (0/empty) [S2].

### Data Structures
*   **Voxel Grid**: A 3D array or sparse structure indexed by integer coordinates $(x, y, z)$.
*   **Plane Parameters**: Floating-point values $A_p, B_p, C_p, D_p$. The normal vector must be normalized such that $\sqrt{A_p^2 + B_p^2 + C_p^2} = 1$ [S2]. Note: Source [S2] contains a typo in the OCR'd formula block stating $\frac{A_p^2 + B_p^2 + C_p^2}{2} = 1$, but the standard normalization condition $\sqrt{A_p^2 + B_p^2 + C_p^2} = 1$ is explicitly written in Equation (3) of the source text [S2].

### Formulas
The inclusion condition for a voxel/pixel at center $(x, y, z)$ or $(x, y)$ is:
$$ -t \leq A_p x + B_p y + C_p z + D_p \leq t $$
$$ -t \leq A_p x + B_p y + D_p \leq t \quad (\text{for 2D}) $$

The threshold $t$ depends on the voxel size $L$ and the angle between the grid diagonal and the surface normal:

**For 3D Planes:**
*   **26-separating**: $t_{26} = \frac{L}{2 \sqrt{3} \cos \alpha}$, where $\alpha$ is the angle between the voxel diagonal $K$ and the plane normal $N$ [S2].
*   **6-separating**: $t_{6} = \frac{L}{2 \cos \beta}$, where $\beta$ is the angle between the plane normal and the normal to the face of the voxel (i.e., the axis-aligned direction) [S2].

**For 2D Lines:**
*   **8-separating**: $t_{8} = \frac{L}{2 \sqrt{2} \cos \alpha}$, where $\alpha$ is the angle between the pixel diagonal and the line normal [S2].
*   **4-separating**: $t_{4} = \frac{L}{2 \cos \beta}$, where $\beta$ is the angle between the line normal and the axis-aligned direction [S2].

### Steps
1.  **Normalize Plane/Line**: Ensure the normal vector $(A_p, B_p, C_p)$ has unit length [S2].
2.  **Calculate Threshold $t$**:
    *   Determine the angle $\alpha$ between the normal and the relevant diagonal (space diagonal for 26/8, face normal for 6/4).
    *   Compute $t$ using the formulas above [S2].
3.  **Iterate Voxels**: For each voxel $(x, y, z)$ in the bounding volume:
    *   Calculate the signed distance value: $val = A_p x + B_p y + C_p z + D_p$ [S2].
    *   Check if $-t \leq val \leq t$.
    *   If true, mark voxel as occupied (1); otherwise, mark as empty (0) [S2].
4.  **Handle Degeneracies**: Use infinitesimal perturbation to ensure that if the surface passes exactly through a corner, edge, or face, the intersection is treated as occurring inside a voxel, preventing ambiguous boundary cases [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_plane(plane_coeffs, grid_origin, grid_size, resolution, separability_type):
    """
    Voxelize a 3D plane.
    plane_coeffs: (Ap, Bp, Cp, Dp)
    grid_origin: (ox, oy, oz)
    grid_size: L (voxel width)
    resolution: (rx, ry, rz)
    separability_type: '6' or '26'
    """
    Ap, Bp, Cp, Dp = plane_coeffs
    
    # Normalize normal vector
    norm_len = math.sqrt(Ap**2 + Bp**2 + Cp**2)
    Ap /= norm_len
    Bp /= norm_len
    Cp /= norm_len
    
    # Calculate threshold t
    # Angle alpha between normal N and voxel diagonal K
    # Diagonal vector K = (1, 1, 1) normalized
    K = (1/math.sqrt(3), 1/math.sqrt(3), 1/math.sqrt(3))
    
    # Cosine of angle between N and K
    cos_alpha = abs(Ap * K[0] + Bp * K[1] + Cp * K[2])
    
    if separability_type == '26':
        t = grid_size / (2 * math.sqrt(3) * cos_alpha)
    elif separability_type == '6':
        # For 6-separability, beta is angle between normal and face normal (axis)
        # The source defines t_6 = L / (2 * cos(beta))
        # However, standard implementation often uses L/2 for axis-aligned checks 
        # or derives beta from the normal components. 
        # Based on source [S2]: t_6 = L / (2 * cos(beta))
        # We need to determine beta. Beta is the angle between N and the normal to the face.
        # Since the face normals are (1,0,0), (0,1,0), (0,0,1), 
        # the "worst case" or specific beta depends on implementation.
        # The source implies a specific geometric derivation. 
        # For simplicity in a general implementation, if cos_alpha is small (normal is axis aligned),
        # we might use a different logic, but we follow the formula structure.
        # Note: Source [S2] states t_6 = L / (2 * cos(beta)). 
        # If the normal is (1,0,0), beta=0, cos(beta)=1, t=L/2.
        # If the normal is (1,1,1)/sqrt(3), the angle to face normal (1,0,0) is arccos(1/sqrt(3)).
        # cos(beta) = 1/sqrt(3). t = L / (2/sqrt(3)) = L*sqrt(3)/2.
        # This matches the 26-separating formula if we consider the diagonal vs face.
        # Let's assume beta is the angle between N and the closest axis-aligned normal.
        cos_beta = max(abs(Ap), abs(Bp), abs(Cp))
        if cos_beta == 0:
            cos_beta = 1e-9 # Guard against division by zero
        t = grid_size / (2 * cos_beta)
    else:
        raise ValueError("Unsupported separability type")

    # Initialize grid
    # Using a dictionary for sparse storage or a 3D array
    voxels = {} 
    
    # Iterate over bounding box
    # Calculate bounds based on plane intersection with grid extent
    # Simplified: iterate all voxels in resolution
    for x in range(resolution[0]):
        for y in range(resolution[1]):
            for z in range(resolution[2]):
                # World coordinate of voxel center
                wx = grid_origin[0] + (x + 0.5) * grid_size
                wy = grid_origin[1] + (y + 0.5) * grid_size
                wz = grid_origin[2] + (z + 0.5) * grid_size
                
                # Signed distance
                val = Ap * wx + Bp * wy + Cp * wz + Dp
                
                # Check inclusion
                if -t <= val <= t:
                    voxels[(x, y, z)] = 1
                else:
                    voxels[(x, y, z)] = 0
                    
    return voxels

def voxelize_line(line_coeffs, grid_origin, grid_size, resolution, separability_type):
    """
    Voxelize a 2D line.
    line_coeffs: (Ap, Bp, Dp)
    grid_origin: (ox, oy)
    grid_size: L
    resolution: (rx, ry)
    separability_type: '4' or '8'
    """
    Ap, Bp, Dp = line_coeffs
    
    # Normalize
    norm_len = math.sqrt(Ap**2 + Bp**2)
    Ap /= norm_len
    Bp /= norm_len
    
    if separability_type == '8':
        # Angle alpha between normal and pixel diagonal (1,1)
        diag = (1/math.sqrt(2), 1/math.sqrt(2))
        cos_alpha = abs(Ap * diag[0] + Bp * diag[1])
        if cos_alpha == 0: cos_alpha = 1e-9
        t = grid_size / (2 * math.sqrt(2) * cos_alpha)
    elif separability_type == '4':
        # Angle beta between normal and axis
        cos_beta = max(abs(Ap), abs(Bp))
        if cos_beta == 0: cos_beta = 1e-9
        t = grid_size / (2 * cos_beta)
    else:
        raise ValueError("Unsupported separability type")

    pixels = {}
    for x in range(resolution[0]):
        for y in range(resolution[1]):
            wx = grid_origin[0] + (x + 0.5) * grid_size
            wy = grid_origin[1] + (y + 0.5) * grid_size
            
            val = Ap * wx + Bp * wy + Dp
            
            if -t <= val <= t:
                pixels[(x, y)] = 1
            else:
                pixels[(x, y)] = 0
                
    return pixels
```

### Parameters and Typical Values
*   $L$: Voxel width. Determined by the scene scale and desired resolution [S2].
*   $N$: Adjacency type. Must be 6 or 26 for 3D, 4 or 8 for 2D [S2].
*   Grid Resolution: Example usage in source [S2] used $256 \times 256 \times 256$ for the 'Connector' scene [S2].

### Edge Cases and Pitfalls
*   **26-Holes**: If a voxel is intersected by the plane but excluded, it creates a 26-hole. The proof shows that if such a voxel $P$ exists with neighbors $Q$ and $R$ on opposite sides not included, the distance from the center $C$ to the plane must be $\leq t_{26}$, meaning $P$ should have been included [S2].
*   **Degeneracies**: Surfaces passing exactly through corners, edges, or faces must be handled via infinitesimal perturbation to ensure intersection happens inside a voxel, never exactly at the boundary [S1, unofficial].
*   **Performance**: The method described is a non-optimized scan conversion. An efficient incremental algorithm is needed for faster voxelization, especially for large meshes [S2].
*   **Vertex/Edge Voxels**: 26-separating voxelization produces significantly more Vertex and Edge voxels than 6-separating voxelization, though Body voxel counts are similar [S2].

## Subtopics

### 4-Separating Line Voxelization
A method to generate a discrete representation of a 2D line that is 4-separating (no 4-holes) and 4-minimal (no redundant voxels) [S2]. It uses the threshold $t_4 = \frac{L}{2 \cos \beta}$ where $\beta$ is the angle between the line normal and the axis-aligned direction [S2]. If the threshold is not calculated correctly based on the angle, pixels that should be included might be excluded, creating 4-holes [S2].

### 8-Separating Line Voxelization
A method to generate a discrete representation of a 2D line that is 8-separating (no 8-holes) and 8-minimal [S2]. It uses the threshold $t_8 = \frac{L}{2 \sqrt{2} \cos \alpha}$ where $\alpha$ is the angle between the line normal and the pixel diagonal [S2].

### 6-Separating Plane Voxelization
A method to generate a discrete representation of a 3D plane that is 6-separating (no 6-holes) and 6-minimal [S2]. It uses the threshold $t_6 = \frac{L}{2 \cos \beta}$ where $\beta$ is the angle between the plane normal and the normal to the face of the voxel [S2]. Visually, 6- and 26-separating models are virtually indistinguishable, but 6-separating is faster to compute [S2].

### 26-Separating Plane Voxelization
A method to generate a discrete representation of a 3D plane that is 26-separating (no 26-holes) and 26-minimal [S2]. It uses the threshold $t_{26} = \frac{L}{2 \sqrt{3} \cos \alpha}$ where $\alpha$ is the angle between the voxel diagonal and the plane normal [S2]. It guarantees that 26-connected ray paths do not penetrate the original separating surface [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The basic implementation iterates over all voxels in the bounding box, resulting in $O(R_x R_y R_z)$ for 3D or $O(R_x R_y)$ for 2D. The source notes that timing results are based on a non-optimized scan conversion and that an efficient incremental algorithm is needed for better performance [S2].
*   **Memory Complexity**: Proportional to the number of voxels in the bounding box. The source mentions that increasing the number of voxels leads to decreasing VPM (Voxels per millisecond) due to linked list maintenance costs if a linked-list data structure is used [S2].
*   **Measured Performance**: In source [S2], the 'Connector' scene (256^3 resolution) showed different VPM values compared to 'Pump' due to average polygon size and voxel count. 6-separating voxelization takes less time than 26-separating [S2].
*   **Strengths**: Guarantees topological correctness (separability and minimality). Eliminates common voxelization artifacts at edges and vertices [S2].
*   **Limitations**: 26-separating voxelization produces significantly more Vertex and Edge voxels than 6-separating [S2]. The basic implementation is slow; optimization is required for real-time or large-scale use [S2].
*   **When to use**: When topological correctness is critical, such as in ray casting against voxelized surfaces where gaps must be avoided [S1, unofficial]. When visual fidelity is required and computational cost allows for 26-separation, or when speed is prioritized with 6-separation [S2].

## Background
The method originates from the need to voxelize polygon meshes with guaranteed topological properties. Early work in scan-converting planar objects focused on filling 2D polygons or 2D projections, often using scan-line algorithms [S2]. Existing algorithms did not guarantee both separability and minimality, especially for polygon meshes where edge and vertex regions posed difficulties [S2]. The specific method for lines and planes was presented to provide a distance criterion based on the normal vector, ensuring numerical accuracy and topological correctness [S2]. The 26-separating property was analyzed by Cohen-Or and Kaufman [COK97, COK95], who showed that a full 3D voxel cover is 26-separating, but this method uses 1D intersection targets for efficiency [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
[S2] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
