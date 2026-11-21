# Ray-Octree Traversal (Gargantini and Atkinson variant)

Also called: Atkinson traversal, Gargantini and Atkinson Octree Ray Traversal, Gargantini and Atkinson Octree Traversal, Gargantini and Atkinson Ray-Octree Traversal, Gargantini and Atkinson variant, Gargantini traversal

## Overview
This method traverses an octree by fully sorting child octants based on the intersection order of the X, Y, and Z mid-planes, exploiting binary arithmetic on integer coordinates to determine the traversal sequence [S3, S4, S2]. Unlike its sibling [Ray Traversal of Octree Nodes](1841-ray-traversal-of-octree-nodes.md), which determines the order of mid-plane intersections to guide recursive descent, this variant explicitly sorts these intersections to march sequentially through child octants, often using neighbor-finding for non-leaf transitions or specialized routines for leaves/caps [S1][S6, unofficial]. It is distinguished by its use of rational arithmetic to maintain precision in t-value comparisons, avoiding the floating-point errors inherent in standard implementations [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Ray defined parametrically as $P(t) = B + t \cdot M$, where $B$ is the origin vector and $M$ is the direction vector [S1].
*   Octree root node $R$ [S1].
*   World space width $W$, which must be a power of 2, with the origin at $(0,0,0)$ [S1].
*   Optional: Isovalue for scalar/cap traversal [S5, unofficial].

**Outputs:**
*   Traversal of octree nodes/leaves intersected by the ray [S5, unofficial].
*   Intersection results (e.g., color, normal) if an object is found [S1].

### Data Structures

**Rational Number:**
A record type used to store t-values exactly [S1].
*   `NUM`: Integer numerator.
*   `DEN`: Integer denominator.

**Cell Record:**
Used to track the current cell during traversal [S1].
*   `T_IN`: Rational, entry t-value.
*   `T_OUT`: Rational, exit t-value.
*   `SIZ`: Integer, cell width.
*   `PTR`: Pointer to the current octree node.
*   `CORNER`: Array of 3 integers, representing the furthest corner coordinates from the origin.
*   `DIRECT`: Direction enum, indicating the direction to the next cell.

**Auxiliary Arrays:**
*   `SIGN_M`: Array of 3 integers (0 or 1), where `SIGN_M[I] = 0` if $M[I] > 0$ else $1$ [S1].
*   `axis_isects`: Array storing the sorted intersection distances with the X, Y, Z mid-planes [S5, unofficial].
*   `axis_table`: Lookup table mapping sorted axis indices to bit positions: `{4, 2, 1}` corresponding to X, Y, Z bits in the child index [S5, unofficial].

### Steps

1.  **Initialization:**
    *   Set pointer $P$ to root $R$ [S1].
    *   Create cell record $C$ [S1].
    *   Compute `SIGN_M[I]` for each axis $I \in \{X, Y, Z\}$ [S1].
    *   Compute the initial entry t-value $T_{OUT}(C)$ using `FIRST_POINT(W, DIR)` [S1]. This involves computing rational t-values for the entry planes:
        $$ \text{NUM}(T[I]) = \text{SIGN\_M}[I] \cdot W - B[I] $$
        $$ \text{DEN}(T[I]) = M[I] $$
    *   Determine the initial direction `DIR` using `NEXT_CELL_DIRECTION` based on pairwise comparisons of entry t-values [S1].
    *   Calculate the initial corner coordinates `CORNER(C)` and point `POINT` using:
        $$ \text{POINT}[I] = \left\lfloor \frac{M[I] \cdot \text{NUM}(T_{OUT}(C))}{\text{DEN}(T_{OUT}(C))} \right\rfloor + B[I] + \text{SIGN\_M}[I] \cdot \text{CHANGE}(\text{DIR}, I) $$
        where $\lfloor x \rfloor$ denotes the floor function [S1].
    *   Validate entry: If any `POINT[I]` is outside $[0, W]$, the ray never enters the space; return immediately [S1].
    *   Locate the initial cell using `FIND_3D_BLOCK(P, POINT, CORNER(C), W)` [S1].

2.  **Traversal Loop:**
    *   Loop indefinitely:
        *   Set `PTR(C) = P`, `SIZ(C) = W`, `T_IN(C) = T_OUT(C)` [S1].
        *   Compute exit t-values for each plane $I \in \{X, Y, Z\}$ using rational arithmetic:
            $$ \text{NUM}(T[I]) = \text{CORNER}(C)[I] - \text{SIGN\_M}[I] \cdot W - B[I] $$
            $$ \text{DEN}(T[I]) = M[I] $$
        *   Perform pairwise comparisons `COMPARE_T(T, CYX, CZX, CZY)` to determine the relative order of exit t-values [S1].
        *   Determine the minimum exit axis (`MIN_AXIS`) and the next cell direction `DIRECT(C)` using `NEXT_CELL_DIRECTION` [S1].
        *   Set `T_OUT(C) = T[MIN_AXIS]` [S1].
        *   **Intersection Test:** Call `RAY_INTERSECTS_OBJECT_IN_CELL(C)` [S1]. If it intersects, perform reflection/refraction or return the result [S1].
        *   **Neighbor Finding:** If no intersection, find the next cell using `OT_GTEQ_NEIGHBOR(P, DIRECT(C), P, W)` [S1].
        *   If the neighbor pointer $P$ is null, the ray has exited the scene; return [S1].
        *   If the neighbor is smaller than the current cell (`GRAY(P)` is true), compute the entry point into the neighbor and use `FIND_3D_BLOCK` to locate the specific block [S1].
        *   Update `CORNER(C)` to align with the new neighbor, handling boundary wrapping if necessary (e.g., modulo $W$) [S1].

3.  **Interior Node Traversal (Variant Optimization):**
    *   For implementations that use the "variant" approach described in [S3, S4][S5, unofficial] (often used for sparse voxel octrees or specific hashing schemes), the traversal of an interior node differs from the pure neighbor-finding approach:
        *   Retrieve the node by depth and index [S3].
        *   Compute the octree-space coordinates of the mid-planes dividing the child octants [S3].
        *   Evaluate `penter` (ray position at entry) and sort the `tcenter` intersection distances with the mid-planes into `axis_isects[]` [S3].
        *   Sequentially march across child octants using the sorted `axis_isects[]` array [S3].
        *   Compute the first child octant using constant shifting and binary-or operations [S3].
        *   Move to subsequent octants by inverting the bitmask (`axisbit`) along the corresponding mid-plane axes traversed [S3].
        *   If the child is a leaf, scalar leaf, or cap node, invoke the respective specialized traversal routine [S5, unofficial].

### Formulas

**Rational Arithmetic for t-values:**
To avoid precision errors, t-values are stored as rational numbers $t = \text{NUM} / \text{DEN}$ [S1].

**Entry Plane t-value:**
$$ \text{NUM}(T[I]) = \text{SIGN\_M}[I] \cdot W - B[I] $$
$$ \text{DEN}(T[I]) = M[I] $$

**Exit Plane t-value:**
$$ \text{NUM}(T[I]) = \text{CORNER}(C)[I] - \text{SIGN\_M}[I] \cdot W - B[I] $$
$$ \text{DEN}(T[I]) = M[I] $$

**Point Location in Cell:**
$$ \text{POINT}[I] = \left\lfloor \frac{M[I] \cdot \text{NUM}(T_{OUT}(C))}{\text{DEN}(T_{OUT}(C))} \right\rfloor + B[I] + \text{SIGN\_M}[I] \cdot \text{CHANGE}(\text{DIR}, I) $$

**Pairwise Comparison (COMPARE_T):**
$$ \text{CYX} = |\text{NUM}(T[Y]) \cdot \text{DEN}(T[X])| - |\text{NUM}(T[X]) \cdot \text{DEN}(T[Y])| $$
$$ \text{CZX} = |\text{NUM}(T[Z]) \cdot \text{DEN}(T[X])| - |\text{NUM}(T[X]) \cdot \text{DEN}(T[Z])| $$
$$ \text{CZY} = |\text{NUM}(T[Z]) \cdot \text{DEN}(T[Y])| - |\text{NUM}(T[Y]) \cdot \text{DEN}(T[Z])| $$

**CHANGE Function:**
$\text{CHANGE}(L, A)$ indicates the smallest amount, with appropriate sign, by which coordinate $A$ changes due to motion in direction $L$. For example, $\text{CHANGE}(\text{'RB'}, \text{'Z'}) = -1$ and $\text{CHANGE}(\text{'RB'}, \text{'Y'}) = 0$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Rational:
    def __init__(self, num, den):
        self.NUM = num
        self.DEN = den

class Cell:
    def __init__(self):
        self.T_IN = None
        self.T_OUT = None
        self.SIZ = 0
        self.PTR = None
        self.CORNER = [0, 0, 0]
        self.DIRECT = None
        self.POINT = [0, 0, 0]
        self.T = {'X': Rational(0, 1), 'Y': Rational(0, 1), 'Z': Rational(0, 1)}

def CHANGE(LA, A):
    # Lookup table for coordinate changes based on direction LA and axis A [S1]
    # Simplified for standard directions; full implementation requires Table 9 from [S1]
    table = {
        ('RB', 'Z'): -1, ('RB', 'Y'): 0, ('RB', 'X'): 0,
        ('RD', 'Z'): 0, ('RD', 'Y'): -1, ('RD', 'X'): 0,
        ('RF', 'Z'): 0, ('RF', 'Y'): 0, ('RF', 'X'): -1,
        ('UB', 'Z'): 0, ('UB', 'Y'): 0, ('UB', 'X'): 0,
        ('UD', 'Z'): 0, ('UD', 'Y'): 0, ('UD', 'X'): 0,
        ('UF', 'Z'): 0, ('UF', 'Y'): 0, ('UF', 'X'): 0,
        ('LB', 'Z'): 1, ('LB', 'Y'): 0, ('LB', 'X'): 0,
        ('LD', 'Z'): 0, ('LD', 'Y'): 1, ('LD', 'X'): 0,
        ('LF', 'Z'): 0, ('LF', 'Y'): 0, ('LF', 'X'): 1,
    }
    return table.get((LA, A), 0)

def FACE_DIR(axis, sign):
    # Returns direction enum based on axis and sign [S1]
    return f"FACE_{axis}_{'POS' if sign == 0 else 'NEG'}"

def EDGE_DIR(edge, sign1, sign2):
    return f"EDGE_{edge}_{'POS' if sign1 == 0 else 'NEG'}_{'POS' if sign2 == 0 else 'NEG'}"

def VERTEX_DIR(sign1, sign2, sign3):
    return f"VERTEX_{'POS' if sign1 == 0 else 'NEG'}_{'POS' if sign2 == 0 else 'NEG'}_{'POS' if sign3 == 0 else 'NEG'}"

def COMPARE_T(T):
    num_x, den_x = T['X'].NUM, T['X'].DEN
    num_y, den_y = T['Y'].NUM, T['Y'].DEN
    num_z, den_z = T['Z'].NUM, T['Z'].DEN
    CYX = abs(num_y * den_x) - abs(num_x * den_y)
    CZX = abs(num_z * den_x) - abs(num_x * den_z)
    CZY = abs(num_z * den_y) - abs(num_y * den_z)
    return CYX, CZX, CZY

def find_min_axis(CYX, CZX, CZY):
    if CZY < 0:
        return 'Z' if CZX < 0 else 'X'
    else:
        return 'Y' if CYX < 0 else 'X'

def find_max_axis(CYX, CZX, CZY):
    if CZY > 0:
        return 'Z' if CZX > 0 else 'X'
    else:
        return 'Y' if CYX > 0 else 'X'

def NEXT_CELL_DIRECTION(A, CYX, CZX, CZY, SIGN_M):
    if A == 'Z':
        return FACE_DIR('Z', SIGN_M['Z'])
    elif A == 'Y':
        return EDGE_DIR('YZ', SIGN_M['Y'], SIGN_M['Z']) if CZY == 0 else FACE_DIR('Y', SIGN_M['Y'])
    elif CZX == 0:
        if CYX == 0:
            return VERTEX_DIR(SIGN_M['X'], SIGN_M['Y'], SIGN_M['Z'])
        else:
            return EDGE_DIR('XZ', SIGN_M['X'], SIGN_M['Z'])
    elif CYX == 0:
        return EDGE_DIR('XY', SIGN_M['X'], SIGN_M['Y'])
    else:
        return FACE_DIR('X', SIGN_M['X'])

def first_point(W, SIGN_M, M, B):
    T = {'X': Rational(SIGN_M['X'] * W - B['X'], M['X']),
         'Y': Rational(SIGN_M['Y'] * W - B['Y'], M['Y']),
         'Z': Rational(SIGN_M['Z'] * W - B['Z'], M['Z'])}
    CYX, CZX, CZY = COMPARE_T(T)
    MAX_AXIS = find_max_axis(CYX, CZX, CZY)
    DIR = NEXT_CELL_DIRECTION(MAX_AXIS, CYX, CZX, CZY, SIGN_M)
    return T[MAX_AXIS], DIR

def ray_tracer(M, B, W, R):
    SIGN_M = {'X': 0 if M['X'] > 0 else 1, 'Y': 0 if M['Y'] > 0 else 1, 'Z': 0 if M['Z'] > 0 else 1}
    C = Cell()
    T_out, DIR = first_point(W, SIGN_M, M, B)
    C.T_OUT = T_out
    for I in ['X', 'Y', 'Z']:
        C.CORNER[I] = W
        C.POINT[I] = (M[I] * T_out.NUM // T_out.DEN) + B[I] + SIGN_M[I] * CHANGE(DIR, I)
    if any(C.POINT[I] < 0 or C.POINT[I] > W for I in ['X', 'Y', 'Z']):
        return
    P = R
    # FIND_3D_BLOCK(P, C.POINT, C.CORNER, W) # [S1]
    while True:
        C.PTR = P
        C.SIZ = W
        C.T_IN = C.T_OUT
        for I in ['X', 'Y', 'Z']:
            C.T[I].NUM = C.CORNER[I] - SIGN_M[I] * W - B[I]
            C.T[I].DEN = M[I]
        CYX, CZX, CZY = COMPARE_T(C.T)
        MIN_AXIS = find_min_axis(CYX, CZX, CZY)
        C.DIRECT = NEXT_CELL_DIRECTION(MIN_AXIS, CYX, CZX, CZY, SIGN_M)
        C.T_OUT = C.T[MIN_AXIS]
        # if RAY_INTERSECTS_OBJECT_IN_CELL(C): return # [S1]
        P = OT_GTEQ_NEIGHBOR(C.PTR, C.DIRECT, W) # [S1]
        if P is None:
            return
        for I in ['X', 'Y', 'Z']:
            change = CHANGE(C.DIRECT, I)
            if change == 1:
                C.CORNER[I] += W
            elif change == -1:
                C.CORNER[I] -= C.SIZ
            else:
                if C.CORNER[I] % W == 0:
                    C.CORNER[I] = 0
                else:
                    C.CORNER[I] = W - (C.CORNER[I] % W)
        if GRAY(P): # [S1]
            for I in ['X', 'Y', 'Z']:
                C.POINT[I] = (M[I] * C.T_OUT.NUM // C.T_OUT.DEN) + B[I] + SIGN_M[I] * CHANGE(C.DIRECT, I)
            # FIND_3D_BLOCK(P, C.POINT, C.CORNER, W) # [S1]

def OT_GTEQ_NEIGHBOR(P, DIRECT, W):
    # Stub for neighbor finding routine [S1]
    return P

def GRAY(P):
    # Stub for checking if node P is smaller than current cell [S1]
    return False

def FIND_3D_BLOCK(P, POINT, CORNER, W):
    # Stub for locating block [S1]
    pass

def RAY_INTERSECTS_OBJECT_IN_CELL(C):
    # Stub for intersection test [S1]
    return False
```

### Parameters
*   $W$: Width of the 3D space, must be a power of 2 [S1].
*   $M$: Direction vector of the ray [S1].
*   $B$: Origin vector of the ray [S1].
*   `axis_table`: `{4, 2, 1}` for bit manipulation in variant traversal [S5, unofficial].

### Edge Cases and Pitfalls
*   **Precision Errors:** Floating-point arithmetic can cause incorrect t-value comparisons, leading to missed intersections or infinite loops. Solved by using rational arithmetic (NUM/DEN) for all t-value calculations [S1].
*   **Rational Arithmetic Growth:** The number of bits required for rational parameters grows geometrically with secondary rays (reflections/refractions). The method is adequate only if secondary rays do not spawn further secondary rays [S1].
*   **Equal t-values:** When t-values are equal (e.g., $CYX=0$), the ray passes through an edge or vertex. `NEXT_CELL_DIRECTION` must handle these cases by checking for edge or vertex directions [S1].
*   **Neighbor Size Differences:** If the neighbor cell is smaller than the current cell, `FIND_3D_BLOCK` must be used to locate the specific block within the neighbor [S1].

## Subtopics

### [Scalar Leaf Traversal](1994-scalar-leaf-traversal.md)
Handles leaf nodes storing a single homogeneous scalar value by recursively traversing to cap depth and performing trilinear isosurface intersection.

### [Ray-Octree Traversal (Gargantini and Atkinson variant) > Cap Node Traversal](2013-ray-octree-traversal-gargantini-and-atkinson-variant-cap-nod.md)
Handles the leaf-level boundary of the octree structure where explicit child pointers are absent, determining isosurface intersections using forward-neighbor voxels.

## Operating characteristics
*   **Time Complexity:** The initial cost involves computing and sorting mid-plane intersections for interior nodes, which is moderate. Subsequent traversal of child octants is trivial, involving only bitmask inversions [S3]. Neighbor-finding for non-leaf transitions can be costly depending on tree depth and structure [S1].
*   **Memory Complexity:** Requires storage for rational t-values and cell records. Rational arithmetic may require larger word sizes, especially for secondary rays [S1].
*   **Performance:** Single-ray techniques are preferred over SIMD/packet traversal for highly-variant scenes or large data, as coherent techniques can cause unnecessary work when pixels cover multiple voxels [S3, S4].
*   **Strengths:** Exact precision with rational arithmetic avoids common floating-point errors in ray tracing. Efficient traversal of child octants using binary arithmetic [S3].
*   **Limitations:** Rational arithmetic precision grows with secondary rays, limiting its use in scenes with multiple reflections/refractions [S1]. Implementation complexity is high due to the need for neighbor-finding routines and boundary handling [S1].

## Background
The algorithm was introduced by Gargantini and Atkinson in 1982 [S1]. It was designed for ray tracing in scenes represented by octrees or quadtrees, using neighbor finding instead of point location to traverse cells [S1]. The method has been adapted and optimized in later works, such as the interactive isosurface ray tracing of large octree volumes by Wald et al. [S2, S3, S4], which exploits binary arithmetic on integer coordinates and fully sorts child octants.

## Sources
[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S3] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S4] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
[S5] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S6] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S7] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
