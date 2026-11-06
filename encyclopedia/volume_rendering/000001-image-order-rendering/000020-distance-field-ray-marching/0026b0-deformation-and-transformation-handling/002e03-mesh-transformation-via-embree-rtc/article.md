# Mesh Transformation via Embree RTC
Also called: xformRtcMeshOnly
## Overview
This method applies a 4x4 transformation matrix to mesh vertices within an Embree ray tracing scene buffer while preserving the original vertex positions in the source mesh data [S1, unofficial]. It solves the problem of updating geometry in a hardware-accelerated ray tracing context without permanently altering the source asset, allowing for reversible or temporary transformations. It differs from [Inverse Model Matrix Transformation for Distance Fields](1962-inverse-model-matrix-transformation-for-distance-fields.md) by operating on explicit mesh vertex buffers rather than implicit distance fields, and from [Motion-Compensated Block Warping](4613-motion-compensated-block-warping.md) by applying a global rigid transform to the entire mesh rather than block-wise warping for compression.
## How it works
**Inputs:**
- A source mesh with vertex positions $\mathbf{v}_{orig}$.
- A 4x4 transformation matrix $M$.
- An Embree geometry object (RTCGeometry) containing the vertex buffer.

**Outputs:**
- Updated vertex buffer in the Embree geometry.
- The source mesh remains unchanged.

**Data Structures:**
- Vertex buffer: Array of 3D floats (x, y, z).
- Transformation matrix: 4x4 float matrix, typically in column-major or row-major format consistent with Embree's expectations.

**Steps:**
1. Retrieve the vertex buffer pointer from the Embree geometry using `rtcGetGeometryBufferData`.
2. Initialize a running sum vector `averagePt` to $(0, 0, 0, 1)$ and `maxlen` to $0$.
3. Iterate through each vertex $i$ from $0$ to $N-1$ (where $N$ is the vertex count):
   a. Reset the current vertex $\mathbf{v}_i$ in the buffer to its original position $\mathbf{v}_{orig, i}$.
   b. Transform the vertex: $\mathbf{v}'_i = M \cdot \mathbf{v}_{orig, i}$.
   c. Update the buffer with the transformed coordinates: $\mathbf{v}_i.x = \mathbf{v}'_i.x$, $\mathbf{v}_i.y = \mathbf{v}'_i.y$, $\mathbf{v}_i.z = \mathbf{v}'_i.z$.
   d. Accumulate the transformed position into the running sum: `averagePt` += $\mathbf{v}'_i$.
   e. Compute the distance $d$ from $\mathbf{v}'_i$ to the current running centroid $\mathbf{c}_i = \text{averagePt} / \max(1, i+1)$.
   f. If $d > \text{maxlen}$, update $\text{maxlen} = d$.
4. After the loop, compute the final centroid $\mathbf{cp} = \text{averagePt} / N$.
5. Commit the geometry and scene updates in Embree.

**Pitfalls:**
- The incremental computation of `maxlen` (distance to the running centroid) is mathematically incorrect for finding the maximum distance from the final centroid. It should be computed after the loop using the final centroid $\mathbf{cp}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_mesh_embree(mesh, M, rtc_geometry):
    # Get vertex buffer
    vertices = rtcGetGeometryBufferData(rtc_geometry)
    N = len(vertices)
    original_positions = mesh.vertices # Assume this is stored
    
    average_pt = [0.0, 0.0, 0.0, 1.0]
    
    for i in range(N):
        # Reset to original
        v_orig = original_positions[i]
        
        # Transform
        v_trans = multiply_matrix_vector(M, v_orig)
        
        # Update buffer
        vertices[i].x = v_trans.x
        vertices[i].y = v_trans.y
        vertices[i].z = v_trans.z
        
        # Accumulate
        average_pt[0] += v_trans.x
        average_pt[1] += v_trans.y
        average_pt[2] += v_trans.z
        # average_pt[3] remains 1.0 for counting purposes in logic, 
        # but strictly it's a 4D vector sum. For centroid, we divide by N.
        
    # Compute final centroid
    centroid = [average_pt[0]/N, average_pt[1]/N, average_pt[2]/N]
    
    # Commit changes
    rtcCommitGeometry(rtc_geometry)
    rtcCommitScene(rtc_geometry.scene)
```
## Subtopics

### [Mesh Transformation](6214-mesh-transformation.md)
Applies a 4x4 transformation matrix to all vertices and normals of a mesh, updating the source mesh data in place while preserving the original positions in the source buffer. It differs from its parent [Mesh Transformation via Embree RTC](5669-mesh-transformation-via-embree-rtc.md) by operating directly on the mesh vertex array rather than maintaining a separate Embree geometry buffer with internal transformation matrices.

## Operating characteristics
Not given in the sources.
## Background
This method is associated with the FLOD project, which involves Gaussian splatting and rendering metrics [S1, unofficial].
## Sources
[S1] whu-lyh/flod, https://github.com/whu-lyh/flod, fetched 2026-10-27 (unofficial)
