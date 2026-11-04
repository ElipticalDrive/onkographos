# Camera Look-at Basis Construction
Also called: Camera Coordinate System Construction, Camera Matrix Generation, LookAt Basis Construction
## Overview
Constructs a 3x3 rotation matrix that transforms world-space vectors into camera-local space, given a camera origin and look-at target [S1, unofficial]. It builds an orthogonal camera coordinate system (basis) from a camera center and a look-at point by deriving an up vector orthogonal to the viewing direction, then computing the right vector via cross product [S2, unofficial]. It differs from [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) by accepting an arbitrary look-at target rather than spherical angles, and from [General Perspective Transformation](1845-general-perspective-transformation.md) by producing only the rotational component of the extrinsic transform, not the full projection matrix.
## How it works
**Inputs:**
- `ro`: Camera origin position (world space, vec3) [S1, unofficial].
- `lookAt`: Target point in world space (vec3) [S1, unofficial].
- `up`: Fixed world up vector (vec3, typically `(0, 1, 0)` or `(0, 0, 1)` depending on convention) [S1, S2, unofficial].

**Outputs:**
- `M`: A 3x3 rotation matrix where columns (or rows, depending on convention) are the camera's local right, up, and depth vectors [S1, unofficial].
- `basis`: A 4x4 homogeneous matrix if the full extrinsic transform is required, constructed by embedding the 3x3 basis and adding a homogeneous row `[0, 0, 0, 1]` [S2, unofficial].

**Data Structures:**
- Vectors are 3-component floating-point arrays.
- The basis matrix is 3x3 (rotation only) or 4x4 (extrinsic transform).

**Steps:**
1. Calculate the camera viewing direction (depth vector) $d$ as the normalized vector from the camera origin to the look-at target: $d = \text{normalize}(ro - lookAt)$ [S1, unofficial]. Note: [S2, unofficial] defines the depth vector as $lookAt - cameraCenter$ and uses it directly without normalization in the projection step, but normalizes the final basis vectors. The direction sign determines if the camera looks along $+Z$ or $-Z$ in local space.
2. Define the world up vector $z_{world}$ (or $up$). [S1, unofficial] uses $(0, 1, 0)$. [S2, unofficial] uses the world Z-axis $(0, 0, 1)$ as the reference for projection.
3. Compute the camera's local up vector $u$. To ensure orthogonality, project the world up vector onto the plane orthogonal to the viewing direction $d$.
   - Calculate the projection scalar $\lambda$:
     $$ \lambda = \frac{z_{world} \cdot d}{d \cdot d} $$
   - Calculate the orthogonal up vector:
     $$ u = z_{world} - \lambda d $$
   [S2, unofficial]. Alternatively, [S1, unofficial] computes the right vector first using the cross product of the world up and the depth vector, then derives the true up vector.
4. Compute the camera's local right vector $r$.
   - Using [S1, unofficial] approach: $r = \text{normalize}(up \times d)$ [S1, unofficial].
   - Using [S2, unofficial] approach: $r = \text{cross}(d, u)$ [S2, unofficial]. Note that the order of cross product determines the handedness.
5. Compute the camera's local up vector $cu$ (if not already computed as orthogonal).
   - $cu = \text{normalize}(d \times r)$ [S1, unofficial].
   - If using [S2, unofficial]'s projection method, $u$ is already orthogonal to $d$, but $r$ must be orthogonal to both. [S2, unofficial] computes $r = \text{cross}(d, u)$ and then implicitly or explicitly ensures orthogonality.
6. Normalize all three vectors $r, u, d$ to unit length [S1, S2, unofficial].
7. Construct the 3x3 rotation matrix $M$.
   - [S1, unofficial] returns `mat3(cr, cu, cd)`, implying columns are right, up, and depth [S1, unofficial].
   - [S2, unofficial] stacks vectors into an inhomogeneous basis `[camera_x_vector, camera_up_vector, depth_vector]` and adds a homogeneous row `[0, 0, 0, 1]` to create a 4x4 matrix [S2, unofficial].
