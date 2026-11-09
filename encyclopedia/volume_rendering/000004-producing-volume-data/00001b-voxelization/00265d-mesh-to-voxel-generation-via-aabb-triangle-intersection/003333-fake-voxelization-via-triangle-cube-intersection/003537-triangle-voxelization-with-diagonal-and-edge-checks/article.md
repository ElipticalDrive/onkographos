# Triangle Voxelization with Diagonal and Edge Checks

Also called: Line-Plane Collision Voxelization, numpy.stl Voxelization

## Overview
This method determines if a voxel is intersected by a triangle by checking if the voxel's diagonals cross the triangle's plane within the triangle bounds, and handling edge cases where the triangle lies entirely within or touches the voxel faces [S1, unofficial]. It is a specific implementation of [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) that avoids complex Constructive Solid Geometry (CSG) operations by treating each triangle independently [S1, unofficial].

## How it works
**Inputs:** A triangle defined by three vertices $p_0, p_1, p_2$ in 3D space [S1, unofficial].
**Outputs:** A list of voxel coordinates $(x, y, z)$ that are intersected by the triangle [S1, unofficial].

**Data Structures:**
*   Triangle vertices: Array of 3 vectors, each with 3 floats $(x, y, z)$ [S1, unofficial].
*   Voxel coordinates: Integers $(x, y, z)$ representing the grid index [S1, unofficial].
*   Plane normal: 3D vector $(n_x, n_y, n_z)$ [S1, unofficial].

**Steps:**
1.  **Compute Bounding Box:** Calculate the minimum and maximum coordinates of the triangle vertices [S1, unofficial]. Determine the voxel grid range to iterate over:
    $$ minx = \lfloor \min(p_{0x}, p_{1x}, p_{2x}) \rfloor, \quad maxx = \lceil \max(p_{0x}, p_{1x}, p_{2x}) \rceil $$
    Repeat for $y$ and $z$ axes [S1, unofficial].
2.  **Compute Plane Normal:** Calculate the normal vector to the triangle plane:
    $$ v_0 = p_1 - p_0 $$
    $$ v_3 = p_2 - p_0 $$
    $$ planeNormal = v_0 \times v_3 $$
    [S1, unofficial].
3.  **Iterate Voxels:** Loop through each voxel $(x, y, z)$ where $x \in [minx, maxx]$, $y \in [miny, maxy]$, $z \in [minz, maxz]$ [S1, unofficial].
4.  **Diagonal Checks:** Check if any of the 4 main diagonals of the voxel cube intersect the triangle plane inside the triangle bounds [S1, unofficial]. The diagonals connect opposite corners of the voxel [S1, unofficial]. The voxel is centered at $(x, y, z)$ with extent $0.5$ in each direction [S1, unofficial]. The corners are at $(x \pm 0.5, y \pm 0.5, z \pm 0.5)$ [S1, unofficial]. The four diagonals are:
    *   From $(x-0.5, y-0.5, z-0.5)$ to $(x+0.5, y+0.5, z+0.5)$
    *   From $(x+0.5, y+0.5, z-0.5)$ to $(x-0.5, y-0.5, z+0.5)$
    *   From $(x+0.5, y-0.5, z-0.5)$ to $(x-0.5, y+0.5, z+0.5)$
    *   From $(x-0.5, y+0.5, z-0.5)$ to $(x+0.5, y-0.5, z+0.5)$
    For each diagonal, perform a line-plane intersection test [S1, unofficial]. If the intersection point lies within the triangle, mark the voxel as intersected [S1, unofficial].
5.  **Edge Case Handling:** If diagonal checks fail, check if the triangle intersects the voxel faces or is completely inside the voxel [S1, unofficial]. This involves checking line-plane collisions on the voxel faces (x, y, z faces) [S1, unofficial]. It also involves checking if the triangle origin (e.g., $p_0$) is within the voxel bounds [S1, unofficial]. If any edge case condition is met, mark the voxel as intersected [S1, unofficial].

**Formulas:**
*   **Line-Plane Intersection Parameter:**
    $$ si = \frac{-planeNormal \cdot w}{ndotu} $$
    where $w = rayPoint - planePoint$ and $ndotu = planeNormal \cdot rayDirection$ [S1, unofficial].
    The intersection point is:
    $$ Psi = w + si \cdot rayDirection + planePoint $$
    [S1, unofficial].
*   **Point-in-Triangle Test:** A point $p$ is inside triangle $abc$ if it is on the same side of edge $bc$ as $a$, same side of $ac$ as $b$, and same side of $ab$ as $c$ [S1, unofficial]. Side check:
    $$ SameSide(p1, p2, a, b) = ((b-a) \times (p1-a)) \cdot ((b-a) \times (p2-a)) \ge 0 $$
    [S1, unofficial].

**Parameters:**
*   **Epsilon:** A small value ($10^{-6}$) used to determine if the ray direction is parallel to the plane ($|planeNormal \cdot rayDirection| < \epsilon$) [S1, unofficial].

