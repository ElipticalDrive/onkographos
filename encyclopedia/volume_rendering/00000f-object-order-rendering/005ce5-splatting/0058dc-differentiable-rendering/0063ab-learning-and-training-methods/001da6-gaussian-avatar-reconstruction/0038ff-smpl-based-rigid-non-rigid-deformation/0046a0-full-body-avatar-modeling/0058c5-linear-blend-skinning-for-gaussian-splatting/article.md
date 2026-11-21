# Linear Blend Skinning for Gaussian Splatting

Also called: Canonical LBS, Canonical to Observation Space Transformation, Forward Skinning, Linear Blend Skinning (LBS) for 3DGS Avatars

## Overview
Linear Blend Skinning (LBS) for Gaussian Splatting is a deformation technique that transforms 3D Gaussians from a canonical (unposed) space to an observation (posed) space using skeleton-driven rigid joint transformations and learnable per-Gaussian skinning weights [S1, S2]. Unlike NeRF-based inverse skinning methods that map observation-space points to canonical space, this forward approach applies fixed skinning weights to transform Gaussian positions and rotations directly [S2]. It avoids pose blend shapes and template vertex deformation, relying instead on an MLP for pose-dependent deformations to handle inaccuracies in shape estimation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Canonical 3D Gaussian attributes (position $\mu_c$, rotation $r_c$, scale $s$, color, opacity), joint transformation matrices $M_{j,t}$ derived from a parametric body model (e.g., SMPL) for each training timestep $t$, and initial learnable skinning weights $w$.
*   **Outputs:** Posed Gaussian attributes (position $\mu_o$, rotation $r_o$) ready for rasterization.

**Data Structures:**
*   **Skinned Weights ($w$):** A vector of size $J$ (total number of joints) per Gaussian. Optimized during training.
*   **Joint Transformation Matrices ($M_{j,t}$):** Precomputed rigid transformation matrices for each joint $j$ and timestep $t$. Each matrix $M_{j,t}$ contains a rotation component $A_R(\theta_t)$ and a translation component $A_T(\theta_t)$ [S2].

**Numbered Steps:**
1.  **Precompute Joint Transformations:** For each training timestep $t$, compute the rigid transformation matrix $M_{j,t}$ for every body joint $j$ based on the pose parameters $\theta_t$ from the parametric mesh [S1, S2].
2.  **Optimize Skinning Weights:** Initialize learnable per-Gaussian skinning weights vector $w$. Optimize $w$ via gradient descent. Note that $w$ can contain negative values during optimization [S1].
3.  **Activate and Normalize Weights:** Apply ReLU activation to each component $w_j$ of the skinning weights vector to ensure non-negativity. Normalize the vector such that its components sum to 1:
    $$ w_j^{final} = \frac{\max(0, w_j)}{\sum_{k=1}^{J} \max(0, w_k)} $$
    This yields a well-defined skinning weights vector [S1].
4.  **Transform Position:** Compute the observation space position $\mu_o$ by applying the weighted sum of joint transformations to the canonical position $\mu_c$:
    $$ \mu_o = A(\theta_t) \cdot \mu_c = \sum_{k=1}^{J} W_k M_{k,t} \cdot \mu_c $$
    where $W_k$ are the normalized skinning weights and $M_{k,t}$ is the transformation matrix for joint $k$ [S2].
5.  **Transform Rotation:** Transform the canonical rotation $r_c$ (represented as a quaternion or vector) to observation space rotation $r_o$ using the rotation component $A_R(\theta_t)$ of the combined transformation matrix:
    $$ r_o = A_R(\theta_t) r_c $$
    where $A_R(\theta_t)$ is the rotation part of the blended transformation [S2].

**Parameters:**
*   **Learning Rate for MLP and Skinning Weights ($w$):** $1 \times 10^{-4}$ [S1].
*   **Learning Rate for Latent Codes ($l$):** $2.5 \times 10^{-3}$ [S1].
*   **Skinned Weights ($W$):** Static attributes of the Gaussian after training; not regressed per-frame [S2].
*   **Joint Transformation Matrices ($\theta_k$):** Pose parameters derived from the parametric human mesh (e.g., SMPL) [S2].

