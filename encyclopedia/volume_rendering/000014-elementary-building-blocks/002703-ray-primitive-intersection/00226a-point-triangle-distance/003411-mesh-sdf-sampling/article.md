# Mesh SDF Sampling

Also called: Signed Distance Field Sampler from Geometry, createGeometrySdfSampler

## Overview
Mesh SDF Sampling creates a signed distance sampler function from a `THREE.BufferGeometry` and a world transform matrix, returning a function that computes the shortest signed distance to the mesh at any given world-space coordinate [S1, unofficial]. It solves the problem of evaluating continuous signed distance fields (SDFs) for arbitrary triangle meshes without precomputing a voxel grid, enabling ray-marching or ray-casting against complex geometry in real-time volume rendering contexts. It differs from [Unsigned Triangle Distance](3587-unsigned-triangle-distance.md) by providing signed values (inside/outside) rather than only the magnitude of the distance, and from [Triangle-Box Intersection Voxelization](7408-triangle-box-intersection-voxelization.md) by operating in continuous world space rather than discrete voxel space.

## How it works
**Inputs:**
- `geometry`: A `THREE.BufferGeometry` instance containing position attributes and an index array.
- `transformMatrix`: A `THREE.Matrix4` representing the world transform of the geometry.

**Outputs:**
- A function `sampler(x, y, z)` returning a `number`: the signed distance. Negative values indicate the point is inside the mesh; positive values indicate it is outside [S1, unofficial].

**Data Structures:**
- `worldPositions`: An array of `Vector3` objects, precomputed from the geometry's position attributes transformed by `transformMatrix`. Size: `geometry.attributes.position.count`.
- Temporary vectors (allocated once to avoid garbage collection): `va`, `vb`, `vc` (triangle vertices), `ab`, `ac`, `bc` (edge vectors), `ap`, `bp`, `cp` (vectors from vertices to point `p`), `n` (triangle normal), `p` (query point), `q` (closest point on triangle), `bestPoint` (closest point found so far), `bestNormal` (normal at `bestPoint`), `toPoint` (vector from `bestPoint` to `p`).

**Algorithm Steps:**
1.  **Validation:** Check if `geometry.index` exists. If falsy, throw an error "Geometry must be indexed" [S1, unofficial].
2.  **Precomputation:** Iterate `i` from 0 to `position.count`. For each `i`, create a `Vector3` from `position` buffer, apply `transformMatrix`, and store in `worldPositions[i]` [S1, unofficial].
3.  **Sampler Function Definition:** Return a function `sampler(x, y, z)`:
    a. Set `p = (x, y, z)`. Initialize `closestDistance2 = Infinity` [S1, unofficial].
    b. Iterate through triangles using `index` array with step 3. For each triangle `i`:
        i. Get indices `idx0 = index[i]`, `idx1 = index[i+1]`, `idx2 = index[i+2]`.
        ii. Retrieve vertices `va = worldPositions[idx0]`, `vb = worldPositions[idx1]`, `vc = worldPositions[idx2]` [S1, unofficial].
        iii. Compute `q = closestPointOnTriangle(p, va, vb, vc)` using the algorithm from [Point-Triangle Distance](2776-point-triangle-distance.md) [S1, unofficial].
        iv. Compute `distance2 = p.distanceToSquared(q)` [S1, unofficial].
        v. If `distance2 < closestDistance2`:
            - Update `closestDistance2 = distance2`.
            - Copy `q` to `bestPoint`.
            - Compute edge vectors `ab = vb - va`, `ac = vc - va`.
            - Compute `bestNormal = normalize(ab × ac)` [S1, unofficial].
