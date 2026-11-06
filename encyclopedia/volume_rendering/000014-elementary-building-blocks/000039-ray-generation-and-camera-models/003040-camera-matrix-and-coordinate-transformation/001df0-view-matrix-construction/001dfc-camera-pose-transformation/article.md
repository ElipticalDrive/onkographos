# Camera Pose Transformation

Also called: Camera Offset and Rotation, Camera Pose Representation, Pose Transformation, RotateX

## Overview
Camera Pose Transformation applies geometric transformations—specifically rotation and translation—to align a camera or scene within a 3D coordinate system. It solves the problem of positioning the viewing origin and orientation relative to the world or object space. It differs from [View Matrix Construction](1644-view-matrix-construction.md) by operating on individual pose components (rotation angles, translation vectors) rather than constructing the final 4x4 view matrix from basis vectors. It differs from [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md) by focusing on the construction and application of the pose itself, whereas the sibling method handles the coordinate frame conversion and rigid body application.

## How it works
**Inputs:**
- Point $p \in \mathbb{R}^3$: The 3D point to transform.
- Translation vector $t \in \mathbb{R}^3$: The offset to apply.
- Rotation parameters: Either an axis-angle pair or specific axis rotation angles (e.g., $rotY$).
- Scale factor $s \in \mathbb{R}$: (Optional, for RigidXform) Uniform scale converting local units to world units.

**Outputs:**
- Transformed point $p' \in \mathbb{R}^3$.
- Or a transformation matrix/structure representing the pose.

**Data Structures:**
- **Pose (SE(3) representation):** A $3 \times 4$ matrix $P$ containing the rotation matrix $R_p \in SO(3)$ and translation vector $t_p \in \mathbb{R}^3$.
  $$ P = \begin{bmatrix} R_p & t_p \end{bmatrix} = \begin{bmatrix} r_{11} & r_{12} & r_{13} & t_x \\ r_{21} & r_{22} & r_{23} & t_y \\ r_{31} & r_{32} & r_{33} & t_z \end{bmatrix} $$
  The full $4 \times 4$ Lie group representation includes a bottom row $[0, 0, 0, 1]$, but is often dropped to $3 \times 4$ for compactness [S1].
- **RigidXform:** A structure mapping local cell space to world metres, composed of:
  - `cell_size_m`: Uniform scale factor.
  - `rotation`: Normalized quaternion or rotation matrix.
  - `translation`: 3D vector.

**Steps:**

1. **Translation:** Apply the translation offset to the point.
   $$ p_{trans} = p + t $$
   In specific implementations, a fixed offset is used to position the camera/scene origin. For example, offsetting by $[0, -1, -2.2]$ [S3, unofficial].

2. **Rotation:** Apply rotation around a specified axis.
   - **Axis-Specific Rotation (e.g., Y-axis):**
     For a rotation angle $\theta$ around the Y-axis, the transformation affects the $y$ and $z$ components:
     $$ y' = y \cos(\theta) + z \sin(\theta) $$
     $$ z' = -y \sin(\theta) + z \cos(\theta) $$
     $$ x' = x $$
     This is equivalent to applying a 2D rotation matrix to the $y,z$ plane [S2, S3, unofficial].
   - **General Rotation:**
     If using a general rotation matrix $R$ derived from axis-angle or quaternions:
     $$ p_{rot} = R \cdot p_{trans} $$

3. **Scaling (RigidXform):**
   If mapping from local cell space to world metres:
   $$ p_{world} = s \cdot (R \cdot p_{local}) + t $$
   Where $s$ is `cell_size_m` [S4, unofficial].

