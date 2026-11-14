# Camera Pose Transformation

Also called: Camera Offset and Rotation, Camera Pose Representation, Pose Transformation, RotateX

## Overview
Camera Pose Transformation applies rigid body transformations—specifically rotation and translation—to camera coordinates or scene points to define the camera's position and orientation in 3D space. It solves the problem of mapping between different coordinate systems (e.g., world-to-camera vs. camera-to-world) and adjusting camera parameters for specific rendering pipelines. It differs from [View Matrix Construction](1644-view-matrix-construction.md) by focusing on the algebraic manipulation of pose components (rotation matrices and translation vectors) rather than constructing the final view matrix from basis vectors. It differs from [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) by often operating on specific subsets of the transformation (e.g., rotation only or translation only) or handling format conversions between pose representations.

## How it works
The method encompasses several distinct operations depending on the context: basic axis-aligned rotation, coordinate offsetting, and format conversion between world-to-camera and camera-to-world representations.

### Inputs and Outputs
- **Inputs**: 
  - A point $p \in \mathbb{R}^3$ or a pose matrix $P \in \mathbb{R}^{3 \times 4}$ [S1][S3, unofficial].
  - Rotation angle $\theta \in \mathbb{R}$ [S2, unofficial].
  - Translation vector $t \in \mathbb{R}^3$ [S1][S3, unofficial].
  - Rotation matrix $R \in SO(3)$ [S1, S5].
- **Outputs**: 
  - Transformed point $p' \in \mathbb{R}^3$ [S3, unofficial].
  - Transformed pose $P' \in \mathbb{R}^{3 \times 4}$ [S1, S5].

### Data Structures
- **Pose Representation**: Camera poses are described by the Lie group $SE(3)$, defined as:
  $$ SE(3) = \left\{ \begin{bmatrix} R & t \\ 0^T & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4} \mid R \in SO(3), t \in \mathbb{R}^3 \right\} $$
  where $R$ is the rotation matrix and $t$ is the translation vector [S1].
- **Compact Pose**: The last row of the Lie group is dropped, so camera poses $P$ are represented as:
  $$ P = \{ [R_p \ t_p] \in \mathbb{R}^{3 \times 4} \mid R_p \in SO(3), t_p \in \mathbb{R}^3 \} $$
  [S1].

### Steps and Formulas

#### 1. Axis-Aligned Rotation (RotateX/Y/Z)
Rotates a shape or point around a specific axis (e.g., x-axis) in the plane perpendicular to that axis [S2, unofficial].

**Steps**:
1. Take input point $p = [x, y, z]^T$ and rotation angle $\theta$ [S2, unofficial].
2. Apply the 2D rotation matrix to the components perpendicular to the rotation axis. For rotation around the Y-axis by angle $\theta$ (as seen in [S3, unofficial]):
   $$ y' = \cos(\theta) y + \sin(\theta) z $$
   $$ z' = -\sin(\theta) y + \cos(\theta) z $$
   $$ x' = x $$
   [S3, unofficial].
3. Return transformed point $p'$ [S3, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rotate_y(p, theta):
    """
    Rotates point p around the Y-axis by angle theta.
    p: list or array of 3 floats [x, y, z]
    theta: float, rotation angle in radians
    """
    x, y, z = p
    cos_t = math.cos(theta)
    sin_t = math.sin(theta)
    
    # Apply rotation to y and z components
    y_new = cos_t * y + sin_t * z
    z_new = -sin_t * y + cos_t * z
    
    return [x, y_new, z_new]
