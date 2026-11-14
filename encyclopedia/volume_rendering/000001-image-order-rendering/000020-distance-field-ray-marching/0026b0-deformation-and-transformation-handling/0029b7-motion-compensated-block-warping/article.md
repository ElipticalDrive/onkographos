# Motion-Compensated Block Warping

Also called: Inter-frame motion compensation

## Overview
Motion-Compensated Block Warping is a technique for compressing temporal sequences of Truncated Signed Distance Fields (TSDF) by predicting the current frame's geometry from the previous frame using estimated rigid body motion [S1]. It solves the problem of high residual energy in dynamic scenes where objects move between frames, allowing the encoder to transmit only the motion parameters and the small residual differences rather than the full geometry [S1]. It differs from static deformation handling by explicitly estimating and transmitting a per-block 6-DoF transform to align the previous frame's data with the current frame's target block [S1].

## How it works
The method operates on a block-based sparse TSDF representation [S1]. The scene is decomposed into voxel blocks [S1]. For each block in the target frame $t$, the algorithm estimates a rigid transform that best aligns the reconstructed TSDF from the previous frame $t-1$ to the current target block [S1].

### Inputs and Outputs
*   **Inputs:**
    *   $s_t^{(b)}$: The TSDF values for block $b$ at frame $t$ [S1].
    *   $\hat{s}_{t-1}^{(b)}$: The reconstructed (decoded) TSDF values for block $b$ at frame $t-1$ [S1].
    *   $\Omega$: The set of sample positions (voxel centers) within block $b$ [S1].
*   **Outputs:**
    *   $\mathbf{r}, \mathbf{t}$: The optimal rotation (Euler angles) and translation (3D vector) parameters [S1].
    *   Residual coefficients: The difference between the current block and the warped predictor, encoded in the transform domain (e.g., DCT) [S1].

### Data Structures
*   **TSDF Block:** A 3D grid of voxel values [S1]. In the source implementation, blocks are stored in a GPU-side spatial hash table (voxel hashing) for fast random access [S1].
*   **Motion Parameters:** A 6-DoF rigid transform consisting of translation $\mathbf{t} \in \mathbb{R}^3$ and rotation $\mathbf{r} \in \mathbb{R}^3$ (parameterized as Euler angles) [S1].

### Steps
1.  **Neighborhood Assembly:** Assemble a stitched $3 \times 3 \times 3$ neighborhood of blocks from the previous frame's reconstructed TSDF $\hat{s}_{t-1}$ centered on the corresponding block location [S1]. This ensures that trilinear interpolation remains valid even when warped sample positions fall into adjacent blocks or between voxel centers [S1].
2.  **Motion Estimation:** Estimate the rigid transform $(\mathbf{r}, \mathbf{t})$ that minimizes the squared difference between the warped previous frame and the current frame [S1]. This is done using a damped Gauss-Newton optimization procedure [S1].
3.  **Inverse Warping (Prediction):** For each sample position $\mathbf{p}_i \in \Omega$ in the target block $b$ at frame $t$, compute the corresponding position in the previous frame using the estimated transform:
    $$ \mathbf{q}_i = \mathbf{T}(\mathbf{p}_i; \mathbf{r}, \mathbf{t}) $$
    where $\mathbf{T}$ maps current frame coordinates to previous frame coordinates [S1].
4.  **Trilinear Interpolation:** Evaluate the TSDF value at the warped location $\mathbf{q}_i$ by trilinearly interpolating the values from the $3 \times 3 \times 3$ neighborhood assembled in Step 1 [S1]. This forms the motion-compensated predictor $\hat{s}_{t-1}^{(b)}(\mathbf{T}(\mathbf{p}_i; \mathbf{r}, \mathbf{t}))$ [S1].
5.  **Residual Encoding:** Compute the residual between the current block $s_t^{(b)}$ and the predictor [S1]. Encode the transform-domain coefficients of this residual [S1].
6.  **Side Information Encoding:** Encode the motion parameters $\mathbf{r}$ and $\mathbf{t}$ as side information [S1].

### Formulas
The residual for sample $i$ is defined as [S1]:
$$ \epsilon_i(\mathbf{r}, \mathbf{t}) = \hat{s}_{t-1}^{(b)}(\mathbf{T}(\mathbf{p}_i; \mathbf{r}, \mathbf{t})) - s_t^{(b)}(\mathbf{p}_i) $$

