# Octree Voxel-Triangle Intersection Test


## Overview
A method to determine if a triangle intersects a voxel within a virtual uniform grid defined by an octree, used for culling or traversal in volume rendering [S1, unofficial]. It implements the Separating Axis Theorem (SAT) to test intersection between a triangle and an axis-aligned bounding box (cube) [S1, unofficial].

## How it works
**Inputs:**
- Triangle vertices $v_0, v_1, v_2$ (3D points) [S1, unofficial].
- Voxel indices $(x, y, z)$ within the virtual uniform grid [S1, unofficial].
- Octree level $L$ [S1, unofficial].
- Virtual grid dimension $D_{grid}$ (total number of voxels along one axis at level 0) [S1, unofficial].
- Octree minimum bounds $O_{min} = (O_{minX}, O_{minY}, O_{minZ})$ [S1, unofficial].
- Triangle bounding box $T_{BB}$ with minimum $T_{BBmin}$ and maximum $T_{BBmax}$ [S1, unofficial].

**Outputs:**
- Boolean: `true` if the triangle intersects the voxel, `false` otherwise [S1, unofficial].

**Data Structures:**
- `DoublePoint`: A struct representing a 3D vector with fields `x`, `y`, `z` of type `double` [S1, unofficial].
  - Static constants: `UnitX = (1.0, 0.0, 0.0)`, `UnitY = (0.0, 1.0, 0.0)`, `UnitZ = (0.0, 0.0, 1.0)` [S1, unofficial].
  - Methods:
    - `Sub(p)`: Returns $(x-p.x, y-p.y, z-p.z)$ [S1, unofficial].
    - `Dot(p)`: Returns $x \cdot p.x + y \cdot p.y + z \cdot p.z$ [S1, unofficial].
    - `Cross(p)`: Returns $(y \cdot p.z - p.y \cdot z, z \cdot p.x - x \cdot p.z, x \cdot p.y - y \cdot p.x)$ [S1, unofficial].

**Steps:**
1. **Calculate Grid Resolution and Multiplier:**
   $$ R = 2^L $$
   $$ M = \frac{1}{R} $$
2. **Calculate Voxel Bounds:**
   $$ V_{minX} = (x \cdot D_{grid}) \cdot M + O_{minX} $$
   $$ V_{minY} = (y \cdot D_{grid}) \cdot M + O_{minY} $$
   $$ V_{minZ} = (z \cdot D_{grid}) \cdot M + O_{minZ} $$
   $$ V_{maxX} = ((x + 1) \cdot D_{grid}) \cdot M + O_{minX} $$
   $$ V_{maxY} = ((y + 1) \cdot D_{grid}) \cdot M + O_{minY} $$
   $$ V_{maxZ} = ((z + 1) \cdot D_{grid}) \cdot M + O_{minZ} $$
3. **Calculate Voxel Center and Half-Dimension:**
   $$ V_{dim} = V_{maxX} - V_{minX} $$
   $$ V_{half} = V_{dim} \cdot 0.5 $$
   $$ C_x = V_{minX} + V_{half} $$
   $$ C_y = V_{minY} + V_{half} $$
   $$ C_z = V_{minZ} + V_{half} $$
4. **Calculate Triangle Bounding Box Center:**
   $$ T_{cx} = T_{BBmin}.x + (T_{BBmax}.x - T_{BBmin}.x) \cdot 0.5 $$
   $$ T_{cy} = T_{BBmin}.y + (T_{BBmax}.y - T_{BBmin}.y) \cdot 0.5 $$
   $$ T_{cz} = T_{BBmin}.z + (T_{BBmax}.z - T_{BBmin}.z) \cdot 0.5 $$
