# 4-Separating Line Voxelization

Also called: 4-separating line representation, 4-minimal line voxelization

## Overview
A method to generate a discrete, 4-separating, and 4-minimal voxel representation of a 2D line segment [S1]. It solves the problem of ensuring topological correctness (no 4-holes) and efficiency (no redundant voxels) for line primitives in a grid. It sets itself apart from standard scanline algorithms by dynamically calculating a thickness threshold $t_4$ based on the line's angle relative to the grid axes, rather than using a fixed width or simple midpoint tests [S1].

## How it works
**Inputs:**
- Line equation parameters $A_p, B_p, D_p$ defining the line $A_p x + B_p y + D_p = 0$.
- Grid voxel size $L$ (assumed uniform).
- Pixel/Voxel center coordinates $(x, y)$.

**Outputs:**
- A set of voxels $\tilde{\mathcal{S}}$ representing the line.

**Data Structures:**
- The output is a boolean mask or a set of integer coordinates $(x, y)$ belonging to $\tilde{\mathcal{S}}$.

**Parameters:**
- $t_4$: The half-thickness of the slab. Its value depends on the angle $\alpha$ between the line's normal and the voxel diagonal.
- $\alpha$: The angle between the normal vector $\mathbf{N}$ to the line and the diagonal vector $\mathbf{K}$ of the voxel.

**Formulas:**
The threshold $t_4$ is defined as:
$$ t_4 = \frac{L}{2\sqrt{2}\cos\alpha} $$
where $\alpha$ is the angle between the line normal and the voxel diagonal. Note that $\cos\alpha$ relates to the orientation of the line relative to the grid. Specifically, if the line normal is $(A_p, B_p)$, the angle $\alpha$ determines the maximum distance from the center to the line edge that still requires inclusion to prevent 4-holes [S1].

**Steps:**
1. Calculate the threshold $t_4$ based on the line's orientation angle $\alpha$.
2. For each voxel center $(x, y)$ in the bounding box of the line segment:
   a. Compute the signed distance value $d = A_p x + B_p y + D_p$.
   b. Check if $|d| \leq t_4$.
   c. If true, include $(x, y)$ in $\tilde{\mathcal{S}}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_line_4_separating(Ap, Bp, Dp, L, x_min, x_max, y_min, y_max):
    """
    Voxelize a 2D line defined by Ap*x + Bp*y + Dp = 0.
    Returns a set of (x, y) coordinates.
    """
    # Calculate angle alpha between line normal (Ap, Bp) and voxel diagonal (1, 1)
    # Normal vector N = (Ap, Bp)
    # Diagonal vector K = (1, 1) (assuming unit square voxels for angle calc)
    # cos(alpha) = (N . K) / (|N| * |K|)
    
    dot_product = Ap * 1 + Bp * 1
    norm_N = sqrt(Ap**2 + Bp**2)
    norm_K = sqrt(1**2 + 1**2) # sqrt(2)
    
    # Guard against division by zero if line is degenerate
    if norm_N == 0:
        return set()
        
    cos_alpha = dot_product / (norm_N * norm_K)
    
    # Theorem 2 states t4 = L / (2 * sqrt(2) * cos(alpha))
    # Note: The source derivation uses L/(2*sqrt(2)*cos(alpha)) for t4.
    # However, standard geometric projection suggests t4 = L / (2 * max(|cos(theta)|, |sin(theta)|))
    # We follow the source's explicit formula for t4.
    
    # Avoid division by zero if cos_alpha is 0 (line parallel to diagonal? No, normal parallel to diagonal means line is at 45 deg)
    if abs(cos_alpha) < 1e-9:
        # Fallback or error handling for degenerate angle
        t4 = L / 2.0 
    else:
        t4 = L / (2.0 * sqrt(2) * abs(cos_alpha))
        
    voxel_set = set()
    
    for x in range(x_min, x_max + 1):
        for y in range(y_min, y_max + 1):
            # Distance from center (x, y) to line Ap*x + Bp*y + Dp = 0
            # The source uses the unnormalized value Apx + Bpy + Dp compared to t4.
            # This implies Ap and Bp are normalized such that sqrt(Ap^2+Bp^2)=1?
            # The source text says "distance ... is ... where ...". 
            # Theorem 2 uses: -t4 <= Ap*x + Bp*y + Dp <= t4.
            # This implies the input Ap, Bp, Dp must be normalized or t4 is scaled accordingly.
            # Standard line equation distance is |Ax+By+C|/sqrt(A^2+B^2).
            # The source formula in Theorem 2 does NOT divide by norm.
            # This implies Ap, Bp are normalized (unit normal) OR t4 is defined in "equation space".
            # Given "distance from pixel center to the line", it usually implies Euclidean distance.
            # If Euclidean, we must normalize. If the source implies unnormalized, we follow source.
            # Source: "distance ... is ...". Then "t4 = L / (2*sqrt(2)*cos(alpha))".
            # Let's assume Euclidean distance for correctness, requiring normalization.
            
            # Re-reading Source S1: "distance between the center of the pixel and the line is ... "
            # Theorem 2: "set of pixels ... -t4 <= Ap x + Bp y + Dp <= t4"
            # This suggests Ap, Bp are normalized.
            
            # Calculate normalized coefficients if not already provided
            norm = sqrt(Ap**2 + Bp**2)
            if norm > 0:
                An = Ap / norm
                Bn = Bp / norm
                Dn = Dp / norm
            else:
                return set()
                
            val = An * x + Bn * y + Dn
            
            if abs(val) <= t4:
                voxel_set.add((x, y))
                
    return voxel_set
```

**Edge Cases and Pitfalls:**
- If the threshold $t_4$ is not calculated correctly based on the angle $\alpha$, pixels that should be included might be excluded, creating 4-holes [S1].
- The proof shows that if a pixel is intersected by the line but excluded, it leads to a contradiction regarding the distance from the center $C$ to the line [S1].
- Degenerate lines (zero length or undefined normal) must be handled to avoid division by zero in normalization [G].

## Subtopics

### [Optimal Scanline Voxelization (Floating-Point)](10209-optimal-scanline-voxelization-floating-point.md)
Extends the 4-separating concept to fill triangle slices by dynamically calculating step sizes to skip covered voxels.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the bounding box of the line segment. Each voxel requires a constant number of arithmetic operations.
- **Memory Complexity:** $O(K)$ where $K$ is the number of voxels in the final representation.
- **Strengths:** Guarantees 4-separability (no 4-holes) and 4-minimality (no redundant voxels) for 2D lines [S1].
- **Limitations:** The method is defined for 2D lines. For 3D planes, the source indicates a similar slab-based approach is used but does not provide the specific 3D threshold formula in the provided text, only referencing 6- or 26-separating representations [S1].
- **When to use:** When topological correctness (connectivity) of line primitives is critical and minimal voxel count is desired.

## Background
Originated in the paper "An Accurate Method for Voxelizing Polygon Meshes" [S1]. The method is part of a broader family of geometric voxelization techniques that rely on distance-based slab inclusion to ensure topological properties like separability and minimality.

## Sources
[S1] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
