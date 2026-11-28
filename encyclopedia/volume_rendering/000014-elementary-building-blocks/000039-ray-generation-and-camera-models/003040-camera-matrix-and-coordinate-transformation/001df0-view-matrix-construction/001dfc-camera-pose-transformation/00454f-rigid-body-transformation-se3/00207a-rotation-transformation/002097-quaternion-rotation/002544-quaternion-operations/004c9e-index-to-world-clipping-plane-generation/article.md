# Index-to-World Clipping Plane Generation

Also called: IJK Bounds Clipping Plane Generation, World-Space Plane Cropping from IJK Bounds, ijkPlanes to VTK Clipping Planes

## Overview
This method computes six world-space clipping planes from IJK index bounds to crop volume data [S2, unofficial]. It aligns rectangular crop regions with the physical orientation of the volume data by transforming index-space bounds into world-space plane primitives [S2, unofficial]. It differs from inverse transformation methods by constructing planes directly from index bounds rather than transforming existing world-space geometry [G].

## How it works
**Inputs:**
- `indexToWorld`: A $4 \times 4$ transformation matrix mapping voxel indices to world coordinates [S2, unofficial].
- `ijkPlanes`: An array of six integers representing the bounding box in index space: `[iMin, iMax, jMin, jMax, kMin, kMax]` [S2, unofficial].

**Outputs:**
- A collection of six clipping plane objects (e.g., `vtkPlane` instances), each defined by an origin point and a normal vector in world space [S2, unofficial].

**Data Structures:**
- **Transformation Matrix**: A $4 \times 4$ matrix where the upper-left $3 \times 3$ submatrix contains rotation and scaling, and the fourth column contains translation [S2, unofficial].
- **Clipping Plane**: Defined by a point $P$ (origin) and a normal vector $\mathbf{n}$ [S2, unofficial].

**Steps:**
1. **Extract Rotation**: Retrieve the rotation quaternion from the `indexToWorld` matrix using `mat4.getRotation(rotation, imageData.getIndexToWorld())` [S2, unofficial].
2. **Define Rotation Helper**: Create a function `rotateVec(vec)` that transforms a 3D vector by the extracted rotation quaternion using `vec3.transformQuat(out, vec, rotation)` [S2, unofficial].
3. **Parse Bounds**: Extract the six integer bounds from the `ijkPlanes` array: `[iMin, iMax, jMin, jMax, kMin, kMax] = ijkPlanes` [S2, unofficial].
4. **Compute World Corners**: Calculate the world coordinates for the crop origin and corner by transforming the minimum and maximum voxel indices: `origin = indexToWorld([iMin, jMin, kMin])` and `corner = indexToWorld([iMax, jMax, kMax])` [S2, unofficial].
5. **Generate Plane Normals**: Define the standard basis vectors $\mathbf{e}_i = [1, 0, 0]$, $\mathbf{e}_j = [0, 1, 0]$, $\mathbf{e}_k = [0, 0, 1]$ and rotate them using the helper [S2, unofficial].
6. **Construct Planes**: Instantiate six planes: X-min (normal rotated $[1,0,0]$ at origin), X-max (normal rotated $[-1,0,0]$ at corner), Y-min (normal rotated $[0,1,0]$ at origin), Y-max (normal rotated $[0,-1,0]$ at corner), Z-min (normal rotated $[0,0,1]$ at origin), Z-max (normal rotated $[0,0,-1]$ at corner) [S2, unofficial].
7. **Apply to Renderer**: Clear existing clipping planes from the volume mapper and add the newly generated planes using `mapper.removeAllClippingPlanes(); planes.forEach((plane) => { mapper.addClippingPlane(plane); }); mapper.modified()` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_clipping_planes(index_to_world_matrix, ijk_planes):
    # 1. Extract rotation quaternion from the 4x4 matrix
    rotation = extract_quaternion_from_matrix(index_to_world_matrix)

    # 2. Define vector rotation helper
    def rotate_vec(vec):
        return quat_rotate_vec(rotation, vec)

    # 3. Parse bounds
    i_min, i_max, j_min, j_max, k_min, k_max = ijk_planes

    # 4. Compute world corners
    origin_world = transform_point(index_to_world_matrix, [i_min, j_min, k_min])
    corner_world = transform_point(index_to_world_matrix, [i_max, j_max, k_max])

    # 5. Define standard basis vectors
    e_i = [1.0, 0.0, 0.0]
    e_j = [0.0, 1.0, 0.0]
    e_k = [0.0, 0.0, 1.0]

    # 6. Rotate basis vectors to world space
    n_i = rotate_vec(e_i)
    n_j = rotate_vec(e_j)
    n_k = rotate_vec(e_k)

    # 7. Construct planes
    # Min planes use origin_world and rotated positive unit vectors as normals
    plane_x_min = VTK_Plane(origin=origin_world, normal=n_i)
    plane_y_min = VTK_Plane(origin=origin_world, normal=n_j)
    plane_z_min = VTK_Plane(origin=origin_world, normal=n_k)

    # Max planes use corner_world and rotated negative unit vectors as normals
    plane_x_max = VTK_Plane(origin=corner_world, normal=[-n_i[0], -n_i[1], -n_i[2]])
    plane_y_max = VTK_Plane(origin=corner_world, normal=[-n_j[0], -n_j[1], -n_j[2]])
    plane_z_max = VTK_Plane(origin=corner_world, normal=[-n_k[0], -n_k[1], -n_k[2]])

    return [plane_x_min, plane_y_min, plane_z_min, plane_x_max, plane_y_max, plane_z_max]