The optimization objective is to minimize the sum of squared residuals [S1]:
$$ \mathbf{t}, \mathbf{r} := \arg \min_{\mathbf{t}, \mathbf{r}} \sum_{i \in \Omega} \epsilon_i(\mathbf{r}, \mathbf{t})^2 $$

The update step in the Gauss-Newton algorithm is calculated as [S1]:
$$ \Delta \mathbf{p} = -(J^\top J + \lambda I)^{-1} J^\top \boldsymbol{\varepsilon} $$
where $J$ is the Jacobian of the residuals with respect to the parameters, $\lambda$ is a damping factor for numerical stability, and $\boldsymbol{\varepsilon}$ is the vector of residuals [S1].

### Parameters
*   **Damping factor $\lambda$:** A small value for numerical stability in the Gauss-Newton solver [S1].
*   **Translation clamp:** 1 voxel per iteration [S1].
*   **Rotation clamp:** 5 degrees per iteration [S1].
*   **Termination threshold:** The optimization terminates when the decrease in the objective function falls below a specific threshold [S1].

### Edge Cases and Pitfalls
*   **Insufficient Data:** If too few reliable samples are available or the $3 \times 3 \times 3$ neighborhood is incomplete (e.g., at the boundary of the volume), the method falls back to zero-motion prediction ($\mathbf{t} = 0, \mathbf{r} = 0$) for that block [S1].
*   **Divergence:** Large motion estimates can cause the Gauss-Newton solver to diverge [S1]. Clamping the per-iteration update magnitudes prevents this [S1].
*   **Fractional Positions:** Rigid motion maps integer voxel centers to fractional positions [S1]. Trilinear interpolation is required to evaluate the TSDF at these non-integer locations [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def motion_compensated_block_warping(target_block_b, prev_frame_reconstructed_tsdf, block_index, block_size):
    # 1. Assemble 3x3x3 neighborhood from previous frame [S1]
    neighborhood = assemble_3x3x3_neighborhood(prev_frame_reconstructed_tsdf, block_index, block_size)
    
    # 2. Estimate motion using damped Gauss-Newton [S1]
    r = np.array([0.0, 0.0, 0.0])  # Euler angles (rad)
    t = np.array([0.0, 0.0, 0.0])  # Translation (voxels)
    damping_lambda = 1e-4
    max_iterations = 10
    convergence_threshold = 1e-6
    prev_error = float('inf')
    
    for iteration in range(max_iterations):
        residuals = []
        jacobian_rows = []
        
        for p_i in get_voxel_centers(block_size):
            # T maps current position to previous frame space for sampling [S1]
            warped_pos = apply_rigid_transform(p_i, r, t)
            pred_val = trilinear_interpolate(neighborhood, warped_pos)
            curr_val = target_block_b[get_index(p_i, block_size)]
            residual = pred_val - curr_val
            residuals.append(residual)
            jacobian_row = compute_jacobian_row(p_i, r, t, neighborhood, block_size)
            jacobian_rows.append(jacobian_row)
            
        residuals = np.array(residuals)
        J = np.array(jacobian_rows)
        
        # Solve for update step: delta_p = -(J^T J + lambda I)^-1 J^T epsilon [S1]
        JTJ = J.T @ J
        JTJ_damped = JTJ + damping_lambda * np.eye(6)
        try:
            delta_p = np.linalg.solve(JTJ_damped, -J.T @ residuals)
        except np.linalg.LinAlgError:
            break
            
        # Clamp updates [S1]
        translation_update = delta_p[3:6]
        rotation_update = delta_p[0:3]
        translation_update = np.clip(translation_update, -1.0, 1.0)
        rotation_update = np.clip(rotation_update, -math.radians(5), math.radians(5))
        
        t = t + translation_update
        r = r + rotation_update
        
        current_error = np.sum(residuals**2)
        if iteration > 0 and abs(prev_error - current_error) < convergence_threshold:
            break
        prev_error = current_error

    # 3. Generate Predictor and Residual [S1]
    predictor = np.zeros_like(target_block_b)
    for p_i in get_voxel_centers(block_size):
        warped_pos = inverse_rigid_transform(p_i, r, t)
        pred_val = trilinear_interpolate(neighborhood, warped_pos)
        predictor[get_index(p_i, block_size)] = pred_val
        
    residual = target_block_b - predictor
    
    # 4. Encode Residual and Motion [S1]
    residual_coeffs = encode_transform_domain(residual)
    return (r, t), residual_coeffs

def assemble_3x3x3_neighborhood(tsdf, block_index, block_size):
    """Retrieves a 3x3x3 grid of blocks centered on block_index [G]."""
    volume = np.zeros((3 * block_size, 3 * block_size, 3 * block_size))
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            for dz in range(-1, 2):
                neighbor_idx = (block_index[0] + dx, block_index[1] + dy, block_index[2] + dz)
                if neighbor_idx in tsdf:
                    block = tsdf[neighbor_idx]
                    volume[dx * block_size:(dx + 1) * block_size,
                           dy * block_size:(dy + 1) * block_size,
                           dz * block_size:(dz + 1) * block_size] = block
    return volume

def get_voxel_centers(block_size):
    """Returns list of (x, y, z) coordinates for voxel centers in a block [G]."""
    centers = []
    for x in range(block_size):
        for y in range(block_size):
            for z in range(block_size):
                centers.append((x + 0.5, y + 0.5, z + 0.5))
    return centers

def get_index(p, block_size):
    """Maps 3D coordinate to flat array index [G]."""
    x, y, z = p
    return int(x) + int(y) * block_size + int(z) * block_size * block_size

def apply_rigid_transform(p, r, t):
    """Applies rigid transform T(p) = R*p + t [G]."""
    R = euler_to_rotation_matrix(r)
    p_arr = np.array(p)
    return np.dot(R, p_arr) + t

def inverse_rigid_transform(p, r, t):
    """Applies inverse rigid transform T^-1(p) = R^T*(p - t) [G]."""
    R = euler_to_rotation_matrix(r)
    R_inv = R.T
    p_arr = np.array(p)
    return np.dot(R_inv, p_arr - t)

def euler_to_rotation_matrix(r):
    """Converts Euler angles (rad) to 3x3 rotation matrix [G]."""
    rx, ry, rz = r
    Rx = np.array([[1, 0, 0], [0, math.cos(rx), -math.sin(rx)], [0, math.sin(rx), math.cos(rx)]])
    Ry = np.array([[math.cos(ry), 0, math.sin(ry)], [0, 1, 0], [-math.sin(ry), 0, math.cos(ry)]])
    Rz = np.array([[math.cos(rz), -math.sin(rz), 0], [math.sin(rz), math.cos(rz), 0], [0, 0, 1]])
    return Rz @ Ry @ Rx

def trilinear_interpolate(volume, point):
    """Performs trilinear interpolation on a 3D volume at a fractional point [G]."""
    x, y, z = point
    x0, y0, z0 = int(math.floor(x)), int(math.floor(y)), int(math.floor(z))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    dx, dy, dz = x - x0, y - y0, z - z0
    
    def clamp(v, min_v, max_v):
        return max(min_v, min(v, max_v))
        
    v000 = volume[clamp(x0, 0, volume.shape[0]-1), clamp(y0, 0, volume.shape[1]-1), clamp(z0, 0, volume.shape[2]-1)]
    v100 = volume[clamp(x1, 0, volume.shape[0]-1), clamp(y0, 0, volume.shape[1]-1), clamp(z0, 0, volume.shape[2]-1)]
    v010 = volume[clamp(x0, 0, volume.shape[0]-1), clamp(y1, 0, volume.shape[1]-1), clamp(z0, 0, volume.shape[2]-1)]
    v110 = volume[clamp(x1, 0, volume.shape[0]-1), clamp(y1, 0, volume.shape[1]-1), clamp(z0, 0, volume.shape[2]-1)]
    v001 = volume[clamp(x0, 0, volume.shape[0]-1), clamp(y0, 0, volume.shape[1]-1), clamp(z1, 0, volume.shape[2]-1)]
    v101 = volume[clamp(x1, 0, volume.shape[0]-1), clamp(y0, 0, volume.shape[1]-1), clamp(z1, 0, volume.shape[2]-1)]
    v011 = volume[clamp(x0, 0, volume.shape[0]-1), clamp(y1, 0, volume.shape[1]-1), clamp(z1, 0, volume.shape[2]-1)]
    v111 = volume[clamp(x1, 0, volume.shape[0]-1), clamp(y1, 0, volume.shape[1]-1), clamp(z1, 0, volume.shape[2]-1)]
    
    c00 = v000 * (1 - dx) + v100 * dx
    c01 = v010 * (1 - dx) + v110 * dx
    c10 = v001 * (1 - dx) + v101 * dx
    c11 = v011 * (1 - dx) + v111 * dx
    c0 = c00 * (1 - dy) + c01 * dy
    c1 = c10 * (1 - dy) + c11 * dy
    return c0 * (1 - dz) + c1 * dz

def compute_jacobian_row(p, r, t, neighborhood, block_size):
    """Computes Jacobian row for sample p using finite difference [S1]."""
    eps = 1e-5
    j_row = np.zeros(6)
    base_pred = trilinear_interpolate(neighborhood, apply_rigid_transform(p, r, t))
    for k in range(6):
        perturbed_r = r.copy()
        perturbed_t = t.copy()
        if k < 3:
            perturbed_r[k] += eps
        else:
            perturbed_t[k - 3] += eps
        perturbed_pred = trilinear_interpolate(neighborhood, apply_rigid_transform(p, perturbed_r, perturbed_t))
        j_row[k] = (perturbed_pred - base_pred) / eps
    return j_row

def encode_transform_domain(residual):
    """Applies standard 3D DCT to residual coefficients [G]."""
    # Standard 3D DCT implementation omitted for brevity; applies separable 1D DCT along each axis.
    return residual  # Placeholder for actual DCT transform [G]
```

## Subtopics

### Per-Block Motion Estimation
This is the core optimization subroutine used within Motion-Compensated Block Warping [S1]. It estimates the 6-DoF rigid transform for a specific block by aligning TSDF samples using a damped Gauss-Newton procedure [S1]. It is chosen when precise local motion estimation is required to minimize residual energy [G]. It differs from global motion estimation by operating on a per-block basis, allowing for independent motion of different parts of the scene [S1]. Key implementation details include clamping translation updates to 1 voxel and rotation updates to 5 degrees per step to prevent divergence, and falling back to zero-motion if the neighborhood is incomplete or samples are unreliable [S1].

### Hierarchical Motion Prediction
A motion modeling approach that decouples coarse and fine-grained motion to mitigate inaccuracies in deformation networks, preventing the querying of wrong grid cells [S2]. It predicts coarse motion $m_c$ corresponding to motion between grid cells, and fine-grained motion $m_f$ that operates only within grid cells [S2]. The step size $\alpha$ controls the magnitude of motion and is set to 0.0001 for synthetic scenes, close to the size of a single grid cell [S2]. The deformation network $\Phi_d$ is implemented as a 3-layer MLP with a width of 64 [S2].

## Operating characteristics
*   **Time Complexity:** The motion estimation step involves an iterative Gauss-Newton optimization [S1]. The cost depends on the number of samples in the block, the number of iterations, and the cost of trilinear interpolation and Jacobian computation [G].
*   **Memory Complexity:** Requires storing the current block, the previous frame's reconstructed TSDF, and the $3 \times 3 \times 3$ neighborhood for interpolation [S1]. The source uses a sparse representation with a GPU-side spatial hash table, reducing memory usage for empty space [S1].
*   **Performance:** Not given in the sources.
*   **Strengths:** Effectively reduces residual energy for rigidly moving objects, leading to better compression ratios for dynamic TSDF volumes [S1].
*   **Limitations:** Relies on the assumption of rigid motion within each block [S1]. Non-rigid deformations within a block will not be well-compensated [G]. Requires accurate motion estimation, which can be computationally expensive [S1].
*   **When to use:** Suitable for compressing dynamic scenes with significant rigid body motion, such as in 3D reconstruction from video sequences [S1].
*   **When not to use:** Not suitable for scenes with significant non-rigid deformation or where motion estimation is unreliable (e.g., textureless regions, occlusions) [S1].

## Background
The method originates from the paper "Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation" [S1]. It is part of a codec for compressing sparse TSDF volumes, leveraging traditional image compression techniques (DCT) and video compression concepts (motion compensation) for 3D data [S1]. The implementation uses voxel hashing for sparse storage [S1].

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
[S2] Ced-NeRF: A Compact and Efficient Method for Dynamic Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i4.28138, fetched 2026-10-26