8. (Optional) Combine with intrinsic parameters to form the full camera transform matrix $T = K \cdot E$, where $E$ is the extrinsic matrix derived from the basis [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cross(v1, v2):
    return [
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    ]

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def normalize(v):
    length = math.sqrt(dot(v, v))
    if length == 0:
        return [0, 0, 0] # [G] Guard against zero vector
    return [v[0]/length, v[1]/length, v[2]/length]

def camera_look_at_basis(lookAt, ro, up_world=None):
    """
    Constructs a 3x3 rotation matrix transforming world vectors to camera space.
    Based on [S1, unofficial] and [S2, unofficial].
    """
    if up_world is None:
        up_world = [0, 1, 0] # Default up vector [S1, unofficial]

    # 1. Calculate depth vector (viewing direction)
    # [S1, unofficial] uses ro - lookAt, [S2, unofficial] uses lookAt - cameraCenter.
    # We follow [S1, unofficial] convention for negative Z camera space, but normalize.
    depth_vector = [lookAt[0] - ro[0], lookAt[1] - ro[1], lookAt[2] - ro[2]]
    cd = normalize(depth_vector)

    # 2. Calculate right vector
    # [S1, unofficial] uses cross(up, cd). [S2, unofficial] uses cross(depth, up_proj).
    # We use [S1, unofficial] method as it is more standard for "lookAt" functions.
    # Note: [S1, unofficial] code shows cross(cz, vec3(0,1,0)) which is cross(cd, up).
    # Let's stick to the explicit code in [S1, unofficial]:
    # vec3 cr = normalize(cross(cz, vec3(0., 1., 0.)));
    # This is cross(cd, up_world).
    cr = normalize(cross(cd, up_world))

    # 3. Calculate true up vector
    # [S1, unofficial] uses cross(cd, cr).
    cu = normalize(cross(cd, cr))

    # 4. Construct 3x3 matrix
    # [S1, unofficial] returns mat3(cr, cu, cd), meaning columns are cr, cu, cd.
    # Matrix M such that M * world_vector = camera_vector.
    # M = [ cr.x, cu.x, cd.x ]
    #     [ cr.y, cu.y, cd.y ]
    #     [ cr.z, cu.z, cd.z ]
    M = [
        [cr[0], cu[0], cd[0]],
        [cr[1], cu[1], cd[1]],
        [cr[2], cu[2], cd[2]]
    ]
    
    return M

def camera_basis_from_depth(camera_depth_z_vector, up_world=None):
    """
    Alternative construction based on [S2, unofficial].
    Projects world Z-axis onto plane orthogonal to depth vector.
    """
    if up_world is None:
        up_world = [0, 0, 1] # [S2, unofficial] uses world Z-axis as reference

    depth_vector = camera_depth_z_vector[:3]
    
    # Calculate projection lambda
    dot_dd = dot(depth_vector, depth_vector)
    if dot_dd == 0:
        return None # [G] Guard against zero depth vector
    
    cartesian_z_projection_lambda = dot(depth_vector, up_world) / dot_dd
    
    # Calculate camera up vector (orthogonal to depth)
    camera_up_vector = [
        up_world[0] - cartesian_z_projection_lambda * depth_vector[0],
        up_world[1] - cartesian_z_projection_lambda * depth_vector[1],
        up_world[2] - cartesian_z_projection_lambda * depth_vector[2]
    ]
    
    # Calculate camera x vector (right)
    camera_x_vector = cross(depth_vector, camera_up_vector)
    
    # Normalize
    camera_x_vector = normalize(camera_x_vector)
    camera_up_vector = normalize(camera_up_vector)
    depth_vector = normalize(depth_vector)
    
    # Stack into 4x4 homogeneous basis
    # [S2, unofficial] stacks [x, up, depth] as rows? Or columns?
    # "Stack vectors into inhomogeneous basis: [camera_x_vector, camera_up_vector, depth_vector, [0,0,0]]"
    # "Add homogeneous row [0,0,0,1]"
    # This implies the first 3 rows are the vectors.
    homogeneous_basis = [
        [camera_x_vector[0], camera_x_vector[1], camera_x_vector[2], 0],
        [camera_up_vector[0], camera_up_vector[1], camera_up_vector[2], 0],
        [depth_vector[0], depth_vector[1], depth_vector[2], 0],
        [0, 0, 0, 1]
    ]
    
    return homogeneous_basis
```

**Parameters:**
- `lookAt`: Target point in world space [S1, unofficial].
- `ro`: Camera origin position [S1, unofficial].
- `up`: Fixed up vector, typically `(0, 1, 0)` [S1, unofficial] or `(0, 0, 1)` [S2, unofficial].

**Edge Cases:**
- If `ro` equals `lookAt`, the depth vector is zero, leading to division by zero or undefined normalization [G].
- If the viewing direction is parallel to the world up vector, the cross product for the right vector will be zero, leading to an undefined basis [G].
- Precision issues may arise if the depth vector is very small [G].

## Operating characteristics
- **Time Complexity:** $O(1)$, involving a constant number of vector operations (subtraction, dot product, cross product, normalization).
- **Memory Complexity:** $O(1)$, storing a few 3D vectors and a 3x3 or 4x4 matrix.
- **Strengths:** Simple to implement, provides a direct way to orient a camera towards a specific point.
- **Limitations:** Can suffer from gimbal lock-like issues if the viewing direction aligns with the up vector (though less severe than Euler angles). The choice of up vector affects the roll of the camera.
- **When to use:** When the camera position and target are known, and a specific orientation is required.
- **When not to use:** When the camera orientation is defined by angles (use [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md)) or when a full projection matrix is needed directly (use [General Perspective Transformation](1845-general-perspective-transformation.md)).

## Background
The look-at function is a standard technique in computer graphics for defining camera orientation. It is widely used in ray tracing, rendering, and 3D modeling. The specific implementation in [S1, unofficial] is from a signed distance field (SDF) ray marching example. The implementation in [S2, unofficial] is from a PyTorch guide for Plenoxels/NeRF, focusing on constructing camera matrices for neural rendering.

## Sources
[S1] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S2] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
