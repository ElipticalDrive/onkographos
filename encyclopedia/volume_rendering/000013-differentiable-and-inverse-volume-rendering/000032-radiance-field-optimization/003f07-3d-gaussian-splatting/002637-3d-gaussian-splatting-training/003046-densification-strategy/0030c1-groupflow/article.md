# GroupFlow

Also called: Groupwise SE(3) Motion Distillation

## Overview
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that replaces per-Gaussian neural deformation fields with shared SE(3) transformations for groups of Gaussians [S1, unofficial]. It solves the computational bottleneck of evaluating individual deformation networks for every Gaussian at every timestep, accelerating rendering by up to 13.71× and reducing training time by 2.53× while maintaining high image quality [S1, unofficial]. Unlike methods that retain per-Gaussian parameters, GroupFlow distills dense motion into a smaller set of groupwise rigid-body trajectories, regularizing noisy motion estimates [S1, unofficial].

## How it works
GroupFlow operates within the training loop of dynamic 3D Gaussian Splatting. It assumes a base set of Gaussians $\mathcal{G}$ with time-varying means $\mu_i^t$. The method replaces the neural deformation network $\mathcal{D}$ with a grouping mechanism and shared SE(3) parameters.

**Inputs:**
*   Set of $N$ Gaussians $\mathcal{G}$.
*   Sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ for each Gaussian $i$ across $F$ timesteps, typically sampled from the neural deformation field or previous training iteration.
*   Number of groups $J$ (where $J \ll N$).
*   Weighting factor $\lambda_r$ for trajectory similarity.

**Outputs:**
*   Group assignments for each Gaussian.
*   Shared SE(3) parameters $\{h_j^0, R_j^t, T_j^t\}$ for each group $j$.

**Data Structures:**
*   **Control Trajectories:** A set of $J$ control points $\{h_j^t\}_{j=0}^{J-1}$, where each $h_j^t \in \mathbb{R}^3$. Initialized via Farthest Point Sampling (FPS) on $\{\mu_i^0\}_{i=0}^{N-1}$ at $t=0$.
*   **Group Parameters:** For each group $j$, stored parameters are the initial control position $h_j^0 \in \mathbb{R}^3$, and time-varying rotation $R_j^t \in SO(3)$ and translation $T_j^t \in \mathbb{R}^3$.

**Steps:**

1.  **Initialization:** Initialize $J$ control trajectories $\{h_j^0\}_{j=0}^{J-1}$ using Farthest Point Sampling on the initial mean positions $\{\mu_i^0\}$ of all Gaussians.
2.  **Similarity Scoring:** For each Gaussian $i$ and each control trajectory $j$, compute the trajectory similarity score $S_{i,j}$:
    $$
    S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
    $$
    where $\mathrm{std}_t$ and $\mathrm{mean}_t$ are computed over the time steps $t=0 \dots F-1$. $\lambda_r$ balances the importance of temporal stability (std) versus average displacement (mean).
3.  **Assignment:** Assign each Gaussian $i$ to the group $j^*$ with the lowest similarity score:
    $$
    j^* = \arg\min_j S_{i,j}
    $$
    Let $\mathcal{M}^j$ be the set of mean positions for all Gaussians assigned to group $j$.
