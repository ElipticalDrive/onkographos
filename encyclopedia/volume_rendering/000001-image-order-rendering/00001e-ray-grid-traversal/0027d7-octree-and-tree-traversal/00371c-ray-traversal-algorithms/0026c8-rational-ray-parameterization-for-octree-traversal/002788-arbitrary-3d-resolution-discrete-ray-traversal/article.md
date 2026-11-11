# Arbitrary 3D Resolution Discrete Ray Traversal

Also called: Discrete Ray Tracing of Implicit Surfaces, Interval Arithmetic Ray Traversal, Recursive Octree Traversal with BSP Ordering, Robust Octree Ray Traversal

## Overview
Arbitrary 3D Resolution Discrete Ray Traversal is a method for tracing rays through implicit surfaces by voxelizing both the scene and the rays using interval arithmetic within a recursive octree subdivision [S1]. It eliminates explicit intersection calculations by ensuring the discrete ray matches the continuous ray down to the resolution of floating-point numbers, guaranteeing robustness against rounding errors [S1]. Unlike sibling methods that calculate explicit intersections or rely on pre-computed structures, this method performs on-the-fly subdivision and uses BSP ordering to determine traversal sequence without geometric intersection tests [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Ray defined by origin $(x_0, y_0, z_0)$ and direction $(dx, dy, dz)$; an octree representing the voxelized implicit surface up to a maximum level $NL$; the implicit function $f(x,y,z)$ defining the surface [S1].
*   **Output:** Boolean indicating if an intersection was found; the intersection point $(x_m, y_m, z_m)$ if found [S1].

**Data Structures**
*   **Octant:** Defined by three intervals $[x_c, X_c]$, $[y_c, Y_c]$, $[z_c, Z_c]$, where $(x_c, y_c, z_c)$ is the vertex with lowest coordinates and $(X_c, Y_c, Z_c)$ is the vertex with highest coordinates [S1].
*   **Sub-octant Index:** An integer from 0 to 7 [S1]. Each bit corresponds to an axis (x, y, z) [S1]. A bit is 1 if the sub-octant lies on the positive side of the splitting plane, 0 otherwise [S1].

**Parameters**
*   $NL$: Maximum subdivision level (leaf node level) [S1]. Default value in code is 30 [S1]. For IEEE 754 double precision, the maximum is 53 [S1].

**Formulas**
*   **Ray Equation:**
    $$x = x_0 + t \cdot dx, \quad y = y_0 + t \cdot dy, \quad z = z_0 + t \cdot dz \quad (1)$$ [S1]
*   **Splitting Planes:** For an octant of size $l$ with lowest vertex $(x_c, y_c, z_c)$, the middle point is $(x_m, y_m, z_m) = (x_c + l/2, y_c + l/2, z_c + l/2)$ [S1].
*   **Implicit Ray Coefficients:**
    $$cx = sign(dx) \cdot |dy| \cdot |dz|$$
    $$cy = sign(dy) \cdot |dz| \cdot |dx|$$
    $$cz = sign(dz) \cdot |dx| \cdot |dy| \quad (3)$$ [S1]
*   **Ray Inclusion Test:** The ray passes through an octant if the intervals derived from the implicit ray equation overlap:
    $$\begin{bmatrix} f_x(x_c) & f_x(X_c) \\ f_y(y_c) & f_y(Y_c) \\ f_z(z_c) & f_z(Z_c) \end{bmatrix} = \begin{bmatrix} cx \cdot (x_c - x_0) & cx \cdot (X_c - x_0) \\ cy \cdot (y_c - y_0) & cy \cdot (Y_c - y_0) \\ cz \cdot (z_c - z_0) & cz \cdot (z_c - z_0) \end{bmatrix} \quad (4)$$ [S1]
    Overlap is defined such that if a lower bound equals an upper bound, there is no overlap [S1].

**Algorithm Steps**
1.  **Calculate Traversal Direction Sign:** Compute `signDir` as a 3-bit integer: `signDir = (dz<0)*2 + (dy<0)*1 + (dx<0)` [S1].
2.  **Base Case Check:** If `intersectionFound` is true, return true [S1]. If `level == NL`, check if the octant contains a surface using `IsoValueTest` [S1]. If false, apply `PartialDifferentialTest` [S1]. If both fail, return false [S1]. If surface is confirmed, set `intersectionFound = true`, `intersectedPoint = mid`, and return true [S1].
3.  **Octant Containment Check:** Test if the ray passes through the current octant using the ray voxelization algorithm (Eq. 4) [S1]. If not, return false [S1].
4.  **Calculate Traversal Order Mask:** Determine the relative position of the ray start point $(x_0, y_0, z_0)$ to the octant middle point $(x_m, y_m, z_m)$ [S1]. Compute `izyx = ((z0 > zm) << 2) + ((y0 > ym) << 1) + (x0 > xm)` [S1]. Compute `iaux = izyx XOR signDir` [S1].
5.  **Iterate Through Sub-octants:** Loop $i$ from 0 to 7 [S1].
    *   If `(iaux AND i)` is non-zero, skip this sub-octant (Octant elimination) [S1].
    *   Calculate the index for the sub-octant in BSP order: `idx = izyx XOR i` [S1].
    *   Retrieve the sub-octant: `suboct = octant[idx]` [S1].
    *   Check if the ray passes through `suboct` [S1]. If not, continue [S1].
    *   If `suboct` contains a part of a surface, recursively call `Traversal(suboct, level+1)` [S1]. If true, break the loop and return true [S1].
6.  **Return:** If the loop completes without finding an intersection, return `intersectionFound` [S1].

**Optimized Interval Calculation**
To avoid branching and frequent rounding mode changes, bounds are calculated using two FPUs (e.g., SSE2 and x87) with independent rounding modes (towards $-\infty$ for lower bounds, $+\infty$ for upper bounds) [S1]. For the x-axis, an array `tx[4]` stores bounds [S1]. The selection of rounding modes depends on `dx` [S1]. For $dx > 0$, intervals are $[f_x^-(x_c), f_x^+(X_c)]$, $[f_x^-(x_c), f_x^+(x_m)]$, $[f_x^-(x_m), f_x^+(X_c)]$ [S1]. For $dx < 0$, intervals are $[f_x^-(X_c), f_x^+(x_c)]$, $[f_x^-(X_c), f_x^+(x_m)]$, $[f_x^-(x_m), f_x^+(x_c)]$ [S1].

**Partial Differential Test**
Applied at leaf nodes to reduce overestimations [S1]. It calculates inclusion functions for partial derivatives $DX = DFX(X)$, $DY = DFY(Y)$, $DZ = DFZ(Z)$ [S1]. If $0 \notin DX$ AND $0 \notin DY$ AND $0 \notin DZ$, the function is monotonic and has no zeroes, so the octant is ignored [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fx_minus(val):
    """Evaluate implicit function at val with rounding towards -inf."""
    return math.floor(f(val))  # [G] Standard interval lower bound rounding

def fx_plus(val):
    """Evaluate implicit function at val with rounding towards +inf."""
    return math.ceil(f(val))   # [G] Standard interval upper bound rounding

def intervals_overlap(int1, int2, int3):
    """Check if three intervals overlap. Equality is no overlap."""
    lower = max(int1[0], int2[0], int3[0])
    upper = min(int1[1], int2[1], int3[1])
    return lower < upper  # [S1] Equality considered no overlap

def IsoValueTest(octant):
    """Check for sign changes at the 8 corners of the octant."""
    corners = octant.get_corners()  # [G] Standard corner enumeration
    signs = [math.copysign(1, f(c)) for c in corners]
    return any(s1 != s2 for s1, s2 in zip(signs, signs[1:]))  # [G] Sign change detection

def PartialDifferentialTest(octant):
    """Check monotonicity via inclusion functions of partial derivatives."""
    X, Y, Z = octant.intervals
    DX = DFX(X)  # [G] Inclusion function for ∂f/∂x
    DY = DFY(Y)  # [G] Inclusion function for ∂f/∂y
    DZ = DFZ(Z)  # [G] Inclusion function for ∂f/∂z
    # Returns true if 0 is NOT in DX, DY, and DZ (monotonic, no zero)
    return (0 not in DX) and (0 not in DY) and (0 not in DZ)  # [S1]

def RayPassesThroughOctant(octant, x0, y0, z0, dx, dy, dz, signDir):
    """Test if ray passes through octant using interval arithmetic."""
    cx = (1 if dx >= 0 else -1) * abs(dy) * abs(dz)
    cy = (1 if dy >= 0 else -1) * abs(dz) * abs(dx)
    cz = (1 if dz >= 0 else -1) * abs(dx) * abs(dy)
    
    xc, Xc = octant.x_min, octant.x_max
    xm = octant.x_mid
    yc, Yc = octant.y_min, octant.y_max
    ym = octant.y_mid
    zc, Zc = octant.z_min, octant.z_max
    zm = octant.z_mid
    
    # Optimized interval bounds selection based on dx
    if dx > 0:
        int1 = (cx * fx_minus(xc - x0), cx * fx_plus(Xc - x0))
        int2 = (cx * fx_minus(xc - x0), cx * fx_plus(xm - x0))
        int3 = (cx * fx_minus(xm - x0), cx * fx_plus(Xc - x0))
    else:
        int1 = (cx * fx_minus(Xc - x0), cx * fx_plus(xc - x0))
        int2 = (cx * fx_minus(Xc - x0), cx * fx_plus(xm - x0))
        int3 = (cx * fx_minus(xm - x0), cx * fx_plus(xc - x0))
        
    # Repeat for y and z axes with corresponding intervals
    if dy > 0:
        int4 = (cy * fx_minus(yc - y0), cy * fx_plus(Yc - y0))
        int5 = (cy * fx_minus(yc - y0), cy * fx_plus(ym - y0))
        int6 = (cy * fx_minus(ym - y0), cy * fx_plus(Yc - y0))
    else:
        int4 = (cy * fx_minus(Yc - y0), cy * fx_plus(yc - y0))
        int5 = (cy * fx_minus(Yc - y0), cy * fx_plus(ym - y0))
        int6 = (cy * fx_minus(ym - y0), cy * fx_plus(yc - y0))
        
    if dz > 0:
        int7 = (cz * fx_minus(zc - z0), cz * fx_plus(Zc - z0))
        int8 = (cz * fx_minus(zc - z0), cz * fx_plus(zm - z0))
        int9 = (cz * fx_minus(zm - z0), cz * fx_plus(Zc - z0))
    else:
        int7 = (cz * fx_minus(Zc - z0), cz * fx_plus(zc - z0))
        int8 = (cz * fx_minus(Zc - z0), cz * fx_plus(zm - z0))
        int9 = (cz * fx_minus(zm - z0), cz * fx_plus(zc - z0))
        
    return intervals_overlap(int1, int4, int7)

def Traversal(octant, level, x0, y0, z0, dx, dy, dz, signDir, NL, found_state, point_state):
    # Base case: intersection already found
    if found_state[0]:
        return True
    
    # Base case: leaf node
    if level == NL:
        if not IsoValueTest(octant):
            if not PartialDifferentialTest(octant):
                return False
        found_state[0] = True
        point_state[0] = octant.mid
        return True

    # Check if ray passes through current octant
    if not RayPassesThroughOctant(octant, x0, y0, z0, dx, dy, dz, signDir):
        return False

    # Calculate traversal order mask
    xm, ym, zm = octant.mid
    izyx = ((z0 > zm) << 2) + ((y0 > ym) << 1) + (x0 > xm)
    iaux = izyx ^ signDir

    # Iterate through sub-octants in BSP order
    for i in range(8):
        # Octant elimination
        if iaux & i:
            continue
        
        # BSP ordering index
        idx = izyx ^ i
        suboct = octant[idx]
        
        # Check if ray passes through sub-octant
        if not RayPassesThroughOctant(suboct, x0, y0, z0, dx, dy, dz, signDir):
            continue
            
        # Check if sub-octant contains surface part
        if suboct.contains_surface():
            if Traversal(suboct, level + 1, x0, y0, z0, dx, dy, dz, signDir, NL, found_state, point_state):
                return True

    return found_state[0]
```

**Edge Cases and Pitfalls**
*   **Rounding Errors in Midpoint:** The midpoint calculation $(x_c + l/2)$ must be exact [S1]. This is guaranteed if the half-length of the octant is not smaller than 1 ULP [S1].
*   **Partial Differential Test False Negatives:** In very special cases, the test may return false (indicating non-monotonicity) even if the octant does not contain a surface part [S1].
*   **Precision:** The intersection point is $(x_m, y_m, z_m)$ [S1]. The maximum distance to the real intersection is $\sqrt{3} \cdot 2^{-NL-1}$ [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources [S1]. The method involves recursive traversal down to level $NL$ [S1].
*   **Memory Complexity:** Not explicitly given in the sources [S1]. The octree is stored, but subdivision is done on-the-fly [S1].
*   **Measured Performance:** Slightly slower than Gargantini's algorithm when using robust interval arithmetic with different rounding modes, but faster otherwise [S1]. Gargantini's algorithm eliminates all non-traversed sub-octants, while this method uses BSP ordering and ray voxelization which only partially eliminate them [S1].
*   **Strengths:** Robustness against rounding errors; eliminates explicit intersection calculations; guarantees that the discrete ray matches the continuous ray [S1].
*   **Limitations:** Slightly slower than Gargantini's algorithm in non-robust mode; Partial Differential Test may have false negatives in special cases [S1].
*   **When to Use:** When robustness is critical and explicit intersection calculations are undesirable or prone to errors [S1]. Suitable for implicit surfaces where interval arithmetic is applicable [S1].
*   **When Not to Use:** When performance is the primary concern and Gargantini's algorithm is acceptable; when the scene is not represented by implicit functions suitable for interval arithmetic [S1].

## Background
*   **Origin:** Proposed by Nilo Stolte in the paper "Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces" [S1].
*   **History:** The method is the first complete solution of an interval arithmetic-based ray tracing with 3D space subdivision using discrete ray traversal [S1]. It builds upon previous work using interval arithmetic for ray casting (e.g., Duff) and Lipschitz conditions for voxelization (e.g., Kalra and Barr), but extends it to 3D space subdivision without explicit intersections [S1].
*   **Usage:** Used for ray tracing implicit surfaces with guaranteed robustness [S1].

## Sources
[S1] Arbitrary 3D Resolution Discrete Ray Tracing of Implicit Surfaces, https://doi.org/10.1007/978-3-540-31965-8_38, fetched 2026-10-24