**Edge Cases and Pitfalls:**
*   **Negative Weights:** During optimization, skinning weights $w$ may become negative. The ReLU activation and normalization step is critical to ensure valid convex combination weights [S1].
*   **Pose Blend Shapes:** Pose blend shapes are explicitly excluded because shape estimation and obtained blend shapes may be inaccurate. An MLP is used to handle pose-dependent deformations instead [S1].
*   **Template Vertex Deformation:** Not necessary as LBS is applied directly on canonical Gaussians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_posed_gaussian(canonical_pos, canonical_rot, skinning_weights_raw, joint_transforms):
    """
    canonical_pos: Tensor of shape (N, 3), canonical positions of N Gaussians
    canonical_rot: Tensor of shape (N, 4), canonical rotations (quaternions) of N Gaussians
    skinning_weights_raw: Tensor of shape (N, J), raw learnable skinning weights
    joint_transforms: List of J matrices, each of shape (4, 4), representing M_{j,t} for current timestep t
    
    Returns:
    posed_pos: Tensor of shape (N, 3)
    posed_rot: Tensor of shape (N, 4)
    """
    J = len(joint_transforms)
    N = canonical_pos.shape[0]
    
    # Step 2 & 3: Activate and Normalize Weights
    # Apply ReLU to ensure non-negativity
    relu_weights = torch.relu(skinning_weights_raw)
    
    # Normalize so components sum to 1
    sum_weights = torch.sum(relu_weights, dim=1, keepdim=True)
    # Handle potential division by zero if all weights are zero (guard [G])
    sum_weights = torch.clamp(sum_weights, min=1e-8)
    normalized_weights = relu_weights / sum_weights
    
    # Step 4: Transform Position
    # Initialize posed position
    posed_pos = torch.zeros_like(canonical_pos)
    
    # Homogeneous coordinates for position (N, 4)
    pos_homogeneous = torch.cat([canonical_pos, torch.ones(N, 1)], dim=1)
    
    for j in range(J):
        # Get transformation matrix for joint j
        M_j = joint_transforms[j] # Shape (4, 4)
        W_j = normalized_weights[:, j:j+1] # Shape (N, 1)
        
        # Apply transformation: M_j * pos_homogeneous
        transformed_pos_hom = torch.matmul(pos_homogeneous, M_j.T) # Shape (N, 4)
        transformed_pos = transformed_pos_hom[:, :3] # Shape (N, 3)
        
        # Add weighted contribution
        posed_pos += W_j * transformed_pos
        
    # Step 5: Transform Rotation
    # Compute blended rotation matrix A_R(theta_t)
    # Note: The sources define A(theta_t) as containing A_R and A_T.
    # We need to extract the rotation part of the blended transformation.
    # Since LBS blends transformations, we blend the rotation matrices.
    
    # Extract rotation matrices from joint transforms
    # Assuming joint_transforms are 4x4, rotation is top-left 3x3
    rot_matrices = []
    for j in range(J):
        M_j = joint_transforms[j]
        R_j = M_j[:3, :3] # Shape (3, 3)
        rot_matrices.append(R_j)
        
    # Blend rotation matrices using normalized weights
    # A_R(theta_t) = sum(W_j * R_j)
    blended_rot_matrix = torch.zeros(3, 3)
    for j in range(J):
        blended_rot_matrix += normalized_weights[:, j:j+1] * rot_matrices[j]
        
    # Apply blended rotation to canonical rotations
    # canonical_rot is (N, 4) quaternions
    # Convert quaternions to rotation matrices, apply blended_rot_matrix, convert back?
    # Or apply quaternion rotation directly if A_R is represented as quaternion.
    # Source [S2] says: r_o = A_R(theta_t) r_c. 
    # If r_c is a vector (direction), it's matrix multiplication.
    # If r_c is a quaternion, it's quaternion multiplication.
    # 3DGS rotations are typically quaternions. 
    # However, the formula r_o = A_R(theta_t) r_c suggests matrix-vector multiplication if r_c is a vector.
    # But 3DGS uses quaternions for orientation. 
    # Let's assume r_c represents the orientation quaternion.
    # To apply a rotation matrix R to a quaternion q, we convert q to R_q, 
    # compute R_total = R @ R_q, then convert back to quaternion.
    
    # Convert canonical quaternions to rotation matrices
    # q is (N, 4), w,x,y,z
    canonical_rot_mats = quaternion_to_rotation_matrix(canonical_rot) # Shape (N, 3, 3)
    
    # Apply blended rotation
    # posed_rot_mats = blended_rot_matrix @ canonical_rot_mats
    # Note: broadcasting. blended_rot_matrix is (3,3), canonical_rot_mats is (N,3,3)
    posed_rot_mats = torch.matmul(blended_rot_matrix.unsqueeze(0), canonical_rot_mats)
    
    # Convert back to quaternions
    posed_rot = rotation_matrix_to_quaternion(posed_rot_mats) # Shape (N, 4)
    
    return posed_pos, posed_rot

def quaternion_to_rotation_matrix(q):
    """Converts quaternions (N, 4) to rotation matrices (N, 3, 3)."""
    # Standard conversion [G]
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    N = q.shape[0]
    
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    R = torch.zeros(N, 3, 3, device=q.device)
    R[:, 0, 0] = 1 - 2 * (yy + zz)
    R[:, 0, 1] = 2 * (xy - wz)
    R[:, 0, 2] = 2 * (xz + wy)
    R[:, 1, 0] = 2 * (xy + wz)
    R[:, 1, 1] = 1 - 2 * (xx + zz)
    R[:, 1, 2] = 2 * (yz - wx)
    R[:, 2, 0] = 2 * (xz - wy)
    R[:, 2, 1] = 2 * (yz + wx)
    R[:, 2, 2] = 1 - 2 * (xx + yy)
    return R