5. **Fast Bounding Box Cull (Separating Axis on Centers):**
   Calculate distance components between centers, scaled by 2:
   $$ D_x = (C_x - T_{cx}) \cdot 2.0 $$
   $$ D_y = (C_y - T_{cy}) \cdot 2.0 $$
   $$ D_z = (C_z - T_{cz}) \cdot 2.0 $$
   Calculate triangle bounding box dimensions:
   $$ T_{dimX} = T_{BBmax}.x - T_{BBmin}.x $$
   $$ T_{dimY} = T_{BBmax}.y - T_{BBmin}.y $$
   $$ T_{dimZ} = T_{BBmax}.z - T_{BBmin}.z $$
   If any axis separates the boxes:
   $$ \text{if } D_x > V_{dim} + T_{dimX} \text{ or } D_y > V_{dim} + T_{dimY} \text{ or } D_z > V_{dim} + T_{dimZ} $$
   Return `false` [S1, unofficial].
6. **Vertex Containment Check:**
   If any triangle vertex lies within the voxel bounds:
   $$ \text{if } (V_{minX} \le v_0.x \le V_{maxX} \land V_{minY} \le v_0.y \le V_{maxY} \land V_{minZ} \le v_0.z \le V_{maxZ}) $$
   $$ \text{or } (V_{minX} \le v_1.x \le V_{maxX} \land V_{minY} \le v_1.y \le V_{maxY} \land V_{minZ} \le v_1.z \le V_{maxZ}) $$
   $$ \text{or } (V_{minX} \le v_2.x \le V_{maxX} \land V_{minY} \le v_2.y \le V_{maxY} \land V_{minZ} \le v_2.z \le V_{maxZ}) $$
   Return `true` [S1, unofficial].
