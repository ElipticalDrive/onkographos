# Camera Pose Transformation

Also called: Camera Offset and Rotation, Camera Pose Representation, Pose Transformation, RotateX

## Overview
Camera Pose Transformation applies a rigid body transformation (rotation and translation) to camera-to-world matrices, typically to adjust the viewing geometry or correct for noisy pose estimates in volume rendering and neural radiance fields. It operates on the Lie group $SE(3)$, ensuring that the resulting transformation preserves the scale of the rotation part [S3, unofficial]. Unlike [View Matrix Construction](1644-view-matrix-construction.md), which builds a matrix from scratch using basis vectors, this method applies a pre-defined transformation to an existing pose representation [S3, unofficial]. It differs from [Average Pose Calculation](1645-average-pose-calculation.md) by applying a specific geometric transform rather than computing a statistical center, and from [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) by not deriving the basis from orbital mechanics [S3, unofficial].

## How it works
**Inputs:**
1.  `camtoworlds`: A batch of $N$ camera-to-world transformation matrices, shape $(N, 4, 4)$ [S3, unofficial].
2.  `matrix`: A $4 \times 4$ transformation matrix representing the pose adjustment [S3, unofficial].
3.  `point` (optional): A 3D point to be transformed directly, often used for scene offsetting [S4, unofficial].
4.  `angle` (optional): A float representing the rotation angle in radians, used for axis-specific rotations [S2, unofficial].

**Outputs:**
1.  `transformed_camtoworlds`: The batch of adjusted camera-to-world matrices, shape $(N, 4, 4)$ [S3, unofficial].
2.  `transformed_point`: The adjusted 3D point [S4, unofficial].

**Data Structures:**
*   **Camera Pose ($P$):** Represented as a $3 \times 4$ matrix $[R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. This is the top $3 \times 4$ portion of the full $4 \times 4$ homogeneous matrix.
*   **Lie Group $SE(3)$:** Defined as:
    $$ SE(3) = \left\{ \begin{bmatrix} R & t \\ 0^T & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4} \mid R \in SO(3), t \in \mathbb{R}^3 \right\} $$
    [S1]

**Steps:**

1.  **Validate Inputs:** Ensure `camtoworlds` is a tensor of shape $(N, 4, 4)$ and `matrix` is a $4 \times 4$ matrix [S3, unofficial].

2.  **Apply Batch Transformation:** Multiply the batch of camera matrices by the transformation matrix. Using Einstein summation notation:
    $$ \text{camtoworlds}_{new} = \text{einsum}('nij, ki \rightarrow nkj', \text{camtoworlds}, \text{matrix}) $$
    [S3, unofficial]
    This applies the transformation to each camera in the batch.

3.  **Normalize Rotation (SE(3) Constraint):** To ensure the transformation preserves pure rotation without scaling artifacts, normalize the rotation part of the resulting matrices.
    a. Compute the scaling factor for each camera's rotation basis vectors:
    $$ \text{scaling}_i = \| \text{camtoworlds}_{new}[i, 0:3, 0:3]_{\text{col}_0} \|_2 $$
    *(Note: The source computes norm on the first row/column slice `[:, 0, :3]`. In standard column-major or row-major conventions, this extracts the first basis vector. We assume the source implies normalizing the rotation submatrix to unit length basis vectors.)*
    b. Divide the rotation submatrix by the scaling factor:
    $$ \text{camtoworlds}_{new}[i, 0:3, 0:3] = \frac{\text{camtoworlds}_{new}[i, 0:3, 0:3]}{\text{scaling}_i} $$
    [S3, unofficial]

4.  **Point Offset (Optional):** If transforming a specific point `p` to adjust the scene origin:
    a. Apply offset:
    $$ p = p + [0, -1, -2.2] $$
    [S4, unofficial]