def rotation_matrix_to_quaternion(R):
    """Converts rotation matrices (N, 3, 3) to quaternions (N, 4)."""
    # Standard conversion [G]
    N = R.shape[0]
    q = torch.zeros(N, 4, device=R.device)
    
    tr = R[:, 0, 0] + R[:, 1, 1] + R[:, 2, 2]
    
    # Case 1: tr > 0
    mask1 = tr > 0
    s1 = 0.5 / torch.sqrt(tr[mask1] + 1e-8)
    q[mask1, 0] = 0.25 / s1
    q[mask1, 1] = (R[mask1, 2, 1] - R[mask1, 1, 2]) * s1
    q[mask1, 2] = (R[mask1, 0, 2] - R[mask1, 2, 0]) * s1
    q[mask1, 3] = (R[mask1, 1, 0] - R[mask1, 0, 1]) * s1
    
    # Case 2: R[0,0] is largest
    mask2 = (~mask1) & (R[:, 0, 0] > R[:, 1, 1]) & (R[:, 0, 0] > R[:, 2, 2])
    s2 = 2.0 * torch.sqrt(1.0 + R[mask2, 0, 0] - R[mask2, 1, 1] - R[mask2, 2, 2] + 1e-8)
    q[mask2, 0] = (R[mask2, 2, 1] - R[mask2, 1, 2]) / s2
    q[mask2, 1] = 0.25 * s2
    q[mask2, 2] = (R[mask2, 0, 1] + R[mask2, 1, 0]) / s2
    q[mask2, 3] = (R[mask2, 0, 2] + R[mask2, 2, 0]) / s2
    
    # Case 3: R[1,1] is largest
    mask3 = (~mask1) & (~mask2) & (R[:, 1, 1] > R[:, 2, 2])
    s3 = 2.0 * torch.sqrt(1.0 + R[mask3, 1, 1] - R[mask3, 0, 0] - R[mask3, 2, 2] + 1e-8)
    q[mask3, 0] = (R[mask3, 0, 2] - R[mask3, 2, 0]) / s3
    q[mask3, 1] = (R[mask3, 0, 1] + R[mask3, 1, 0]) / s3
    q[mask3, 2] = 0.25 * s3
    q[mask3, 3] = (R[mask3, 1, 2] + R[mask3, 2, 1]) / s3
    
    # Case 4: R[2,2] is largest
    mask4 = ~mask1 & ~mask2 & ~mask3
    s4 = 2.0 * torch.sqrt(1.0 + R[mask4, 2, 2] - R[mask4, 0, 0] - R[mask4, 1, 1] + 1e-8)
    q[mask4, 0] = (R[mask4, 1, 0] - R[mask4, 0, 1]) / s4
    q[mask4, 1] = (R[mask4, 0, 2] + R[mask4, 2, 0]) / s4
    q[mask4, 2] = (R[mask4, 1, 2] + R[mask4, 2, 1]) / s4
    q[mask4, 3] = 0.25 * s4
    
    return q
```

## Subtopics

### Canonical LBS
The specific application of LBS where Gaussians are defined in a canonical space and deformed forward to the observation space. It differs from inverse skinning by avoiding the need to query the canonical space from observation space points. [Canonical LBS](canonical-lbs.md)

### Forward Skinning
Emphasizes the direction of deformation (canonical to observation). It is chosen when real-time rendering of posed avatars is required without iterative inverse mapping. [Forward Skinning](forward-skinning.md)

### Inverse Skinning
The alternative approach used in NeRF-based methods, mapping observation-space points to canonical space. It is slower for rendering but can handle complex non-rigid deformations differently. [Inverse Skinning](inverse-skinning.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot J)$ per Gaussian per frame, where $N$ is the number of Gaussians and $J$ is the number of joints. The precomputation of joint transforms is $O(J)$.
*   **Memory Complexity:** $O(N \cdot J)$ to store skinning weights. Joint transforms are $O(J)$ per timestep.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of Gaussians and joints. Precomputing joint transforms improves efficiency during training [S1].
*   **Strengths:** Avoids inaccuracies from pose blend shapes; efficient precomputation of joint transforms; direct transformation of Gaussian attributes.
*   **Limitations:** Relies on the accuracy of the parametric body model (e.g., SMPL); skinning weights are static after training, limiting per-frame adaptability; negative weights must be handled via ReLU and normalization.
*   **When to use:** Real-time rendering of animatable avatars where pose-dependent deformations are handled by an MLP and rigid joint transformations are sufficient for the base deformation.
*   **When not to use:** When high-fidelity non-rigid deformations beyond rigid joint transformations are required without additional MLP capacity, or when inverse mapping is preferred for specific rendering pipelines.

## Background
Linear Blend Skinning for Gaussian Splatting was introduced in the context of human avatar modeling using 3D Gaussian Splatting. It builds on the SMPL body model and adapts LBS to work with canonical Gaussians. The method was proposed to address inaccuracies in pose blend shapes and template vertex deformation by using learnable skinning weights and an MLP for pose-dependent deformations [S1]. It is part of the broader field of differentiable rendering and Gaussian Avatar Reconstruction [S2].

## Sources
[S1] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S2] Relightable and Dynamic Gaussian Avatar Reconstruction from Monocular Video, https://doi.org/10.1145/3746027.3754851, fetched 2026-11-06