# Helper definitions
def extract_quaternion_from_matrix(matrix):
    # Extracts the rotation component from a 4x4 affine matrix [G]
    return matrix_to_quaternion(matrix[:3, :3])

def transform_point(matrix, point):
    # Applies 4x4 transformation matrix to a 3D point [G]
    x, y, z = point
    m = matrix
    return [
        m[0][0]*x + m[0][1]*y + m[0][2]*z + m[0][3],
        m[1][0]*x + m[1][1]*y + m[1][2]*z + m[1][3],
        m[2][0]*x + m[2][1]*y + m[2][2]*z + m[2][3]
    ]

def quat_rotate_vec(q, v):
    # Standard quaternion-vector multiplication: v' = q * v * q* [G]
    # Assumes q is a normalized quaternion [w, x, y, z]
    qx, qy, qz = q[1], q[2], q[3]
    vx, vy, vz = v[0], v[1], v[2]
    return [
        2*(qx*vx + qy*vy + qz*vz)*qx + (1 - 2*(qx**2 + qy**2 + qz**2))*vx,
        2*(qx*vx + qy*vy + qz*vz)*qy + (1 - 2*(qx**2 + qy**2 + qz**2))*vy,
        2*(qx*vx + qy*vy + qz*vz)*qz + (1 - 2*(qx**2 + qy**2 + qz**2))*vz
    ]

class VTK_Plane:
    def __init__(self, origin, normal):
        self.origin = origin
        self.normal = normal
```

**Parameters:**
- `ijkPlanes`: Array of 6 integers `[minI, maxI, minJ, maxJ, minK, maxK]` [S2, unofficial].
- `indexToWorld`: $4 \times 4$ matrix [S2, unofficial].

**Edge Cases:**
- If the `indexToWorld` matrix includes non-uniform scaling, the normal vectors derived from rotating basis vectors may not be unit length. Normalization of the resulting normals is required for correct clipping behavior [G].
- Degenerate bounds (e.g., $i_{min} > i_{max}$) should be handled or validated before generation [G].

## Subtopics

### Dynamic Cropping Plane Generation
This variant generates clipping planes dynamically based on user interaction or changing crop regions [S3, unofficial]. It follows the same core logic but updates the `ijkPlanes` input in real-time [S3, unofficial]. The key distinction is the continuous recalculation of the `indexToWorld` transformation if the volume orientation changes, or the re-evaluation of the crop bounds as the user drags handles [S3, unofficial]. It is chosen when interactive volume exploration requires precise, axis-aligned cropping in world space [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ for plane generation, as it involves a fixed number of matrix-vector multiplications and plane constructions regardless of volume size [G].
- **Memory Complexity**: $O(1)$, storing only six plane objects [G].
- **Strengths**: Accurately respects the physical orientation of the volume data; efficient for interactive clipping [G].
- **Limitations**: Requires access to the full index-to-world transformation matrix; assumes the clipping region is axis-aligned in index space [G].
- **When to use**: When cropping a volume based on voxel indices but needing the clip boundaries to align with the world coordinate system [G].
- **When not to use**: When the clipping region is defined in world space directly (use inverse transformation) or when the volume has no spatial transformation (identity matrix) [G].

## Background
This method is commonly used in web-based volume rendering applications using VTK.js [S1, S2, unofficial]. It bridges the gap between logical voxel indices and physical world coordinates, enabling intuitive cropping interfaces where users define regions in index space but the renderer operates in world space [G]. It is part of the broader set of coordinate transformation techniques used in medical imaging and scientific visualization [G].

## Sources
[S1] AbdelrahmanKK/Volume-Rendering-Web-Application-VTK, https://github.com/AbdelrahmanKK/Volume-Rendering-Web-Application-VTK, fetched 2026-10-27 (unofficial)
[S2] GhadaDx/Volume-Rendering-VTKjs, https://github.com/GhadaDx/Volume-Rendering-VTKjs, fetched 2026-10-27 (unofficial)
[S3] mohammed-abo-arab/3D_WebGL_VolumeRendering, https://github.com/mohammed-abo-arab/3D_WebGL_VolumeRendering, fetched 2026-10-27 (unofficial)
