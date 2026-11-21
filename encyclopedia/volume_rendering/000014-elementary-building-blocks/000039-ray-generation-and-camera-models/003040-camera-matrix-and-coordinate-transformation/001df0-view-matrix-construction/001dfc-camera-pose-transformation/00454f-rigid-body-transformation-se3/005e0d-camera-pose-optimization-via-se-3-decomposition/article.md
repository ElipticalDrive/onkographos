# Camera Pose Optimization via SE(3) Decomposition

Also called: Non-parametric SE(3) Optimization, SE(3) Decomposition for Optimization

## Overview
This method optimizes camera poses, represented as $SE(3)$ transformations, during the training of generative radiance fields by decomposing the transformation matrix into manageable components. It solves the problem of manifold escape, where direct optimization of rotation matrices causes them to violate orthogonality constraints ($RR^T \neq I$) [S1]. Unlike standard parametric approaches that optimize Euler angles directly, this method parameterizes rotations using sine and cosine pairs, ensuring valid rotation matrices through normalization [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of initial camera poses $T = \{T_1, \dots, T_N\}$ where $T_i \in SE(3)$ [S1].
*   **Output:** Optimized camera poses $T$ after backpropagating adversarial losses [S1].

**Data Structures**
The transformation $T$ is decomposed into a rotation $R \in SO(3)$ and a translation $p \in \mathbb{R}^3$:
$$
T = \begin{pmatrix} R & p \\ 0 & 1 \end{pmatrix}
$$
The rotation $R$ is decomposed into Z-Y-X Euler angle rotations:
$$
R = R_z R_y R_x
$$
where $R_{axis} \in SO(3)$ [S1].

Instead of optimizing angles $\theta$, the method optimizes the sine and cosine components directly. The data structures carried for optimization are:
1.  $\mathcal{R}_z = [\cos \theta_z, \sin \theta_z]$
2.  $\mathcal{R}_y = [\cos \theta_y, \sin \theta_y]$
3.  $\mathcal{R}_x = [\cos \theta_x, \sin \theta_x]$
4.  $p = [p_x, p_y, p_z]$

The full transformation data structure is $[\mathcal{R}_z, \mathcal{R}_y, \mathcal{R}_x, p]$ [S1].

**Formulas**
The rotation matrix around the z-axis is defined as:
$$
R_z = \begin{pmatrix} \cos \theta_z & -\sin \theta_z & 0 \\ \sin \theta_z & \cos \theta_z & 0 \\ 0 & 0 & 1 \end{pmatrix}
$$
Similar definitions apply for $R_y$ and $R_x$ using their respective sine and cosine values [S1].

**Steps**
1.  Initialize the data structures $\mathcal{R}_z, \mathcal{R}_y, \mathcal{R}_x$ and $p$ from initial camera poses [S1].
2.  During training, randomly sample camera poses from the set $T$ [S1].
3.  Construct the rotation matrix $R$ by multiplying $R_z, R_y, R_x$ using the current sine/cosine values [S1].
4.  Construct the full transformation matrix $T$ using $R$ and $p$ [S1].
5.  Render the scene from the sampled camera poses [S1].
6.  Compute the adversarial loss [S1].
7.  Backpropagate gradients to update $\mathcal{R}_z, \mathcal{R}_y, \mathcal{R}_x$ and $p$ [S1].
8.  After each update to any rotation parameter $\mathcal{R}_{axis}$, normalize the vector so that $\|\mathcal{R}_{axis}\| = 1$ to ensure valid rotation matrices [S1].

**Parameters**
*   **Optimization Scope:** In early training stages (when expected patch scale > 0.5), only $p_x, p_z$ and $R_y$ are optimized, assuming cameras are at the same height and rotate along the vertical axis [S1].

**Edge Cases and Pitfalls**
*   **Manifold Escape:** Directly optimizing the full $4 \times 4$ matrix can cause $RR^T \neq I$ [S1].
*   **Discontinuity:** Directly optimizing Euler angles $\theta$ is difficult due to discontinuity at $\theta = 0$ [S1].
*   **Normalization:** Failure to normalize $\mathcal{R}_{axis}$ after updates results in invalid rotation matrices [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize_cos_sin(cos_val, sin_val):
    """Normalize the cosine and sine vector to unit norm."""
    norm = math.sqrt(cos_val**2 + sin_val**2)
    if norm == 0:
        return 1.0, 0.0
    return cos_val / norm, sin_val / norm

def get_rotation_z(cos_z, sin_z):
    """Construct Rz matrix from cos and sin."""
    return [
        [cos_z, -sin_z, 0],
        [sin_z, cos_z, 0],
        [0, 0, 1]
    ]

def get_rotation_y(cos_y, sin_y):
    """Construct Ry matrix from cos and sin."""
    return [
        [cos_y, 0, sin_y],
        [0, 1, 0],
        [-sin_y, 0, cos_y]
    ]

def get_rotation_x(cos_x, sin_x):
    """Construct Rx matrix from cos and sin."""
    return [
        [1, 0, 0],
        [0, cos_x, -sin_x],
        [0, sin_x, cos_x]
    ]

def multiply_matrices(A, B):
    """Multiply two 3x3 matrices."""
    C = [[0, 0, 0], [0, 0, 0], [0, 0, 0]]
    for i in range(3):
        for j in range(3):
            for k in range(3):
                C[i][j] += A[i][k] * B[k][j]
    return C

def update_pose(cos_z, sin_z, cos_y, sin_y, cos_x, sin_x, p):
    """
    Update pose and return new components.
    This function simulates one optimization step.
    """
    # 1. Normalize rotation components
    cos_z, sin_z = normalize_cos_sin(cos_z, sin_z)
    cos_y, sin_y = normalize_cos_sin(cos_y, sin_y)
    cos_x, sin_x = normalize_cos_sin(cos_x, sin_x)
    
    # 2. Construct rotation matrices
    Rz = get_rotation_z(cos_z, sin_z)
    Ry = get_rotation_y(cos_y, sin_y)
    Rx = get_rotation_x(cos_x, sin_x)
    
    # 3. Combine rotations: R = Rz * Ry * Rx
    R_zy = multiply_matrices(Rz, Ry)
    R = multiply_matrices(R_zy, Rx)
    
    # 4. Construct full transformation T (4x4)
    # T = [[R, p], [0, 0, 0, 1]]
    T = [
        [R[0][0], R[0][1], R[0][2], p[0]],
        [R[1][0], R[1][1], R[1][2], p[1]],
        [R[2][0], R[2][1], R[2][2], p[2]],
        [0, 0, 0, 1]
    ]
    
    # 5. Render and compute loss (Placeholder for external process)
    # loss = render_and_compute_adversarial_loss(T)
    
    # 6. Backpropagate gradients (Placeholder for external process)
    # gradients = compute_gradients(loss, [cos_z, sin_z, cos_y, sin_y, cos_x, sin_x, p])
    
    # 7. Update parameters (Example update rule)
    # Note: In practice, gradients are applied via an optimizer like Adam.
    # Here we assume 'gradients' are provided by the external loop.
    # For the purpose of this self-contained code, we just return the valid T.
    
    return T, (cos_z, sin_z, cos_y, sin_y, cos_x, sin_x, p)
```

## Subtopics

### Camera Coordinate Homogenization and Rigid Transformation
Handles the conversion of camera coordinates to homogeneous space and applies rigid transformations. Choose this when setting up the initial coordinate system before optimization. [Link](4005-camera-coordinate-homogenization-and-rigid-transformation.md)

### Scene-Related Transforms
Manages transformations specific to the scene geometry rather than the camera. Choose this when adjusting scene elements independently of camera pose. [Link](11646-scene-related-transforms.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is used in SinGRAF with a spatial resolution of $256 \times 256$ and feature channel $C=32$ [S1].
*   **Strengths:** Avoids manifold escape issues ($RR^T \neq I$) and discontinuities associated with direct Euler angle optimization [S1].
*   **Limitations:** Requires careful normalization after every update [S1].
*   **When to use:** When training generative radiance fields where camera poses are optimized via adversarial losses [S1].
*   **When not:** When high-precision continuous rotation is required without discrete normalization steps, or when computational resources for backpropagation are extremely limited.

## Background
The method originates from the SinGRAF paper, which introduces a non-parametric optimization of camera poses during training [S1]. It was developed to address the difficulties of optimizing $SE(3)$ transformations directly, specifically the risk of the rotation matrix escaping the $SO(3)$ manifold [S1]. The approach uses a decomposition into sine/cosine components to facilitate stable gradient-based optimization [S1].

## Sources
[S1] SinGRAF: Learning a 3D Generative Radiance Field for a Single Scene, https://doi.org/10.1109/cvpr52729.2023.00822, fetched 2026-10-29
