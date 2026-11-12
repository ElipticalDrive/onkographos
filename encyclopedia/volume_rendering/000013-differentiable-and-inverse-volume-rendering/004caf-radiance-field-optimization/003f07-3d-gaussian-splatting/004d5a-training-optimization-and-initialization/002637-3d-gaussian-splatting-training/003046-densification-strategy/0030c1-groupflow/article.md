# GroupFlow

Also called: Groupwise SE(3) Motion Distillation

## Overview
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that replaces per-Gaussian neural deformation fields with shared SE(3) transformations for groups of Gaussians [S1, unofficial]. It solves the computational bottleneck of evaluating individual deformation networks for every primitive at every timestep by distilling local motion coherence into a smaller set of group-level rigid-body trajectories [S1, unofficial]. Unlike sibling methods that may use global rigid models or dense neural fields, GroupFlow adapts to local motion patterns via clustering, offering a trade-off between the fidelity of neural fields and the speed of rigid body approximations [S1, unofficial].

## How it works
GroupFlow operates on a set of 3D Gaussians $\mathcal{G}$, each represented by a sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ across $F$ timesteps [S1, unofficial]. The method consists of three main phases: initialization and clustering, trajectory fitting, and deformation prediction [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Set of Gaussians $\mathcal{G}$ with temporal mean positions $\mu_i^t$ for $t \in [0, F-1]$.
    *   Number of groups $J$ (number of control trajectories).
    *   Weighting factor $\lambda_r$ for the similarity score.
*   **Outputs:**
    *   Group assignments for each Gaussian.
    *   Shared SE(3) parameters $\{h_j^0, R_j^t, T_j^t\}$ for each group $j \in [0, J-1]$.

### Data Structures
*   **Control Trajectories:** A set of $J$ control points $\{h_j^t\}_{j=0}^{J-1}$ for each timestep $t$.
*   **Group Parameters:** For each group $j$, store the initial control position $h_j^0$, the rotation sequence $R_j^t \in SO(3)$, and the translation sequence $T_j^t \in \mathbb{R}^3$ for $t \in [0, F-1]$.

### Steps
1.  **Initialize Control Trajectories:** Select $J$ initial control points $\{h_j^0\}_{j=0}^{J-1}$ from the set of Gaussian means at $t=0$ using farthest point sampling [S1, unofficial].
2.  **Assign Gaussians to Groups:** For each Gaussian $i$, compute the trajectory similarity score $S_{i,j}$ to each control trajectory $j$ and assign $i$ to the group $j^*$ with the minimum score [S1, unofficial].
3.  **Fit Group-Wise SE(3) Trajectories:** For each group $j$, optimize the parameters $\{h_j^0, R_j^t, T_j^t\}$ to best fit the assigned Gaussians' trajectories [S1, unofficial].
4.  **Predict Deformation:** For a Gaussian $i$ assigned to group $j$, compute its mean position at timestep $t$ using the learned SE(3) transformation [S1, unofficial].

### Formulas
The trajectory similarity score $S_{i,j}$ determines the assignment of Gaussian $i$ to group $j$ [S1, unofficial]:

$$
S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
$$

Where:
*   $S_{i,j}$ is the similarity score between Gaussian $i$ and control trajectory $j$.
*   $\lambda_r$ is a weighting factor (typically between 0 and 1).
*   $\mathrm{std}_t(\cdot)$ computes the standard deviation over time $t$.
*   $\mathrm{mean}_t(\cdot)$ computes the mean over time $t$.
*   $\| \cdot \|$ is the Euclidean norm.
*   $\mu_i^t$ is the mean position of Gaussian $i$ at time $t$.
*   $h_j^t$ is the position of control trajectory $j$ at time $t$.

The predicted mean position $\mu_i^t$ for a Gaussian $i$ in group $j$ at timestep $t$ is given by [S1, unofficial]:

$$
\mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
$$

Where:
*   $\mu_i^t$ is the predicted mean position of Gaussian $i$ at time $t$.
*   $R_j^t$ is the rotation matrix for group $j$ at time $t$.
*   $T_j^t$ is the translation vector for group $j$ at time $t$.
*   $\mu_i^0$ is the initial mean position of Gaussian $i$.
*   $h_j^0$ is the initial position of control trajectory $j$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def groupflow_init(gaussians_means, F, J, lambda_r):
    """
    Initialize GroupFlow clustering and parameters.
    
    Args:
        gaussians_means: Array of shape (N, F, 3) containing mean positions for N Gaussians over F timesteps.
        F: Number of timesteps.
        J: Number of groups/control trajectories.
        lambda_r: Weighting factor for similarity score.
        
    Returns:
        group_assignments: Array of shape (N,) with group indices.
        control_params: Dictionary containing h_j0, R_j, T_j for each group j.
    """
    N = gaussians_means.shape[0]
    
    # Step 1: Initialize control trajectories using farthest point sampling at t=0
    # Note: Exact FPS implementation details are not provided in sources [S1, unofficial].
    # Placeholder for FPS logic:
    h_j0 = farthest_point_sampling(gaussians_means[:, 0, :], J)
    
    # Initialize control trajectories for all timesteps (initially same as t=0)
    h_jt = np.tile(h_j0[:, np.newaxis, :], (1, F, 1))
    
    # Step 2: Assign Gaussians to groups
    group_assignments = np.zeros(N, dtype=int)
    
    for i in range(N):
        min_score = np.inf
        best_j = 0
        for j in range(J):
            # Compute distance for all timesteps
            dists = np.linalg.norm(gaussians_means[i, :, :] - h_jt[j, :, :], axis=1)
            
            # Compute similarity score
            std_t = np.std(dists)
            mean_t = np.mean(dists)
            score = lambda_r * std_t + (1 - lambda_r) * mean_t
            
            if score < min_score:
                min_score = score
                best_j = j
        group_assignments[i] = best_j
        
    # Step 3: Fit group-wise SE(3) trajectories
    # This involves optimizing R_j^t and T_j^t for each group to minimize reconstruction error.
    # The sources do not provide the specific optimization algorithm (e.g., Kabsch, SVD, or gradient descent).
    # Placeholder for SE(3) fitting logic:
    control_params = fit_se3_trajectories(gaussians_means, group_assignments, J, F)
    
    return group_assignments, control_params

def groupflow_predict(gaussians_means, group_assignments, control_params, t):
    """
    Predict mean positions for all Gaussians at timestep t using GroupFlow.
    
    Args:
        gaussians_means: Array of shape (N, F, 3) containing initial mean positions (t=0).
        group_assignments: Array of shape (N,) with group indices.
        control_params: Dictionary containing h_j0, R_j, T_j for each group j.
        t: Target timestep.
        
    Returns:
        predicted_means: Array of shape (N, 3) with predicted mean positions.
    """
    N = gaussians_means.shape[0]
    predicted_means = np.zeros((N, 3))
    
    for i in range(N):
        j = group_assignments[i]
        mu_i0 = gaussians_means[i, 0, :]
        h_j0 = control_params['h_j0'][j]
        R_jt = control_params['R_j'][j, t]
        T_jt = control_params['T_j'][j, t]
        
        # Apply SE(3) transformation
        predicted_means[i, :] = R_jt @ (mu_i0 - h_j0) + h_j0 + T_jt
        
    return predicted_means
```

### Parameters and Edge Cases
*   **$\lambda_r$**: Weighting factor for the similarity score. Not given in the sources.
*   **$J$**: Number of groups. Not given in the sources.
*   **$F$**: Number of timesteps. Not given in the sources.
*   **Edge Cases**: If a Gaussian has no temporal variation, the standard deviation term in the similarity score will be zero, and the assignment will be driven by the mean distance. If $J$ is too small, motion coherence within groups may be lost; if $J$ is too large, the compression benefit is reduced.

## Operating characteristics
GroupFlow reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group) [S1, unofficial]. On the MonoDyGauBench dataset, integrating GroupFlow with Temporal Sensitivity Pruning (TSP) and Temporal Sensitivity Sampling (TSS) accelerates rendering by 13.71× and reduces training time by 2.53× compared to baselines [S1, unofficial]. It uses 10× fewer primitives than the neural-field baseline while maintaining image quality [S1, unofficial]. The method is most effective in scenes with locally coherent motion patterns.

## Background
GroupFlow was introduced in the paper "SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping" [S1, unofficial]. It is part of the SpeeDe3DGS framework, which aims to accelerate dynamic 3D Gaussian Splatting by combining temporal pruning, sampling, and motion grouping. The method addresses the computational cost of per-Gaussian neural deformation fields by distilling them into shared SE(3) transformations.

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