```

#### 2. Translation / Offset
Offsets a point by a constant vector to position the camera or scene origin [S3, unofficial].

**Steps**:
1. Take input point $p$ and offset vector $o$ [S3, unofficial].
2. Compute $p' = p + o$ [S3, unofficial].
3. Return $p'$ [S3, unofficial].

**Example**: Offset by $[0, -1, -2.2]$ [S3, unofficial].

#### 3. World-to-Camera to Camera-to-World Conversion
Converts COLMAP's world-to-camera pose parameters into the camera-to-world transformation matrix required by formats like LLFF [S5].

**Steps**:
1. Input world-to-camera rotation $R$ and translation $t$ [S5].
2. Compute the camera-to-world translation $t'$ using the transpose of the rotation matrix $R$ (since $R^{-1} = R^T$ for orthogonal matrices):
   $$ t' = -R^T \cdot t $$
   [S5].
3. Construct the camera-to-world matrix $M$:
   $$ M = \begin{bmatrix} R^T & t' \\ 0 & 1 \end{bmatrix} $$
   Note: The rotation part of the camera-to-world matrix is $R^T$ [S5].
4. Return $M$ [S5].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_world_to_camera_to_camera_to_world(R_wc, t_wc):
    """
    Converts world-to-camera pose (R_wc, t_wc) to camera-to-world pose.
    R_wc: 3x3 numpy array, rotation matrix from world to camera.
    t_wc: 3x1 numpy array, translation vector from world to camera.
    Returns:
    R_cw: 3x3 numpy array, rotation matrix from camera to world.
    t_cw: 3x1 numpy array, translation vector from camera to world.
    """
    # Inverse of rotation is its transpose
    R_cw = R_wc.T
    
    # Translation conversion: t_cw = -R_cw * t_wc
    # Derived from t' = -R^T * t in source [S5]
    t_cw = -np.dot(R_cw, t_wc)
    
    return R_cw, t_cw
```

### Parameters
- **angle**: A float input representing the angle of rotation in radians [S2, unofficial].
- **offset**: A 3D vector $[x, y, z]$ for translation [S3, unofficial].
- **noise coefficient**: In noisy pose optimization, a scalar (e.g., 0.35) used to generate 6-dimensional random normal distribution noise in the Lie algebra $se(3)$ [S1].

### Edge Cases and Pitfalls
- **Precision**: When composing poses with noise, the noise $n \in se(3)$ is generated as $n = \sigma N(0, I) \in \mathbb{R}^6$ [S1]. This corresponds to an average deviation of $30.4^\circ$ in rotation and $0.56$ in translation for $\sigma=0.35$ [S1].
- **Cheating with Offsets**: Offsetting points in distance functions (like in [S3, unofficial]) can make the distance function mathematically incorrect for strict raymarching, requiring careful handling [S3, unofficial].
- **Matrix Inversion**: The conversion relies on $R$ being orthogonal, such that $R^{-1} = R^T$. If $R$ is not orthogonal (e.g., due to numerical error), the conversion may be inaccurate [S5].

## Subtopics

### [Pose Recentering](1786-pose-recentering.md)
Pose Recentering transforms a set of camera poses so that their average pose becomes the identity transformation, effectively centering the scene at the origin and aligning the coordinate axes.

### [Rotation Transformation](2279-rotation-transformation.md)
Rotation Transformation constructs a 4x4 homogeneous rotation matrix to rotate points or shapes around a specified axis by a given angle. It serves as a fundamental building block for constructing camera poses and deforming Signed Distance Fields (SDFs) [G]. Unlike [Quaternion Rotation](2302-quaternion-rotation.md), which operates in 3D space with unit quaternions to avoid gimbal lock, this method explicitly constructs orthogonal matrices for single axes or composes them sequentially [G]. Unlike [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md), it handles orientation only, leaving translation to sibling methods like [Translation Transformation](2281-translation-transformation.md).

### [Translation Transformation](2281-translation-transformation.md)
A geometric transformation that moves a shape by a given vector. It adjusts positional offsets without altering orientation, distinguishing it from rotation transformations [G].

### [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md)
This method converts C-arm X-ray geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center point, and table translation) into a camera pose defined by position, look-at point, and up vector in RAS (Right-Anterior-Superior) coordinates. It solves the problem of generating physically plausible camera matrices for digital tomosynthesis or projection rendering from high-level interventional radiology acquisition settings. It differs from generic [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) by explicitly modeling the kinematic constraints of a C-arm gantry (rotation around specific axes relative to the patient table) rather than applying arbitrary SE3 transformations.

### [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)
Decomposes a 3×4 projection matrix into an intrinsic matrix $K$, a rotation matrix $R$, and a translation vector $t$. It separates the camera's internal optical properties from its external physical pose.

### [CameraMover](8734-cameramover.md)
CameraMover is a utility class for constructing and incrementally updating a 4x4 extrinsic camera transformation matrix ($M_{ext}$) using sequential translations and rotations. It solves the problem of maintaining a consistent camera pose in world space by applying incremental changes to position and orientation, which are then used to generate world-space ray directions for volume rendering. It differs from sibling methods like [RotateY](1720-rotatey.md) or [Translation Transformation](2281-translation-transformation.md) by encapsulating the entire state management and matrix composition logic into a single class that supports both axis-aligned incremental moves and aiming at specific world positions.