4.  **Sign Determination:** After the loop, compute `toPoint = p - bestPoint`. Calculate `dot = toPoint · bestNormal`.
5.  **Return:** Return `sqrt(closestDistance2) * sign`, where `sign = 1` if `dot >= 0`, else `-1` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def createGeometrySdfSampler(geometry, transformMatrix):
    if not geometry.index:
        raise Error("Geometry must be indexed")

    position = geometry.attributes.position
    index = geometry.index.array
    vertexCount = position.count

    # Precompute world-space vertices
    worldPositions = []
    for i in range(vertexCount):
        x = position.getX(i)
        y = position.getY(i)
        z = position.getZ(i)
        # Apply transform matrix to vector (x, y, z, 1)
        # Assuming transformMatrix is a 4x4 matrix stored as array or object
        # Standard matrix-vector multiplication:
        wx = transformMatrix.elements[0]*x + transformMatrix.elements[4]*y + transformMatrix.elements[8]*z + transformMatrix.elements[12]
        wy = transformMatrix.elements[1]*x + transformMatrix.elements[5]*y + transformMatrix.elements[9]*z + transformMatrix.elements[13]
        wz = transformMatrix.elements[2]*x + transformMatrix.elements[6]*y + transformMatrix.elements[10]*z + transformMatrix.elements[14]
        worldPositions.append(Vector3(wx, wy, wz))

    # Temporary vectors for reuse
    va = Vector3()
    vb = Vector3()
    vc = Vector3()
    ab = Vector3()
    ac = Vector3()
    toPoint = Vector3()
    bestPoint = Vector3()
    bestNormal = Vector3()
    q = Vector3()

    def closestPointOnTriangle(p, a, b, c, target):
        # Implementation from Point-Triangle Distance article
        # Returns closest point on triangle abc to p in target
        pass # See [2776-point-triangle-distance.md]

    def sampler(x, y, z):
        p = Vector3(x, y, z)
        closestDistance2 = float('inf')

        numTriangles = len(index) // 3
        for i in range(numTriangles):
            idx0 = index[i * 3]
            idx1 = index[i * 3 + 1]
            idx2 = index[i * 3 + 2]

            va.copy(worldPositions[idx0])
            vb.copy(worldPositions[idx1])
            vc.copy(worldPositions[idx2])

            closestPointOnTriangle(p, va, vb, vc, q)

            distance2 = p.distanceToSquared(q)

            if distance2 < closestDistance2:
                closestDistance2 = distance2
                bestPoint.copy(q)
                ab.subVectors(vb, va)
                ac.subVectors(vc, va)
                bestNormal.crossVectors(ab, ac).normalize()

        toPoint.subVectors(p, bestPoint)
        dot = toPoint.dot(bestNormal)
        sign = 1.0 if dot >= 0.0 else -1.0

        return math.sqrt(closestDistance2) * sign

    return sampler
```

**Parameters:**
- None explicitly defined in the source. The method relies on the geometry's intrinsic properties and the provided transform matrix [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-indexed Geometry:** The method requires an index array. If `geometry.index` is falsy, it throws an error [S1, unofficial].
- **Precision:** Uses squared distances for comparison to avoid square root operations during the loop, improving performance and stability [S1, unofficial].
- **Degenerate Triangles:** The `closestPointOnTriangle` function (from [Point-Triangle Distance](2776-point-triangle-distance.md)) handles degenerate cases (collinear vertices, zero-area triangles) via barycentric coordinate logic [S1, unofficial].
- **Normal Direction:** The sign depends on the winding order of the triangle vertices. If the mesh has inconsistent winding, the inside/outside distinction may be unreliable for some regions [S1, unofficial].

## Subtopics

### Signed Distance Field Sampler from Mesh
Creates a signed distance sampler function from a `THREE.Mesh` instance [S1, unofficial]. It validates that the input is a `THREE.Mesh`, updates its world matrix via `instance.updateMatrixWorld(true)`, and delegates to `createGeometrySdfSampler` using `instance.geometry` and `instance.matrixWorld` [S1, unofficial]. This variant simplifies usage for scene graph objects that already maintain world transforms.

## Operating characteristics
- **Time Complexity:** $O(N)$ per query, where $N$ is the number of triangles in the mesh. Each query iterates through all triangles to find the closest one [S1, unofficial].
- **Memory Complexity:** $O(V)$, where $V$ is the number of vertices, for storing `worldPositions` [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades linearly with mesh complexity. For large meshes, this method is impractical without spatial acceleration structures (e.g., BVH, which are not implemented here) [S1, unofficial].
- **Strengths:** No preprocessing required; works with any indexed triangle mesh; provides exact signed distances [S1, unofficial].
- **Limitations:** Slow for high-poly meshes due to linear scan; requires indexed geometry; sensitive to triangle winding for sign correctness [S1, unofficial].
- **When to Use:** Small to medium-sized meshes where preprocessing is undesirable or geometry is dynamic [S1, unofficial].
- **When Not to Use:** Large static meshes where voxelization or BVH-based acceleration is feasible [S1, unofficial].

## Background
Originated in the `three.js-volume-renderer` project by Donitzo, an unofficial implementation for volume rendering in JavaScript using Three.js [S1, unofficial]. It leverages Christer Ericson's "Real-Time Collision Detection" algorithm for point-triangle distance computation [S1, unofficial].

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
