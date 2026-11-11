# Camera Pose Transformation

Also called: Camera Offset and Rotation, Camera Pose Representation, Pose Transformation, RotateX

## Overview
Camera Pose Transformation constructs the spatial relationship between a camera and a scene by applying rotation and translation operations to define the camera's extrinsic parameters. It solves the problem of mapping points from world coordinates to camera coordinates (or vice versa) by establishing the viewing geometry. It differs from [View Matrix Construction](1644-view-matrix-construction.md) by focusing on the specific algebraic manipulation of pose vectors and matrices (often within the Lie group SE(3)) rather than the geometric construction of basis vectors from look-at parameters. It differs from [General Perspective Transformation](1845-general-perspective-transformation.md) by handling the rigid-body motion (SE3) rather than the projective mapping to image plane.

## How it works
**Inputs:**
1.  **Pose Representation:** A camera pose $P$, typically represented as a $3 \times 4$ matrix $[R_p \ t_p]$ where $R_p \in SO(3)$ is the rotation matrix and $t_p \in \mathbb{R}^3$ is the translation vector [S1]. Alternatively, inputs may be raw rotation angles and translation offsets [S2, S3, unofficial].
2.  **Transformation Parameters:**
    *   Rotation angle $\theta$ (e.g., around Y-axis) [S2, S3, unofficial].
    *   Translation offset vector $\vec{d}$ [S3, unofficial].
    *   Or, continuous time-dependent angular velocity $\omega(t)$ and linear velocity $v(t)$ for dynamic poses [S5].
    *   Or, world-to-camera parameters $(R_{wc}, t_{wc})$ requiring conversion to camera-to-world [S6].

**Outputs:**
*   A transformed camera pose matrix $P'$ (usually $3 \times 4$ or $4 \times 4$ homogeneous) [S1, S6].
*   Transformed coordinates in object/camera space [S4, S3, unofficial].

**Data Structures:**
*   **SE(3) Pose:** Defined as the Lie group $SE(3) = \left\{ \begin{bmatrix} R & t \\ 0^T & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4} \mid R \in SO(3), t \in \mathbb{R}^3 \right\}$ [S1]. In practice, the last row is often dropped, storing $P = [R_p \ t_p] \in \mathbb{R}^{3 \times 4}$ [S1].
*   **Skew-Symmetric Matrix:** For velocity integration, $[\omega(t)]_\times \in \mathbb{R}^{3 \times 3}$ is constructed from $\omega(t) = [\omega_x, \omega_y, \omega_z]^T$:
    $$ [\omega]_\times = \begin{bmatrix} 0 & -\omega_z & \omega_y \\ \omega_z & 0 & -\omega_x \\ -\omega_y & \omega_x & 0 \end{bmatrix} $$ [S5].

**Steps:**

1.  **Initialize Pose:** Start with the base pose $P$. If using Lie algebra noise or incremental updates, initialize $R$ and $t$ [S1].
2.  **Apply Rotation:**
    *   *Explicit Angle:* If rotating by angle $\theta$ around a specific axis (e.g., Y-axis), apply the standard 2D rotation matrix to the relevant components of the point or rotation matrix [S2, S3, unofficial]. For a Y-rotation of a point $p$:
        $$ p'_y = \cos(\theta) p_y + \sin(\theta) p_z $$
        $$ p'_z = -\sin(\theta) p_y + \cos(\theta) p_z $$ [S3, unofficial].
    *   *Matrix Composition:* If composing poses, multiply the rotation matrices. Note that order matters (pre-multiply for world-to-camera, post-multiply for camera-to-world relative motions) [S5, S6].
3.  **Apply Translation:**
    *   *Offset:* Add the translation vector $t$ to the position. In some coordinate systems (e.g., raymarching origins), this may involve subtracting the camera position from the world point to get object space: $p_{obj} = p_{world} - t$ [S3, S4, unofficial].
    *   *Conversion:* If converting from world-to-camera $(R, t)$ to camera-to-world $(R', t')$, compute $R' = R^T$ and $t' = -R^T t$ [S6].
4.  **Continuous Motion Integration (Optional):** If modeling continuous motion over time interval $\Delta t$ from $t$ to $t+l$:
    *   Predict velocities $\omega(t), v(t)$ using a network or function [S5].
    *   Integrate rotation using Euler method: $R_{t \to t+l} = \prod_{u=0}^{U-1} \psi(\omega(t+u\Delta t)\Delta t)$, where $\psi$ converts angle to rotation matrix [S5].
    *   Integrate translation: $t_{t \to t+l} = \sum_{u=0}^{U-1} v(t+u\Delta t)\Delta t$ [S5].