### [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md)
Rigid Body Transformation (SE3) applies a rotation and translation to 3D points or vectors without altering their relative distances or angles, preserving the shape of the object. It solves the problem of unifying rotation and translation into a single linear operation, which is essential for composing complex camera poses and scene transforms in volume rendering. It differs from [Rotation Transformation](2279-rotation-transformation.md) and [Translation Transformation](2281-translation-transformation.md) by combining both operations into a single $4 \times 4$ matrix multiplication rather than separating them into sequential vector additions and rotations.

### [Camera-to-Object Space Transformation & MVP Computation](12391-camera-to-object-space-transformation-mvp-computation.md)
Precomputes model-view-projection (MVP) matrix and transforms camera direction/position into object space for use in shaders, supporting both perspective and orthographic projections.

### Continuous Camera Motion Modeling
A method to model camera motion as continuous time-dependent angular velocity $\omega(t)$ and linear velocity $v(t)$ instead of discrete camera-to-world poses, enabling the handling of large motions by integrating relative motions over time [S4].
- **Steps**:
  1. Predict angular velocity $\omega(t)$ and linear velocity $v(t)$ at time $t$ using an MLP $\phi_v$: $\omega(t), v(t) = \phi_v(t)$ [S4].
  2. Obtain the camera transformation $P$ between any two frames by integrating the learned continuous camera motion ($\omega$ and $v$) over time [S4].
- **Formula**: The relationship between camera transformation (rotation $R$, position $t$) and velocities is defined by the differential equations:
  $$ \frac{dR}{dt} = [\omega(t)]_\times R $$
  $$ \frac{dt}{dt} = v(t) $$
  where $[\omega(t)]_\times \in so(3)$ is the skew-symmetric matrix of $\omega(t)$ [S4].
- **Parameters**: $\omega(t), v(t) \in \mathbb{R}^3$: The angular velocity and linear velocity vectors at time $t$ [S4].

### LLFF Camera Pose Conversion
A procedure to convert COLMAP's world-to-camera pose parameters into the camera-to-world transformation matrix required by the LLFF dataset format [S5].
- **Formula**: The translation vector in world-to-camera format $t$ is converted to camera-to-world format $t'$ using the transpose of the rotation matrix $R$: $t' = -R^T \cdot t$ [S5].
- **Layout**: LLFF dataset components: (1) Image sequences, (2) Camera intrinsic parameters, (3) Camera extrinsic parameters (rotation matrix $R^T$ and translation vector $t'$) [S5].

## Operating characteristics
- **Time Complexity**: Basic rotation and translation are $O(1)$ per point. Matrix conversions are $O(1)$ for fixed-size $3 \times 3$ or $3 \times 4$ matrices. Continuous motion modeling requires integration steps, scaling with the number of time steps [S4].
- **Memory Complexity**: $O(1)$ for storing a single pose ($3 \times 4$ matrix) [S1].
- **Strengths**: Simple and efficient for standard pose adjustments. The Lie group formulation provides a robust mathematical framework for handling noisy poses [S1].
- **Limitations**: Discrete pose transformations may struggle with large motions in video sequences compared to continuous modeling [S4]. Offset-based transformations in SDFs can break distance function properties [S3, unofficial].
- **When to use**: Use for standard camera positioning, coordinate system conversions (e.g., COLMAP to LLFF), and basic scene transformations. Use continuous modeling for video-based NeRF with significant camera motion [S4].

## Background
The concept of camera poses as elements of the Lie group $SE(3)$ is widely used in computer vision and robotics for representing rigid body transformations [S1]. The specific application of transforming poses for NeRF training, such as handling noisy poses (CBARF) [S1] or converting between dataset formats like LLFF [S5], has become standard in neural rendering pipelines. The use of continuous camera motion modeling for video NeRF was introduced to handle large motions that discrete pose estimation struggles with [S4]. Basic rotation and translation operations are fundamental to raymarching and SDF rendering, as demonstrated in early interactive raymarching examples [S2, S3, unofficial].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S2] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S3] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S4] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
[S5] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