**Pitfalls:**
*   **Degenerate/Edge Case:** If the triangle is completely inside the voxel or touches its faces, diagonal checks might miss it [S1, unofficial]. The `intersect` function handles this by checking line-plane collisions on the voxel faces (x, y, z faces) and checking if the triangle origin is within the voxel bounds [S1, unofficial].
*   **Precision:** Floating-point precision issues may arise when checking if the intersection point lies exactly on the triangle edges or voxel boundaries [S1, unofficial]. Using a small epsilon helps mitigate this [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def triangle_voxalize(triangle):
    p0 = np.array(triangle[0])
    p1 = np.array(triangle[1])
    p2 = np.array(triangle[2])
    
    v0 = p1 - p0
    v3 = p2 - p0
    planeNormal = np.cross(v0, v3)
    
    trix = [p0[0], p1[0], p2[0]]
    triy = [p0[1], p1[1], p2[1]]
    triz = [p0[2], p1[2], p2[2]]
    
    minx = int(np.floor(np.min(trix)))
    maxx = int(np.ceil(np.max(trix)))
    miny = int(np.floor(np.min(triy)))
    maxy = int(np.ceil(np.max(triy)))
    minz = int(np.floor(np.min(triz)))
    maxz = int(np.ceil(np.max(triz)))
    
    points = []
    
    for x in range(minx, maxx + 1):
        for y in range(miny, maxy + 1):
            for z in range(minz, maxz + 1):
                if LinePlaneCollision(triangle, planeNormal, p0, [1, 1, 1], [x-0.5, y-0.5, z-0.5], [x, y, z]):
                    points.append([x, y, z])
                    continue
                if LinePlaneCollision(triangle, planeNormal, p0, [-1, -1, 1], [x+0.5, y+0.5, z-0.5], [x, y, z]):
                    points.append([x, y, z])
                    continue
                if LinePlaneCollision(triangle, planeNormal, p0, [-1, 1, 1], [x+0.5, y-0.5, z-0.5], [x, y, z]):
                    points.append([x, y, z])
                    continue
                if LinePlaneCollision(triangle, planeNormal, p0, [1, -1, 1], [x-0.5, y+0.5, z-0.5], [x, y, z]):
                    points.append([x, y, z])
                    continue
                
                if intersect(triangle, [x, y, z], v0, p0):
                    points.append([x, y, z])
                    
    return points

def LinePlaneCollision(triangle, planeNormal, planePoint, rayDirection, rayPoint, point, epsilon=1e-6):
    planeNormal = np.array(planeNormal)
    planePoint = np.array(planePoint)
    rayDirection = np.array(rayDirection)
    rayPoint = np.array(rayPoint)
    point = np.array(point)
    
    ndotu = np.dot(planeNormal, rayDirection)
    if abs(ndotu) < epsilon:
        return False
    
    w = rayPoint - planePoint
    si = -np.dot(planeNormal, w) / ndotu
    Psi = w + si * rayDirection + planePoint
    
    if np.abs(Psi[0] - point[0]) <= 0.5 and np.abs(Psi[1] - point[1]) <= 0.5 and np.abs(Psi[2] - point[2]) <= 0.5:
        if PointInTriangle(Psi, triangle):
            return True
    return False

def PointInTriangle(p, triangle):
    a, b, c = triangle[0], triangle[1], triangle[2]
    
    def SameSide(p1, p2, a, b):
        cp1 = np.cross(b - a, p1 - a)
        cp2 = np.cross(b - a, p2 - a)
        return np.dot(cp1, cp2) >= 0
    
    if SameSide(p, a, b, c) and SameSide(p, b, a, c) and SameSide(p, c, a, b):
        return True
    return False

def intersect(triangle, voxel_center, vector, origin):
    x, y, z = voxel_center
    
    for xcube in range(x, x + 2):
        xcube -= 0.5
        if LinePlaneCollision(triangle, [1, 0, 0], [xcube, y, z], vector, origin, [x, y, z]):
            return True
            
    for ycube in range(y, y + 2):
        ycube -= 0.5
        if LinePlaneCollision(triangle, [0, 1, 0], [x, ycube, z], vector, origin, [x, y, z]):
            return True
            
    for zcube in range(z, z + 2):
        zcube -= 0.5
        if LinePlaneCollision(triangle, [0, 0, 1], [x, y, zcube], vector, origin, [x, y, z]):
            return True
            
    if origin[0] <= x + 0.5 and origin[0] >= x - 0.5:
        if origin[1] <= y + 0.5 and origin[1] >= y - 0.5:
            if origin[2] <= z + 0.5 and origin[2] >= z - 0.5:
                return True
    return False
```

## Operating characteristics

## Background

## Sources
[S1] In Python, how do I voxelize a 3D mesh - Stack Overflow, https://stackoverflow.com/questions/11851342/in-python-how-do-i-voxelize-a-3d-mesh, fetched 2026-10-26 (unofficial)
