# Mesh Transformation via Embree RTC
Also called: xformRtcMeshOnly
## Overview
This method applies a 4x4 transformation matrix to mesh vertices within an Embree ray tracing scene buffer while preserving the original vertex positions in the source mesh data [S1, unofficial].
## How it works
**Inputs:**
- A source mesh with vertex positions $\mathbf{v}_{orig}$ [S1, unofficial].
- A 4x4 transformation matrix $M$ [S1, unofficial].
- An Embree geometry object containing the vertex buffer [S1, unofficial].

**Outputs:**
- Updated vertex buffer in the Embree geometry [S1, unofficial].
- The source mesh remains unchanged [S1, unofficial].

**Data Structures:**
- Vertex buffer: Array of 3D floats $(x, y, z)$ [S1, unofficial].
- Transformation matrix: 4x4 float matrix consistent with Embree's expectations [S1, unofficial].

**Steps:**
1. Retrieve the vertex buffer pointer from the Embree geometry using `rtcGetGeometryBufferData` [S1, unofficial].
2. Initialize a running sum vector `averagePt` to $(0, 0, 0, 1)$ and `maxlen` to $0$ [S1, unofficial].
3. Iterate through each vertex $i$ from $0$ to $N-1$ (where $N$ is the vertex count):
   a. Reset the current vertex $\mathbf{v}_i$ in the buffer to its original position $\mathbf{v}_{orig, i}$ [S1, unofficial].
   b. Transform the vertex: $\mathbf{v}' = M \cdot \mathbf{v}_{orig, i}$ (using homogeneous coordinates with $w=1$) [S1, unofficial].
   c. Update the buffer with the transformed coordinates: $\mathbf{v}_i.x = \mathbf{v}'[0]$, $\mathbf{v}_i.y = \mathbf{v}'[1]$, $\mathbf{v}_i.z = \mathbf{v}'[2]$ [S1, unofficial].
   d. Accumulate the transformed position into the running sum: `averagePt` += $\mathbf{v}'$ [S1, unofficial].
   e. Compute the distance $d$ from $\mathbf{v}'$ to the current running centroid $\mathbf{c}_i = \text{averagePt} / \max(1, i)$ [S1, unofficial].
   f. If $d > \text{maxlen}$, update $\text{maxlen} = d$ [S1, unofficial].
4. After the loop, compute the final centroid $\mathbf{cp} = \text{averagePt} / N$ [S1, unofficial].
5. Commit the geometry and scene updates in Embree [S1, unofficial].

**Pitfalls:**
- The incremental computation of `maxlen` (distance to the running centroid) is mathematically incorrect for finding the maximum distance from the final centroid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_mesh_embree(vertices, original_positions, M, N):
    # vertices: array of objects with .x, .y, .z attributes
    # M: 4x4 matrix (list of lists)
    # N: vertex count
    average_pt = [0.0, 0.0, 0.0, 1.0]
    maxlen = 0.0

    for i in range(N):
        # Reset to original
        v_orig = original_positions[i]
        v_h = [v_orig[0], v_orig[1], v_orig[2], 1.0]

        # Transform v = M * v_h
        v_trans = [0.0, 0.0, 0.0, 0.0]
        for r in range(4):
            for c in range(4):
                v_trans[r] += M[r][c] * v_h[c]

        # Update buffer
        vertices[i].x = v_trans[0]
        vertices[i].y = v_trans[1]
        vertices[i].z = v_trans[2]

        # Accumulate
        average_pt[0] += v_trans[0]
        average_pt[1] += v_trans[1]
        average_pt[2] += v_trans[2]
        # average_pt[3] remains 1.0 per source logic

        # Compute distance to running centroid
        divisor = 1 if i == 0 else i
        centroid_x = average_pt[0] / divisor
        centroid_y = average_pt[1] / divisor
        centroid_z = average_pt[2] / divisor
        d = ((v_trans[0] - centroid_x)**2 + (v_trans[1] - centroid_y)**2 + (v_trans[2] - centroid_z)**2)**0.5

        if d > maxlen:
            maxlen = d

    # Final centroid
    cp_x = average_pt[0] / N
    cp_y = average_pt[1] / N
    cp_z = average_pt[2] / N

    # Commit geometry and scene updates in Embree
```
## Subtopics

### [Mesh Transformation](6214-mesh-transformation.md)
Applies a 4x4 transformation matrix to all vertices and normals of a mesh. It differs from its parent [Mesh Transformation via Embree RTC](5669-mesh-transformation-via-embree-rtc.md) by operating directly on the mesh data structure rather than managing Embree's internal buffer state, and from [Mesh Transformation for Printable Volume](6858-mesh-transformation-for-printable-volume.md) by lacking volume-constraint logic or axis alignment.

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
