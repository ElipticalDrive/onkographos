# 4-Separating Line Voxelization

Also called: 4-separating line representation, 4-minimal line voxelization

## Overview
A method to generate a discrete, 4-separating, and 4-minimal voxel representation of a 2D line segment [S1]. It solves the problem of ensuring topological correctness (no 4-holes) and efficiency (no redundant voxels) for line primitives in a grid [S1]. It sets itself apart from standard scanline algorithms by dynamically calculating a thickness threshold $t_4$ based on the line's angle relative to the grid axes, rather than using a fixed width or simple midpoint tests [S1].

## How it works
**Inputs:**
- Line equation parameters $A_p, B_p, D_p$ defining the line $A_p x + B_p y + D_p = 0$, where $A_p^2 + B_p^2 = 1$ (unit normal vector) [S1].
- Grid voxel size $L$ (assumed uniform) [S1].
- Bounding box coordinates $(x_{min}, x_{max}, y_{min}, y_{max})$ covering the line segment [G].

**Outputs:**
- A set of voxel centers $\tilde{\mathcal{S}}$ representing the line [S1].

**Data Structures:**
- The output is a set of integer coordinate tuples $(x, y)$ belonging to $\tilde{\mathcal{S}}$ [S1].

**Numbered Steps:**
1. Calculate $\cos\alpha$ from the line normal and the grid diagonal direction [S1].
2. Compute the threshold $t_4$ using the geometric formula [S1].
3. Iterate over every integer coordinate $(x, y)$ in the bounding box [S1].
4. Evaluate the signed distance inequality for each coordinate [S1].
5. Add coordinates satisfying the inequality to $\tilde{\mathcal{S}}$ [S1].

**Formulas:**
The threshold $t_4$ is defined as:
$$ t_4 = \frac{L}{2\sqrt{2}\cos\alpha} $$
where $\alpha$ is the angle between the line normal $\mathbf{N}=(A_p, B_p)$ and the voxel diagonal vector $\mathbf{K}=(1, 1)$ [S1]. The inclusion condition for a voxel center $(x, y)$ is:
$$ -t_4 \leq A_p x + B_p y + D_p \leq t_4 $$
which is equivalent to checking if the absolute signed Euclidean distance from the center to the line is $\leq t_4$ [S1].

**Parameters:**
- $t_4$: The half-thickness of the slab [S1].
- $\alpha$: The angle between the normal vector $\mathbf{N}$ to the line and the diagonal vector $\mathbf{K}$ of the voxel [S1].
- Typical values: $L=1$ for unit grids; $t_4$ scales linearly with $L$ and varies with $\alpha$ [S1].

**Edge Cases and Pitfalls:**
- If the threshold $t_4$ is not calculated correctly based on the angle $\alpha$, pixels that should be included might be excluded, creating 4-holes [S1].
- The proof shows that if a pixel is intersected by the line but excluded, it leads to a contradiction regarding the distance from the center $C$ to the line [S1].
- Degenerate lines (zero length or undefined normal) must be handled to avoid division by zero in normalization [G].
- When $\cos\alpha \approx 0$ (line normal perpendicular to grid diagonal), $t_4$ approaches infinity; a practical guard clamps $t_4$ to a maximum reasonable value or falls back to $L/2$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_line_4_separating(Ap, Bp, Dp, L, x_min, x_max, y_min, y_max):
    """
    Voxelize a 2D line defined by Ap*x + Bp*y + Dp = 0.
    Returns a set of (x, y) coordinates.
    """
    # Ensure normal is unit length for Euclidean distance
    norm = math.hypot(Ap, Bp)
    if norm == 0:
        return set()
    An, Bn, Dn = Ap / norm, Bp / norm, Dp / norm

    # Angle alpha between normal (An, Bn) and grid diagonal (1, 1)
    # cos(alpha) = (An*1 + Bn*1) / (|N| * |K|) = (An + Bn) / sqrt(2)
    cos_alpha = (An + Bn) / math.sqrt(2)

    # Threshold t4
    if abs(cos_alpha) < 1e-9:
        t4 = L / 2.0
    else:
        t4 = L / (2.0 * math.sqrt(2) * abs(cos_alpha))

    voxel_set = set()
    for x in range(x_min, x_max + 1):
        for y in range(y_min, y_max + 1):
            val = An * x + Bn * y + Dn
            if abs(val) <= t4:
                voxel_set.add((x, y))
    return voxel_set
```

## Subtopics

### [Optimal Scanline Voxelization (Floating-Point)](10209-optimal-scanline-voxelization-floating-point.md)
Extends the 4-separating concept to fill triangle slices by dynamically calculating step sizes to skip covered voxels.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the bounding box of the line segment [G]. Each voxel requires a constant number of arithmetic operations [G].
- **Memory Complexity:** $O(K)$ where $K$ is the number of voxels in the final representation [G].
- **Strengths:** Guarantees 4-separability (no 4-holes) and 4-minimality (no redundant voxels) for 2D lines [S1].
- **Limitations:** The method is defined for 2D lines. For 3D planes, the source indicates a similar slab-based approach is used but does not provide the specific 3D threshold formula in the provided text, only referencing 6- or 26-separating representations [S1].
- **When to use:** When topological correctness (connectivity) of line primitives is critical and minimal voxel count is desired [G].

## Background
Originated in the paper "An Accurate Method for Voxelizing Polygon Meshes" [S1]. The method is part of a broader family of geometric voxelization techniques that rely on distance-based slab inclusion to ensure topological properties like separability and minimality [G].

## Sources
[S1] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
