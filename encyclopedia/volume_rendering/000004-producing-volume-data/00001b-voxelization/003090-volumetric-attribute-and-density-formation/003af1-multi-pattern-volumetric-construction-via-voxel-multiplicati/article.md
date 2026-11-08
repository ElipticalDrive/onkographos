# Multi-pattern Volumetric Construction via Voxel Multiplication

Also called: Multi-pattern volumetric volume construction via voxel multiplication, Multiplicative voxel encoding, Voxel value determination by pattern intersection

## Overview
This method constructs a 3D voxel grid that projects multiple distinct 2D patterns along different axes by calculating voxel values as the product of the pixel values from those patterns at their respective projection intersections [S1]. It differs from additive accumulation methods (e.g., [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md)) by using multiplicative encoding, which allows independent pattern recognition along orthogonal views but introduces background noise interference [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ 2D patterns (images), where each pattern $P_i$ is defined on a 2D grid with coordinates $(u_i, v_i)$ and pixel values. For full-color support, each pixel value is a vector $\mathbf{p} = [R, G, B]^T$. The patterns are associated with specific projection axes (e.g., Pattern A along the x-axis, Pattern B along the y-axis, Pattern C along the z-axis).
*   **Outputs:** A 3D voxel grid $V(x, y, z)$, where each voxel value is a vector representing the amount of ink or density at that position.

**Data Structures**
*   **Voxel Grid:** A 3D array of dimensions $X \times Y \times Z$. Each element $V(x, y, z)$ stores a color vector $\mathbf{V} = [V_R, V_G, V_B]$.
*   **Pattern Data:** A list of 2D arrays, one for each pattern. Each array has dimensions $U_i \times V_i$ (width $\times$ height) and stores pixel values $\mathbf{p}_i(u, v)$.

**Algorithm Steps**
1.  **Initialize Grid:** Create a 3D voxel grid $V(x, y, z)$ of the desired resolution. Initialize all voxel values to zero or a neutral background value (e.g., black $[0,0,0]$ for additive ink models, or $[1,1,1]$ for transmissive models, depending on the specific implementation of "product" semantics; the source implies multiplication of values where non-zero indicates presence).
2.  **Map Projection Axes:** For each pattern $P_i$ (e.g., A, B, C), define its projection axis.
    *   If Pattern A is projected along the X-axis, its 2D coordinates $(u_A, v_A)$ map to the Y and Z dimensions of the voxel grid. Specifically, $u_A$ corresponds to $y$ and $v_A$ corresponds to $z$ (or vice versa, depending on convention).
    *   If Pattern B is projected along the Y-axis, its coordinates $(u_B, v_B)$ map to X and Z.
    *   If Pattern C is projected along the Z-axis, its coordinates $(u_C, v_C)$ map to X and Y.
3.  **Calculate Voxel Values:** For each voxel $(x, y, z)$ in the grid:
    *   Identify the intersection point on Pattern A. Let the coordinates on Pattern A be $(u_A, v_A)$ corresponding to the voxel's position relative to the projection axis.
    *   Identify the intersection point on Pattern B. Let the coordinates be $(u_B, v_B)$.
    *   Identify the intersection point on Pattern C. Let the coordinates be $(u_C, v_C)$.
    *   Retrieve the pixel values $\mathbf{a}(u_A, v_A)$, $\mathbf{b}(u_B, v_B)$, and $\mathbf{c}(u_C, v_C)$ from the respective pattern images.
    *   Compute the voxel value $\mathbf{V}(x, y, z)$ as the element-wise product of these pixel values:
        $$ \mathbf{V}(x, y, z) = \mathbf{a}(u_A, v_A) \odot \mathbf{b}(u_B, v_B) \odot \mathbf{c}(u_C, v_C) $$
        Where $\odot$ denotes element-wise multiplication. For RGB components:
        $$ V_R(x,y,z) = a_R \cdot b_R \cdot c_R $$
        $$ V_G(x,y,z) = a_G \cdot b_G \cdot c_G $$
        $$ V_B(x,y,z) = a_B \cdot b_B \cdot c_B $$
4.  **Repeat for Arbitrary N:** Extend the logic to $N$ patterns by multiplying the pixel values from all $N$ patterns at their respective intersections.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_pattern_voxel_construction(patterns, grid_dims):
    """
    patterns: List of 2D arrays (images). Each pattern is associated with a projection axis.
              Assumption: patterns[0] -> X-axis projection (maps to Y,Z),
                          patterns[1] -> Y-axis projection (maps to X,Z),
                          patterns[2] -> Z-axis projection (maps to X,Y).
    grid_dims: Tuple (X, Y, Z) defining the voxel grid resolution.
    """
    X, Y, Z = grid_dims
    # Initialize voxel grid with zeros (or 1.0 if using transmissive/normalized model)
    # Using 0.0 for additive ink model where product of 0 is 0 (black/absent)
    voxel_grid = [[[0.0, 0.0, 0.0] for _ in range(Z)] for _ in range(Y)]
    for y in range(Y):
        for x in range(X):
            voxel_grid[y][x] = [[0.0, 0.0, 0.0] for _ in range(Z)]

    # Map patterns to axes
    # Pattern A (index 0) projects along X. Its 2D coords (u,v) map to (y,z).
    # Pattern B (index 1) projects along Y. Its 2D coords (u,v) map to (x,z).
    # Pattern C (index 2) projects along Z. Its 2D coords (u,v) map to (x,y).
    
    pat_A = patterns[0] # Shape (Y, Z) or (Z, Y) depending on convention. 
                        # Assuming u->y, v->z based on standard projection logic.
    pat_B = patterns[1] # Shape (X, Z)
    pat_C = patterns[2] # Shape (X, Y)

    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                # Get pixel values from each pattern at the intersection
                # Pattern A: intersection at (y, z)
                val_A = pat_A[y][z] # Returns [R, G, B]
                
                # Pattern B: intersection at (x, z)
                val_B = pat_B[x][z] # Returns [R, G, B]
                
                # Pattern C: intersection at (x, y)
                val_C = pat_C[x][y] # Returns [R, G, B]
                
                # Calculate voxel value as product
                # Element-wise multiplication for RGB
                voxel_grid[y][x][z][0] = val_A[0] * val_B[0] * val_C[0]
                voxel_grid[y][x][z][1] = val_A[1] * val_B[1] * val_C[1]
                voxel_grid[y][x][z][2] = val_A[2] * val_B[2] * val_C[2]

    return voxel_grid
```

**Parameters and Typical Values**
*   **Number of Patterns:** The source example uses three patterns (A, B, C) [S1]. The algorithm supports an arbitrary number $N$, but there is a trade-off between the number of patterns and image quality [S1].
*   **Voxel Value Representation:** For full-color, voxel values comprise three components: $V_R(x,y,z)$, $V_G(x,y,z)$, and $V_B(x,y,z)$ [S1].
*   **Pattern Resolution:** The resolution of the 2D patterns determines the effective resolution of the 3D structure along the corresponding axes.

**Edge Cases and Pitfalls**
*   **Background Noise:** The projected pattern along any axis contains interference from other patterns. Mathematically, the projected pattern $a'(u_A, v_A)$ is the original pattern $a(u_A, v_A)$ multiplied by a background noise term $\sum_{P_A} b(u_B, v_B) \cdot c(u_C, v_C)$ [S1]. This noise can degrade image quality.
*   **Image Deterioration:** In physical implementations, projected patterns may deteriorate due to contrast degradation (absorption by films) and blurring (diffraction/refraction) [S1].
*   **Dominance:** The component of the original pattern tends to be dominant over the background noise when 2D images are used, allowing recognition of the original patterns [S1].
*   **Noise Suppression:** Background noise can be suppressed by introducing an iteration algorithm or other optimization methods, though this is not part of the basic multiplicative construction [S1].

## Operating characteristics
*   **Time Complexity:** $O(X \cdot Y \cdot Z \cdot N)$, where $X,Y,Z$ are grid dimensions and $N$ is the number of patterns. Each voxel requires $N$ lookups and $N-1$ multiplications.
*   **Memory Complexity:** $O(X \cdot Y \cdot Z)$ for the voxel grid.
*   **Strengths:** Allows independent encoding of multiple views/patterns in a single 3D volume. Simple to implement.
*   **Limitations:** Multiplicative nature leads to background noise in projections. Contrast and resolution are limited by the physical properties of the medium (e.g., film absorption, diffraction) in physical displays [S1].
*   **When to Use:** When the goal is to create a 3D object that displays specific 2D images when viewed from specific orthogonal directions (e.g., volumetric displays, holographic printing).
*   **When Not to Use:** When high-fidelity, noise-free projections from all angles are required without post-processing or iterative optimization.

## Background
This method was proposed for inkjet printing-based volumetric displays to project multiple full-colour 2D patterns [S1]. It builds on a previous study [13] and extends it to full-color representation. The method is designed to create cross-sectional images of a 3D structure that exhibits multiple 2D patterns by calculating voxel values based on the intersection of projection axes [S1].

## Sources
[S1] Inkjet printing-based volumetric display projecting multiple full-colour 2D patterns., https://doi.org/10.1038/srep46511, fetched 2026-10-26