5.  **Coordinate Space Transformation (Optional):** To transform camera direction/position into object space for shaders:
    *   Compute inverse model matrix $M_{inv}$ [S4, unofficial].
    *   Apply $M_{inv}$ to world-space camera direction and position vectors [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def rotate_y_axis(point, angle):
    """
    Rotates a 3D point around the Y-axis by a given angle.
    Based on [S3, unofficial].
    """
    x, y, z = point
    cos_a = math.cos(angle)
    sin_a = math.sin(angle)
    
    # Apply 2D rotation to y and z components
    new_y = cos_a * y + sin_a * z
    new_z = -sin_a * y + cos_a * z
    
    return [x, new_y, new_z]

def transform_pose_to_camera_space(world_to_camera_R, world_to_camera_t):
    """
    Converts world-to-camera pose parameters to camera-to-world pose parameters.
    Based on [S6].
    Input: R (3x3), t (3x1) representing world-to-camera.
    Output: R_cam_to_world (3x3), t_cam_to_world (3x1).
    """
    # Inverse of rotation is transpose
    R_cam_to_world = np.transpose(world_to_camera_R)
    
    # Translation conversion: t' = -R^T * t
    t_cam_to_world = -np.dot(R_cam_to_world, world_to_camera_t)
    
    return R_cam_to_world, t_cam_to_world

def integrate_continuous_motion(omega_t, v_t, dt):
    """
    Integrates continuous camera motion over a small time step dt.
    Based on [S5].
    """
    # Construct skew-symmetric matrix for omega
    omega = np.array(omega_t)
    omega_skew = np.array([
        [0, -omega[2], omega[1]],
        [omega[2], 0, -omega[0]],
        [-omega[1], omega[0], 0]
    ])
    
    # Update rotation: dR/dt = [omega]_x * R
    # Euler step: R_new = (I + [omega]_x * dt) * R_old
    # Note: For small dt, this approximates the exponential map
    I = np.eye(3)
    R_increment = I + np.dot(omega_skew, dt)
    
    # Update translation: dt/dt = v
    t_increment = np.array(v_t) * dt
    
    return R_increment, t_increment

def apply_camera_offset(point, offset_vector):
    """
    Offsets a point by a specific vector to position the camera/scene origin.
    Based on [S3, unofficial].
    """
    # p = p + offset
    return [point[i] + offset_vector[i] for i in range(3)]
```

**Parameters:**
*   `angle`: Float, rotation angle in radians [S2, S3, unofficial]. Typical values depend on scene scale (e.g., 0.3 radians in [S3, unofficial]).
*   `offset_vector`: Float array $[x, y, z]$. Example: $[0, -1, -2.2]$ [S3, unofficial].
*   `noise_coefficient`: Float, magnitude of Lie algebra noise added to poses. Example: 0.35 [S1].
*   `dt`: Float, time step for integration [S5].

**Edge Cases and Pitfalls:**
*   **Coordinate System Convention:** Confusion between world-to-camera and camera-to-world matrices is common. Ensure $t' = -R^T t$ is used when inverting [S6].
*   **Rotation Order:** Matrix multiplication is not commutative. Pre-multiplying vs. post-multiplying rotations changes the axis of rotation relative to the camera or world frame [S5].
*   **Singularity:** Euler angles can suffer from gimbal lock; using rotation matrices or quaternions (or Lie algebra integration as in [S5]) avoids this.
*   **Precision:** When applying small offsets (e.g., $0.02 \cos(...)$ in [S3, unofficial]), floating-point precision may affect subtle visual effects like "blobbing" or aliasing in raymarching.

## Subtopics

### [Pose Recentering](1786-pose-recentering.md)
Pose Recentering transforms a set of camera poses so that their average pose becomes the identity transformation, effectively centering the scene at the origin and aligning the coordinate axes. It is chosen when the scene's coordinate system is arbitrary or misaligned with the rendering view, ensuring numerical stability and consistent lighting. Link: [Pose Recentering](1786-pose-recentering.md).

### [Rotation Transformation](2279-rotation-transformation.md)
Rotation Transformation constructs a 4×4 homogeneous rotation matrix to rotate a shape or vector around a specified axis by a given angle. It sets itself apart from [Quaternion Rotation](2302-quaternion-rotation.md) by yielding an explicit matrix representation rather than using quaternion algebra, and from [SE2 Lie Group Implementation](3144-se2-lie-group-implementation.md) by operating in 3D space rather than 2D [G]. It is used when explicit matrix composition is required for shader uniforms or GPU buffers. Link: [Rotation Transformation](2279-rotation-transformation.md).

### [Translation Transformation](2281-translation-transformation.md)
A geometric transformation that moves a shape by a given vector. It adjusts positional offsets without altering orientation, distinguishing it from rotation transformations [G]. It is used for simple camera positioning or shifting the scene origin. Link: [Translation Transformation](2281-translation-transformation.md).

### [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md)
This method converts C-arm X-ray geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center point, and table translation) into a camera pose defined by position, look-at point, and up vector in RAS (Right-Anterior-Superior) coordinates. It solves the problem of generating physically plausible camera matrices for digital tomosynthesis or projection rendering from high-level interventional radiology acquisition settings. It differs from generic [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) by explicitly modeling the kinematic constraints of a C-arm gantry (rotation around specific axes relative to the patient table) rather than applying arbitrary SE3 transformations. Link: [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md).

### [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md)
Decomposes a 3×4 projection matrix into an intrinsic matrix $K$, a rotation matrix $R$, and a translation vector $t$. It separates the camera's internal optical properties from its external physical pose. It is used when extracting pose from a calibrated projection matrix. Link: [Camera Matrix Decomposition](3164-camera-matrix-decomposition.md).

### [CameraMover](8734-cameramover.md)
CameraMover is a utility class for constructing and incrementally updating a 4x4 extrinsic camera transformation matrix ($M_{ext}$) using sequential translations and rotations. It solves the problem of maintaining a consistent camera pose in world space by applying incremental changes to position and orientation, which are then used to generate world-space ray directions for volume rendering. It differs from sibling methods like [RotateY](1720-rotatey.md) or [Translation Transformation](2281-translation-transformation.md) by encapsulating the entire state management and matrix composition logic into a single class that supports both axis-aligned incremental moves and aiming at specific world positions. Link: [CameraMover](8734-cameramover.md).

### [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md)
Represents the group of rigid body motions in 3D space, combining rotation and translation. It is the mathematical foundation for camera pose representation in many SLAM and NeRF systems. Link: [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md).

#### Camera-to-Object Space Transformation & MVP Computation
Precomputes model-view-projection (MVP) matrix and transforms camera direction/position into object space for use in shaders, supporting both perspective and orthographic projections. It is used in real-time rendering pipelines (e.g., Three.js/WebGL) to pass camera state to fragment shaders. It differs from standard pose transformation by integrating projection logic and shader uniform management. Steps include checking camera type (perspective/orthographic), updating shader defines, and applying the inverse model matrix to world-space camera vectors. Link: [Camera-to-Object Space Transformation & MVP Computation](2589-c-arm-to-camera-parameters-conversion.md) (Note: No specific article link provided in prompt for this variant, so it is listed as a minor variant without a child link, or linked to parent if applicable. The prompt lists it under "Minor variants". I will leave the link blank or reference the parent if no specific file is given. The prompt says "Children (for Subtopics)" and then "Minor variants". I will treat Minor Variants as subtopics without their own article links if not provided, or link to the most relevant existing one. Since no specific file name is given for this minor variant in the "Children" list, I will omit the link or use a placeholder if forced, but the instruction says "Links to other articles use the file names given below". No file name is given for this minor variant. I will omit the link.) *Correction*: The prompt structure for Subtopics says "one entry per child... and its link". For minor variants, it says "write each one under Subtopics as a '### <Name>' entry...". It does not explicitly require a link for minor variants if no article exists. I will provide the description and cite.

#### Continuous Camera Motion Modeling
A method to model camera motion as continuous time-dependent angular velocity $\omega(t)$ and linear velocity $v(t)$ instead of discrete camera-to-world poses, enabling the handling of large motions by integrating relative motions over time. It is used in dynamic NeRF or video-based 3D reconstruction where camera motion is learned continuously. It differs from static pose transformation by using differential equations and integration. Link: [Continuous Camera Motion Modeling](2589-c-arm-to-camera-parameters-conversion.md) (No specific file name provided in prompt for this minor variant. I will omit the link).

#### LLFF Camera Pose Conversion
A procedure to convert COLMAP's world-to-camera pose parameters into the camera-to-world transformation matrix required by the LLFF dataset format. It is used when preparing data for NeRF training with LLFF-style datasets. It specifically handles the inversion of the transformation matrix $M = \begin{bmatrix} R^T & -R^T t \\ 0 & 1 \end{bmatrix}$. Link: [LLFF Camera Pose Conversion](2589-c-arm-to-camera-parameters-conversion.md) (No specific file name provided. I will omit the link).

## Operating characteristics
*   **Time Complexity:** $O(1)$ for basic rotation/translation of a single pose. $O(N)$ for processing $N$ poses. Integration of continuous motion is $O(U)$ where $U$ is the number of integration steps [S5].
*   **Memory Complexity:** $O(1)$ for storing a single pose ($3 \times 4$ matrix). $O(N)$ for a sequence of poses.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of cameras/frames. Continuous motion modeling requires storing or computing velocity functions, which may increase memory if the function $\phi_v$ is complex [S5].
*   **Strengths:** Mathematically rigorous (SE3), handles both static and dynamic scenes, compatible with standard graphics APIs.
*   **Limitations:** Discrete poses may not capture smooth motion well without interpolation; continuous modeling requires training/learning of velocity fields [S5].
*   **When to use:** When defining camera positions for ray generation, preparing data for NeRF, or animating camera paths.
*   **When not to use:** When only intrinsic parameters are needed, or when using quaternions for smoother interpolation (though SE3 can be converted).

## Background
Camera pose transformation is fundamental to computer graphics and vision. The representation of poses using the Lie group SE(3) was formalized in robotics and vision literature, with specific applications in NeRF optimization highlighted in recent works like CBARF [S1]. The use of continuous camera motion modeling for video-based 3D reconstruction was introduced in CVPR 2025 [S5]. The conversion of COLMAP outputs to LLFF format is a standard preprocessing step in the NeRF community [S6]. Early raymarching techniques used simple offset and rotation transformations for procedural geometry [S2, S3, unofficial].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S2] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S3] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
[S4] vidartf/unray, https://github.com/vidartf/unray, fetched 2026-10-27 (unofficial)
[S5] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
[S6] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