4.  **Optimization:** Optimize the shared parameters $\{h_j^0, R_j^t, T_j^t\}$ jointly with the scene reconstruction loss. The deformation for any Gaussian $i$ in group $j$ is predicted by applying the SE(3) transformation relative to the control point:
    $$
    \mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
    $$
    Here, $\mu_i^0$ is the initial mean of Gaussian $i$, and $h_j^0$ is the initial position of the control trajectory. $R_j^t$ and $T_j^t$ are learned per-group, per-timestep.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def groupflow_assign_and_transform(means_history, num_groups, lambda_r):
    """
    means_history: Tensor of shape (N, F, 3)
    num_groups: int J
    lambda_r: float
    Returns: group_indices (N,), control_params (J, 3), rot_params (J, F, 3), trans_params (J, F, 3)
    """
    N, F, _ = means_history.shape
    
    # Step 1: Initialize control trajectories using Farthest Point Sampling on t=0
    # Note: FPS implementation details omitted; assume function returns indices of J farthest points
    initial_means = means_history[:, 0, :] # (N, 3)
    control_init_indices = farthest_point_sampling(initial_means, num_groups)
    h_j_0 = initial_means[control_init_indices] # (J, 3)
    
    # Step 2 & 3: Compute similarity and assign groups
    # Expand dimensions for broadcasting: means (N, F, 1, 3), controls (1, F, J, 3)
    # Note: h_j^t is not fixed; for assignment, we use the current estimate or a proxy.
    # The source implies assignment is based on similarity to control trajectories.
    # In practice, this might be iterative or use a fixed initialization. 
    # Assuming h_j^t is approximated by h_j^0 for initial assignment or updated iteratively.
    # For a single pass implementation based on source description:
    
    # Compute distances for all timesteps
    # We need h_j^t. If not provided, we might use h_j^0 as a static reference for assignment
    # or assume a preliminary deformation. The source says "assign... using trajectory similarity".
    # Let's assume we use the current control positions h_j^t. If h_j^t is optimized, 
    # this step is part of the joint optimization loop.
    
    # Simplified assignment using initial control positions as proxy for trajectory
    # This is a simplification; in the paper, this is likely part of the joint optimization.
    # We compute S_ij based on the history of mu_i and the current estimate of h_j.
    
    # To make this self-contained, we assume h_j^t is initialized to h_j^0 for the assignment step
    # or that the assignment is done based on the initial configuration.
    
    h_j_t = h_j_0.unsqueeze(0).unsqueeze(0) # (1, 1, J, 3)
    mu_i_t = means_history.unsqueeze(2) # (N, F, 1, 3)
    
    dists = torch.norm(mu_i_t - h_j_t, dim=3) # (N, F, J)
    
    mean_dist = torch.mean(dists, dim=1) # (N, J)
    std_dist = torch.std(dists, dim=1) # (N, J)
    
    S_ij = lambda_r * std_dist + (1 - lambda_r) * mean_dist # (N, J)
    
    group_indices = torch.argmin(S_ij, dim=1) # (N,)
    
    # Step 4: Return parameters for optimization
    # In the actual method, h_j^0, R_j^t, T_j^t are learnable parameters optimized via backprop.
    # This function returns the assignment and initial control points.
    return group_indices, h_j_0

def groupflow_deform(mu_0, group_idx, h_j_0, R_j_t, T_j_t):
    """
    mu_0: (N, 3)
    group_idx: (N,)
    h_j_0: (J, 3)
    R_j_t: (J, 3) # Axis-angle representation for SO(3)
    T_j_t: (J, 3)
    Returns: mu_t (N, 3)
    """
    # Gather group parameters
    h_j_0_g = h_j_0[group_idx] # (N, 3)
    R_j_t_g = R_j_t[group_idx] # (N, 3)
    T_j_t_g = T_j_t[group_idx] # (N, 3)
    
    # Convert axis-angle to rotation matrix
    R_j_t_mat = axis_angle_to_matrix(R_j_t_g) # (N, 3, 3)
    
    # Apply SE(3) transformation
    # mu_i^t = R_j^t (mu_i^0 - h_j^0) + h_j^0 + T_j^t
    diff = mu_0 - h_j_0_g # (N, 3)
    rotated = torch.bmm(R_j_t_mat.unsqueeze(1), diff.unsqueeze(2)).squeeze(2) # (N, 3)
    mu_t = rotated + h_j_0_g + T_j_t_g
    
    return mu_t
```

**Parameters:**
*   $\lambda_r$: Weighting factor for trajectory similarity. Not given in the sources.
*   $J$: Number of groups. Not given in the sources, but $J \ll N$.
*   $F$: Number of timesteps. Not given in the sources.

**Edge Cases:**
*   **Degenerate Groups:** If a group contains only one Gaussian, the SE(3) fit may be unstable. [G] Ensure $J < N$ and handle singleton groups by falling back to per-Gaussian deformation or cloning.
*   **Precision:** SE(3) parameters must be maintained in a valid manifold (e.g., using axis-angle or quaternions for rotation) to avoid drift. [G]

## Operating characteristics
*   **Time Complexity:** Rendering complexity reduces from $O(N \cdot F)$ deformation evaluations to $O(J \cdot F)$, where $J \ll N$. Training time is reduced by 2.53× [S1, unofficial].
*   **Memory Complexity:** Stores $J$ sets of SE(3) parameters instead of $N$ neural network weights or per-Gaussian deformation parameters.
*   **Performance:** On MonoDyGauBench, GroupFlow achieves 13.71× faster rendering and 2.53× shorter training compared to baselines [S1, unofficial].
*   **Strengths:** High rendering speed, reduced training time, temporal coherence, robustness to noise.
*   **Limitations:** Assumes locally coherent motion; may struggle with complex, non-rigid deformations that do not align well with SE(3) groups.

## Background
GroupFlow was introduced in "SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping" [S1, unofficial]. It is part of a pipeline that includes Temporal Sensitivity Pruning (TSP) and Temporal Sensitivity Sampling (TSS). The method addresses the high computational cost of dynamic 3D Gaussian Splatting by distilling neural motion fields into shared SE(3) transformations.

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