5.  **Axis-Specific Rotation (Optional):** If applying a rotation around the Y-axis by angle $\theta$:
    a. Extract y and z components: $y = p[1], z = p[2]$.
    b. Apply 2D rotation:
    $$ y' = y \cos(\theta) + z \sin(\theta) $$
    $$ z' = -y \sin(\theta) + z \cos(\theta) $$
    c. Update point: $p[1] = y', p[2] = z'$.
    [S4, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def transform_camera_poses(camtoworlds: np.ndarray, matrix: np.ndarray) -> np.ndarray:
    """
    Applies a 4x4 transformation matrix to a batch of camera-to-world matrices.
    
    Args:
        camtoworlds: Array of shape (N, 4, 4).
        matrix: Array of shape (4, 4).
        
    Returns:
        Transformed camtoworlds of shape (N, 4, 4).
    """
    # Step 1: Validate inputs
    if camtoworlds.shape != (camtoworlds.shape[0], 4, 4):
        raise ValueError("camtoworlds must be Nx4x4")
    if matrix.shape != (4, 4):
        raise ValueError("matrix must be 4x4")

    # Step 2: Apply transformation using einsum
    # 'nij, ki -> nkj' performs matrix multiplication for each camera in the batch
    transformed = np.einsum('nij, ki -> nkj', camtoworlds, matrix)

    # Step 3: Normalize rotation part to ensure pure SE(3)
    # Extract rotation submatrix (first 3 rows, first 3 cols)
    rot_part = transformed[:, :3, :3]
    
    # Compute scaling factor. Source uses norm of first row slice [:, 0, :3].
    # Assuming this extracts the first basis vector (row 0 of the 3x3 rotation block).
    # Note: In standard OpenGL/DirectX conventions, rows or columns represent basis vectors.
    # We follow the source's indexing exactly.
    scaling = np.linalg.norm(rot_part[:, 0, :], axis=1) # Shape (N,)
    
    # Avoid division by zero
    scaling = np.maximum(scaling, 1e-9)
    
    # Normalize the rotation submatrix
    # scaling[:, None, None] broadcasts to (N, 1, 1)
    transformed[:, :3, :3] = rot_part / scaling[:, None, None]

    return transformed

def rotate_point_y_axis(point: np.ndarray, angle: float) -> np.ndarray:
    """
    Rotates a 3D point around the Y-axis.
    
    Args:
        point: Array of shape (3,).
        angle: Rotation angle in radians.
        
    Returns:
        Rotated point.
    """
    p = point.copy()
    
    # Step 4: Offset point (example from source)
    p = p + np.array([0, -1, -2.2])
    
    # Step 5: Rotate around Y-axis
    y = p[1]
    z = p[2]
    
    cos_a = np.cos(angle)
    sin_a = np.sin(angle)
    
    p[1] = y * cos_a + z * sin_a
    p[2] = -y * sin_a + z * cos_a
    
    return p
```

**Parameters:**
*   `angle`: A float input representing the angle of rotation in radians [S2, unofficial]. Typical values depend on the scene scale; e.g., 0.3 radians for specific artistic offsets [S4, unofficial].
*   `cell_size_m`: (In variant `RigidXform`) Uniform scale factor converting local cells to world metres [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Orientation:** If the rotation quaternion is zero (invalid), the transformation should be rejected [S5, unofficial].
*   **Scaling Artifacts:** Without normalization, the rotation part of the matrix may acquire scaling factors, violating the $SO(3)$ constraint. Normalization is required to maintain pure rigid body motion [S3, unofficial].
*   **Precision:** When applying transformations to Signed Distance Fields (SDFs), offsetting points can distort the distance function, requiring careful handling of the resulting geometry [S4, unofficial].

## Subtopics

### RotateY
RotateY is a geometric transformation primitive that rotates a point or shape around the Y-axis in 3D space [S2, unofficial]. It serves as a fundamental building block for constructing camera poses and deforming Signed Distance Fields (SDFs) [G]. Unlike [Rotation Transformation](2279-rotation-transformation.md), which may imply a general rotation matrix, RotateY specifically isolates the Y-axis component [G]. Unlike [Twist](1722-twist.md), it applies a uniform rotation angle to the entire object rather than a spatially varying one [G]. It is a sibling to [RotateX](2278-rotatex.md) and [RotateZ](2280-rotatez.md) [S2, unofficial].

### [Pose Recentering](1786-pose-recentering.md)
Pose Recentering transforms a set of camera poses so that their average pose becomes the identity transformation, effectively centering the scene at the origin and aligning the coordinate axes. This solves the problem of arbitrary camera coordinate systems in Structure-from-Motion (SfM) outputs, which can cause numerical instability and incorrect depth sampling in volume rendering. It differs from [RotateY](1720-rotatey.md) and [Rotation Transformation](2279-rotation-transformation.md) by simultaneously handling both translation (centering) and rotation (axis alignment) via a single rigid transformation derived from the statistical properties of the pose set, rather than applying fixed or incremental rotations.

### [Rotation Transformation](2279-rotation-transformation.md)
Rotation Transformation constructs a 4×4 homogeneous rotation matrix to rotate a shape or vector around a specified axis by a given angle. It sets itself apart from [Quaternion Rotation](2302-quaternion-rotation.md) by yielding an explicit matrix representation rather than using quaternion algebra, and from [SE2 Lie Group Implementation](3144-se2-lie-group-implementation.md) by operating in 3D space rather than 2D [G].

### [Rotation Matrix Computation](2280-rotation-matrix-computation.md)
Computes a 3x3 rotation matrix from a rotation axis and angle using Rodrigues' rotation formula. It differs from [RotateY](1720-rotatey.md) and [Rotation Transformation](2279-rotation-transformation.md) by accepting an arbitrary axis vector rather than fixed coordinate axes [G]. It constructs the matrix directly from geometric parameters rather than decomposing an existing matrix like [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md) [G].

### [Translation Transformation](2281-translation-transformation.md)
A geometric transformation that moves a shape by a given vector. It adjusts positional offsets without altering orientation, distinguishing it from rotation transformations [G].

### [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md)
This method converts C-arm geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center, table translation) into camera position, look-at point, and up vector in RAS coordinates. It explicitly models the C-arm's mechanical degrees of freedom (AP and lateral tilts) relative to a patient-centered coordinate system to generate synthetic Digital Radiography (DRR) views. Unlike sibling methods that handle isolated rotations or translations, this method composes them into a complete C-arm pose transformation pipeline [1720-rotatey.md][2279-rotation-transformation.md][2280-rotation-matrix-computation.md][2281-translation-transformation.md][3164-camera-matrix-decomposition.md].

### [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)
Decomposes a 3×4 projection matrix into an intrinsic matrix $K$, a rotation matrix $R$, and a translation vector $t$. It separates the camera's internal optical properties from its external physical pose.

### [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md)
This method converts 3D points and vectors into homogeneous coordinates and applies a rigid body transformation (rotation and translation) to map them between coordinate frames, typically from camera space to world space. It builds on [Camera Pose Transformation](1672-camera-pose-transformation.md).

### [CameraMover](8734-cameramover.md)
CameraMover is a utility class for constructing and incrementally updating a 4x4 extrinsic camera transformation matrix ($M_{ext}$) using sequential translations and rotations. It solves the problem of maintaining a consistent camera pose in world space by applying incremental changes to position and orientation, which are then used to generate world-space ray directions for volume rendering. It differs from sibling methods like [RotateY](1720-rotatey.md) or [Translation Transformation](2281-translation-transformation.md) by encapsulating the entire state management and matrix composition logic into a single class that supports both axis-aligned incremental moves and aiming at specific world positions.

#### RotateZ
RotateZ is a geometric transformation primitive that rotates a point or shape around the Z-axis in 3D space [S2, unofficial]. It is similar to RotateX and RotateY but operates on the plane perpendicular to the Z-axis. It takes an angle input representing the rotation magnitude [S2, unofficial].

#### RigidXform
RigidXform is a transformation structure that maps a volume's local cell space to world metres by applying a uniform cell scale, a rotation, and a translation [S5, unofficial]. It is constructed from a Pose by dequantising the quaternion, renormalising it, and rejecting if the orientation is zero [S5, unofficial]. The stored rotation is normalised, ensuring the conjugate is the exact inverse of the rotation [S5, unofficial]. A key parameter is `cell_size_m`, the uniform scale factor converting local cells to world metres [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for transforming $N$ camera poses, as the operation is a constant-time matrix multiplication per pose [S3, unofficial].
*   **Memory Complexity:** $O(N)$ to store the batch of $N$ camera matrices [S3, unofficial].
*   **Performance:** The method is computationally efficient, leveraging vectorized operations like `einsum` for batch processing [S3, unofficial].
*   **Strengths:** Preserves the rigid body nature of camera poses when normalized; handles batch processing efficiently; integrates well with Lie group formulations for pose optimization [S1][S3, unofficial].
*   **Limitations:** Requires careful normalization to avoid scaling artifacts; assumes the input transformation matrix is valid [S3, S5, unofficial].
*   **When to use:** When adjusting a set of pre-existing camera poses (e.g., from SfM) to a new coordinate system or correcting for known offsets [S3, S4, unofficial].
*   **When not to use:** When constructing a camera pose from scratch using look-at vectors (use [View Matrix Construction](1644-view-matrix-construction.md)) or when decomposing a projection matrix (use [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)).

## Background
Camera poses in computer vision and graphics are often described by the Lie group $SE(3)$ [S1]. This representation allows for the composition of rotations and translations in a mathematically rigorous way. The method of applying transformations to batches of camera matrices is common in neural radiance field (NeRF) optimization, where noisy initial poses are refined or adjusted [S3, unofficial]. The specific offsetting technique (e.g., $[0, -1, -2.2]$) originates from raymarching distance function demonstrations, where scene origin adjustments are used to create specific visual effects [S4, unofficial]. The normalization step ensures that the transformation remains within the special Euclidean group, preventing the introduction of non-rigid scaling during optimization or transformation pipelines [S3, unofficial].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S2] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S3] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S4] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S5] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