4. **Pose Composition:**
   To represent the full pose $P$, combine $R$ and $t$ into the $3 \times 4$ matrix. If noise is introduced (e.g., in bundle adjustment), generate noise $n \in \mathbb{R}^6$ from a normal distribution, convert to a transformation matrix $N$, and compose:
   $$ \hat{P} = N \cdot P $$
   Where $\hat{P}$ is the imperfect pose [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rotate_y(p: list[float], angle: float) -> list[float]:
    """
    Rotates point p around the Y-axis by angle (radians).
    Based on [S2, S3, unofficial].
    """
    x, y, z = p
    cos_a = math.cos(angle)
    sin_a = math.sin(angle)
    
    # Apply 2D rotation to y and z components
    new_y = y * cos_a + z * sin_a
    new_z = -y * sin_a + z * cos_a
    
    return [x, new_y, new_z]

def apply_pose_transformation(p: list[float], translation: list[float], 
                              rotation_matrix: list[list[float]]) -> list[float]:
    """
    Applies translation and rotation to point p.
    Based on [S1][S3, S4, unofficial].
    """
    # Step 1: Translation
    # p_trans = p + translation
    p_trans = [p[0] + translation[0], 
               p[1] + translation[1], 
               p[2] + translation[2]]
    
    # Step 2: Rotation
    # p_rot = R * p_trans
    r = rotation_matrix
    x, y, z = p_trans
    new_x = r[0][0]*x + r[0][1]*y + r[0][2]*z
    new_y = r[1][0]*x + r[1][1]*y + r[1][2]*z
    new_z = r[2][0]*x + r[2][1]*y + r[2][2]*z
    
    return [new_x, new_y, new_z]

def construct_rigid_xform(cell_size: float, quaternion: list[float], 
                          translation: list[float]) -> dict:
    """
    Constructs a RigidXform from quaternion and translation.
    Based on [S4, unofficial].
    """
    # Dequantize and normalize quaternion
    # Reject if orientation is zero
    norm = math.sqrt(sum(q*q for q in quaternion))
    if norm == 0:
        raise ValueError("Zero orientation")
    
    normalized_q = [q / norm for q in quaternion]
    
    return {
        "cell_size_m": cell_size,
        "rotation": normalized_q,
        "translation": translation
    }

def transform_with_rigid_xform(p_local: list[float], xform: dict) -> list[float]:
    """
    Transforms local point to world space using RigidXform.
    local_cell -> scale -> rotate -> translate.
    Based on [S4, unofficial].
    """
    # Note: Quaternion to matrix conversion is assumed to be handled 
    # by a helper or standard library, as the core definition of 
    # quaternion-to-matrix is not fully detailed in the provided notes 
    # for this specific method, but RigidXform relies on it.
    # [G] Standard quaternion-to-rotation-matrix conversion required.
    R = quaternion_to_rotation_matrix(xform["rotation"])
    
    # Scale
    p_scaled = [x * xform["cell_size_m"] for x in p_local]
    
    # Rotate
    x, y, z = p_scaled
    r = R
    new_x = r[0][0]*x + r[0][1]*y + r[0][2]*z
    new_y = r[1][0]*x + r[1][1]*y + r[1][2]*z
    new_z = r[2][0]*x + r[2][1]*y + r[2][2]*z
    
    # Translate
    t = xform["translation"]
    return [new_x + t[0], new_y + t[1], new_z + t[2]]
```

**Parameters:**
- `angle`: Float representing the rotation angle in radians [S2, unofficial].
- `cell_size_m`: Uniform scale factor converting local cells to world metres [S4, unofficial].
- `translation`: 3D vector $[t_x, t_y, t_z]$ [S1].

**Edge Cases and Pitfalls:**
- **Zero Orientation:** When constructing RigidXform from a quaternion, a zero orientation must be rejected [S4, unofficial].
- **Precision:** The stored rotation in RigidXform is normalized, ensuring the conjugate is the exact inverse, which is critical for round-trip transformations (e.g., testing identity, pure translation, arbitrary-axis rotations) [S4, unofficial].
- **Boundary Conditions:** In ray traversal using these transformations, origins exactly on a boundary belong to the cell defined by the floor function [S4, unofficial].

## Subtopics

### RotateY
RotateY is a geometric transformation primitive that rotates a point or shape around the Y-axis in 3D space [S2, unofficial]. It serves as a fundamental building block for constructing camera poses and deforming Signed Distance Fields (SDFs) [G]. Unlike [Rotation Transformation](2279-rotation-transformation.md), which may imply a general rotation matrix, RotateY specifically isolates the Y-axis component [G]. Unlike [Twist](1722-twist.md), it applies a uniform rotation angle to the entire object rather than a spatially varying one [G]. It is a sibling to [RotateX](2278-rotatex.md) and [RotateZ](2280-rotatez.md) [S2, unofficial].

### [Pose Recentering](1786-pose-recentering.md)
Recenters a set of poses such that their average pose becomes the identity transformation (origin at center, aligned axes).

### [Rotation Transformation](2279-rotation-transformation.md)
Rotation Transformation constructs a 4×4 homogeneous rotation matrix to rotate a shape or vector around a specified axis by a given angle. It sets itself apart from [Quaternion Rotation](2302-quaternion-rotation.md) by yielding an explicit matrix representation rather than using quaternion algebra, and from [SE2 Lie Group Implementation](3144-se2-lie-group-implementation.md) by operating in 3D space rather than 2D [G].

### [Rotation Matrix Computation](2280-rotation-matrix-computation.md)
Computes a 3x3 rotation matrix from a rotation axis and angle using Rodrigues' rotation formula. It differs from [RotateY](1720-rotatey.md) and [Rotation Transformation](2279-rotation-transformation.md) by accepting an arbitrary axis vector rather than fixed coordinate axes [G]. It constructs the matrix directly from geometric parameters rather than decomposing an existing matrix like [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md) [G].

### [Translation Transformation](2281-translation-transformation.md)
A geometric transformation that moves a shape by a given vector. It adjusts positional offsets without altering orientation, distinguishing it from rotation transformations [G].

### [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md)
This method converts C-arm geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center, table translation) into camera position, look-at point, and up vector in RAS coordinates. It explicitly models the C-arm's mechanical degrees of freedom (AP and lateral tilts) relative to a patient-centered coordinate system to generate synthetic Digital Radiography (DRR) views. Unlike sibling methods that handle isolated rotations or translations, this method composes them into a complete C-arm pose transformation pipeline [1720-rotatey.md][2279-rotation-transformation.md][2280-rotation-matrix-computation.md][2281-translation-transformation.md][3164-camera-matrix-decomposition.md].

### [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)
Decomposes a 3x4 projection matrix into intrinsic matrix $K$, rotation matrix $R$, and translation vector $t$. It extracts the camera's internal parameters and external pose from a combined projection matrix, enabling the separation of optical properties from physical camera position and orientation.

### [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md)
This method converts 3D points and vectors into homogeneous coordinates and applies a rigid body transformation (rotation and translation) to map them between coordinate frames, typically from camera space to world space. It serves as the core geometric primitive for ray generation and coordinate alignment in volume rendering pipelines. It differs from pure rotation or translation methods by handling both operations simultaneously via a single transformation matrix, and from decomposition methods by applying the full pose rather than extracting components. It builds on [Camera Pose Transformation](1672-camera-pose-transformation.md).

#### RotateZ
Similar to RotateX, but rotates around the z-axis instead [S2, unofficial].
**Parameter:** `angle`: A float input representing the angle of rotation (implied by similarity to RotateX) [S2, unofficial].

#### RigidXform
A transformation structure that maps a volume's local cell space to world metres by applying a uniform cell scale, a rotation, and a translation [S4, unofficial].
**Steps:** Construct RigidXform from a Pose by dequantising the quaternion, renormalising it, and rejecting if the orientation is zero [S4, unofficial].
**Parameter:** `cell_size_m`: uniform scale factor converting local cells to world metres [S4, unofficial].
**Pitfall:** The stored rotation is normalised, ensuring the conjugate is the exact inverse of the rotation [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for point-wise transformation. Matrix construction is $O(1)$ for fixed-size matrices.
- **Memory Complexity:** $O(1)$ for storing a single pose ($3 \times 4$ matrix or equivalent quaternion/vector).
- **Performance:** Efficient for real-time raymarching and SDF deformation [S3, unofficial].
- **Strengths:** Simple to implement for axis-aligned rotations; provides a clear separation of rotation and translation [S2, S3, unofficial].
- **Limitations:** Axis-specific rotations (RotateX/Y/Z) are not composable for arbitrary orientations without combining multiple steps or using general rotation matrices [S2, unofficial].
- **When to use:** When the camera motion is constrained to specific axes or when constructing poses from simple angular offsets [S3, unofficial].
- **When not to use:** For arbitrary 3D rotations where a general rotation matrix or quaternion is more appropriate [S1].

## Background
Camera pose transformation is fundamental to computer graphics and vision, rooted in the representation of rigid body motion in 3D space. The mathematical foundation lies in the Special Euclidean group SE(3), which describes rigid transformations as a combination of rotation (SO(3)) and translation [S1]. In volume rendering and neural radiance fields (NeRF), accurate camera poses are critical for ray generation and scene reconstruction. Methods like CBARF use SE(3) to model and optimize noisy camera poses [S1]. In raymarching, simple transformations like RotateY are used to animate or position SDFs [S3, unofficial]. The RigidXform structure is used in voxel-based rendering to map discrete cell spaces to continuous world coordinates [S4, unofficial].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S2] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S3] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S4] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