7. **Detailed SAT Box-Triangle Intersection:**
   Call `isectboxtri(C_x, C_y, C_z, V_{half}, v_0, v_1, v_2)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DoublePoint:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    def Sub(self, p):
        return DoublePoint(self.x - p.x, self.y - p.y, self.z - p.z)
    def Dot(self, p):
        return self.x * p.x + self.y * p.y + self.z * p.z
    def Cross(self, p):
        return DoublePoint(
            self.y * p.z - self.z * p.y,
            self.z * p.x - self.x * p.z,
            self.x * p.y - self.y * p.x
        )

UnitX = DoublePoint(1.0, 0.0, 0.0)
UnitY = DoublePoint(0.0, 1.0, 0.0)
UnitZ = DoublePoint(0.0, 0.0, 1.0)

def octree_voxel_triangle_intersection(v0, v1, v2, x, y, z, level, gridDim, octreeMin, triBBmin, triBBmax):
    # 1. Calculate grid resolution and multiplier
    gridRes = 1 << level
    gridResMul = 1.0 / gridRes

    # 2. Calculate voxel bounds
    voxelMinX = (x * gridDim) * gridResMul + octreeMin.x
    voxelMinY = (y * gridDim) * gridResMul + octreeMin.y
    voxelMinZ = (z * gridDim) * gridResMul + octreeMin.z
    voxelMaxX = ((x + 1) * gridDim) * gridResMul + octreeMin.x
    voxelMaxY = ((y + 1) * gridDim) * gridResMul + octreeMin.y
    voxelMaxZ = ((z + 1) * gridDim) * gridResMul + octreeMin.z

    # 3. Calculate voxel center and half-dimension
    voxelDim = voxelMaxX - voxelMinX
    voxelHalfDim = voxelDim * 0.5
    centerX = voxelMinX + voxelHalfDim
    centerY = voxelMinY + voxelHalfDim
    centerZ = voxelMinZ + voxelHalfDim

    # 4. Calculate triangle bounding box center
    triBBcenterX = triBBmin.x + (triBBmax.x - triBBmin.x) * 0.5
    triBBcenterY = triBBmin.y + (triBBmax.y - triBBmin.y) * 0.5
    triBBcenterZ = triBBmin.z + (triBBmax.z - triBBmin.z) * 0.5

    # 5. Fast bounding box cull
    distX = (centerX - triBBcenterX) * 2.0
    distY = (centerY - triBBcenterY) * 2.0
    distZ = (centerZ - triBBcenterZ) * 2.0
    triDimX = triBBmax.x - triBBmin.x
    triDimY = triBBmax.y - triBBmin.y
    triDimZ = triBBmax.z - triBBmin.z
    if distX > voxelDim + triDimX or distY > voxelDim + triDimY or distZ > voxelDim + triDimZ:
        return False

    # 6. Vertex containment check
    if (voxelMinX <= v0.x <= voxelMaxX and voxelMinY <= v0.y <= voxelMaxY and voxelMinZ <= v0.z <= voxelMaxZ):
        return True
    if (voxelMinX <= v1.x <= voxelMaxX and voxelMinY <= v1.y <= voxelMaxY and voxelMinZ <= v1.z <= voxelMaxZ):
        return True
    if (voxelMinX <= v2.x <= voxelMaxX and voxelMinY <= v2.y <= voxelMaxY and voxelMinZ <= v2.z <= voxelMaxZ):
        return True

    # 7. Detailed SAT box-triangle intersection
    return isectboxtri(centerX, centerY, centerZ, voxelHalfDim, v0, v1, v2)

def isectboxtri(centerX, centerY, centerZ, boxHalfDim, v0, v1, v2):
    # Generate 8 box vertices relative to center
    boxpoints = [
        DoublePoint(centerX + boxHalfDim, centerY + boxHalfDim, centerZ + boxHalfDim),
        DoublePoint(centerX + boxHalfDim, centerY + boxHalfDim, centerZ - boxHalfDim),
        DoublePoint(centerX + boxHalfDim, centerY - boxHalfDim, centerZ + boxHalfDim),
        DoublePoint(centerX + boxHalfDim, centerY - boxHalfDim, centerZ - boxHalfDim),
        DoublePoint(centerX - boxHalfDim, centerY + boxHalfDim, centerZ + boxHalfDim),
        DoublePoint(centerX - boxHalfDim, centerY + boxHalfDim, centerZ - boxHalfDim),
        DoublePoint(centerX - boxHalfDim, centerY - boxHalfDim, centerZ + boxHalfDim),
        DoublePoint(centerX - boxHalfDim, centerY - boxHalfDim, centerZ - boxHalfDim)
    ]
    tripoints = [v0, v1, v2]

    # Test separation along X, Y, Z axes
    if not overlap(boxpoints, tripoints, UnitX): return False
    if not overlap(boxpoints, tripoints, UnitY): return False
    if not overlap(boxpoints, tripoints, UnitZ): return False

    # Calculate triangle edges and normal
    triedge1 = tripoints[1].Sub(tripoints[0])
    triedge2 = tripoints[2].Sub(tripoints[1])
    trinormal = triedge1.Cross(triedge2)
    if not overlap(boxpoints, tripoints, trinormal): return False

    triedge3 = tripoints[0].Sub(tripoints[2])

    # Test separation along 9 edge cross products
    axes = [UnitX, UnitY, UnitZ]
    edges = [triedge1, triedge2, triedge3]
    for ax in axes:
        for ed in edges:
            cross_ax_ed = ax.Cross(ed)
            if not overlap(boxpoints, tripoints, cross_ax_ed): return False

    return True

def overlap(points, tripoints, axis):
    def get_min(pts, ax):
        min_val = float('inf')
        for p in pts:
            dotprod = p.Dot(ax)
            if dotprod < min_val:
                min_val = dotprod
        return min_val
    def get_max(pts, ax):
        max_val = float('-inf')
        for p in pts:
            dotprod = p.Dot(ax)
            if dotprod > max_val:
                max_val = dotprod
        return max_val

    min_P = get_min(points, axis)
    max_P = get_max(points, axis)
    min_T = get_min(tripoints, axis)
    max_T = get_max(tripoints, axis)

    if min_P > max_T or max_P < min_T:
        return False
    return True
```

**Parameters:**
- `nTHREADS_TRAV_KERNEL`: 128 [S1, unofficial].
- `nTHREADS_DRAW_KERNEL`: 128 [S1, unofficial].
- `nTHREADS_DRAW_SHADOW_KERNEL`: 192 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The method uses `double` for coordinate calculations to maintain precision during grid resolution scaling [S1, unofficial].
- **Boundary Conditions:** The vertex containment check uses inclusive bounds ($\le$) [S1, unofficial].
- Not given in the sources for other edge cases.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] denniskb/asvo, https://github.com/denniskb/asvo, fetched 2026-10-26 (unofficial)
