# GroupFlow

Also called: Groupwise SE(3) Motion Distillation, Motion Grouping

## Overview
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that distills per-Gaussian neural deformation fields into shared SE(3) transformations for groupwise motion [S1, unofficial]. It solves the computational bottleneck of evaluating individual neural networks for every Gaussian at every timestep by clustering Gaussians into groups and assigning each group a single learned rigid-body trajectory [S1, unofficial]. This reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group), accelerating rendering by up to 13.71× and shortening training time by 2.53× compared to baseline deformable methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dense set of dynamic 3D Gaussians $\mathcal{G}$, where each Gaussian $i$ has a sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ across $F$ timesteps, derived from a pre-trained neural deformation field [S1, unofficial].
*   **Output:** A set of $J$ control trajectories $\{h_j^t\}_{j=1}^J$ and associated SE(3) parameters $\{R_j^t, T_j^t\}$ that define the motion for each group [S1, unofficial].

**Data Structures:**
*   **Gaussian Sequence:** For each Gaussian $i$, store the mean positions $\mu_i^t \in \mathbb{R}^3$ for $t=0 \dots F-1$.
*   **Control Trajectories:** $J$ control points $h_j^0 \in \mathbb{R}^3$ initialized at $t=0$.
*   **Group Parameters:** For each group $j$ and timestep $t$, store rotation matrix $R_j^t \in \mathbb{R}^{3 \times 3}$ and translation vector $T_j^t \in \mathbb{R}^3$.

**Algorithm Steps:**

1.  **Initialization:** Initialize $J$ control trajectories $\{h_j^t\}$ by performing Farthest Point Sampling (FPS) on the initial mean positions $\{\mu_i^0\}_{i=1}^N$ at $t=0$ [S1, unofficial].
2.  **Assignment:** Assign each Gaussian $i$ to the group $j$ that minimizes the trajectory similarity score $S_{i,j}$ [S1, unofficial].
3.  **Optimization:** Jointly optimize the shared flow parameters $\{h_j^0, R_j^t, T_j^t\}$ with the scene representation to minimize reconstruction error [S1, unofficial].

**Formulas:**

The trajectory similarity score $S_{i,j}$ determines the assignment of Gaussian $i$ to control trajectory $j$:
$$
S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
$$
Where:
*   $S_{i,j}$ is the similarity score between Gaussian $i$ and control trajectory $j$.
*   $\lambda_r$ is a weighting factor balancing stability and accuracy.
*   $\mathrm{std}_t(\cdot)$ and $\mathrm{mean}_t(\cdot)$ compute the standard deviation and mean over timesteps $t=0 \dots F-1$.
*   $\mu_i^t$ is the mean position of Gaussian $i$ at time $t$.
*   $h_j^t$ is the position of control trajectory $j$ at time $t$.

The predicted mean position $\mu_i^t$ for a Gaussian $i$ belonging to group $j$ is calculated using the learned SE(3) transformation:
$$
\mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
$$
Where:
*   $\mu_i^t$ is the predicted mean position of Gaussian $i$ at time $t$.
*   $R_j^t$ is the rotation matrix for group $j$ at time $t$.
*   $T_j^t$ is the translation vector for group $j$ at time $t$.
*   $\mu_i^0$ is the initial mean position of Gaussian $i$.
*   $h_j^0$ is the initial position of control trajectory $j$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def groupflow_distill(gaussians, F, J, lambda_r):
    """
    Distills per-Gaussian neural motion into groupwise SE(3) transformations.
    
    Args:
        gaussians: List of N Gaussians, each with attribute 'means' of shape (F, 3).
        F: Number of timesteps.
        J: Number of groups/control trajectories.
        lambda_r: Weighting factor for similarity score.
        
    Returns:
        control_points: Array of shape (J, 3) for h_j^0.
        rotations: Array of shape (J, F, 3, 3) for R_j^t.
        translations: Array of shape (J, F, 3) for T_j^t.
        assignments: Array of shape (N,) indicating group index for each Gaussian.
    """
    N = len(gaussians)
    
    # Step 1: Initialize control trajectories via Farthest Point Sampling at t=0
    initial_means = np.array([g.means[0] for g in gaussians]) # Shape (N, 3)
    control_points = farthest_point_sampling(initial_means, J) # Shape (J, 3)
    
    # Initialize SE(3) parameters (identity rotation, zero translation)
    rotations = np.zeros((J, F, 3, 3))
    for j in range(J):
        rotations[j, :, :, :] = np.eye(3)
    translations = np.zeros((J, F, 3))
    
    # Step 2 & 3: Joint Optimization Loop
    # Note: In practice, this involves backpropagating through the rendering loss
    # to update control_points, rotations, and translations.
    # We simulate the assignment and prediction logic here.
    
    assignments = np.zeros(N, dtype=int)
    
    for iteration in range(num_optimization_steps):
        # Compute similarity scores for all Gaussians and all control trajectories
        # S_{i,j} = lambda_r * std_t(||mu_i^t - h_j^t||) + (1-lambda_r) * mean_t(||mu_i^t - h_j^t||)
        
        scores = np.zeros((N, J))
        for i in range(N):
            mu_i = gaussians[i].means # Shape (F, 3)
            for j in range(J):
                h_j = control_points[j] # Shape (3,)
                # Note: h_j^t is not explicitly stored as a full trajectory in the 
                # simplified model, but the similarity is computed against the 
                # *learned* trajectory or the current estimate. 
                # The source implies h_j^t is part of the optimization state.
                # For implementation, we assume h_j^t is derived from h_j^0, R_j^t, T_j^t
                # or stored separately. The source formula uses h_j^t directly.
                
                # Calculate distances over time
                # Assuming h_j^t is available or computed from current params
                dists = np.linalg.norm(mu_i - control_points_trajectory(j, iteration), axis=1) # Shape (F,)
                
                std_t = np.std(dists)
                mean_t = np.mean(dists)
                
                scores[i, j] = lambda_r * std_t + (1 - lambda_r) * mean_t
        
        # Assign each Gaussian to the best group
        assignments = np.argmin(scores, axis=1)
        
        # Update SE(3) parameters for each group based on assigned Gaussians
        for j in range(J):
            group_indices = np.where(assignments == j)[0]
            if len(group_indices) == 0:
                continue
            
            # Fit SE(3) to the means of assigned Gaussians
            # This typically involves solving a Procrustes problem or optimizing
            # R_j^t and T_j^t to minimize || mu_i^t - (R_j^t(mu_i^0 - h_j^0) + h_j^0 + T_j^t) ||
            # for all i in group_indices and t in 0..F-1.
            fit_se3_trajectory(gaussians, group_indices, rotations[j], translations[j], control_points[j])

    return control_points, rotations, translations, assignments

def fit_se3_trajectory(gaussians, group_indices, R, T, h0):
    """
    Optimizes R and T for a group to fit the Gaussians' trajectories.
    """
    # Placeholder for the actual optimization step (e.g., gradient descent)
    # The source states parameters are optimized jointly with the scene.
    pass

def control_points_trajectory(j, iteration):
    """
    Returns the current estimated trajectory h_j^t for group j.
    In the source, h_j^t is used in the similarity score.
    It is likely that h_j^t is either a separate learnable parameter or
    derived from the SE(3) parameters. The source formula uses h_j^t directly.
    """
    # If h_j^t is a separate parameter:
    # return h_j_trajectory[j] 
    # If it's derived, we would compute it. The source is slightly ambiguous 
    # on whether h_j^t is an independent variable or derived.
    # Given "fit a group-wise SE(3) trajectory... by aligning sampled means",
    # h_j^t likely represents the center of the group's motion.
    pass
```

**Parameters:**
*   $J$: Number of groups. Not given in the sources.
*   $\lambda_r$: Weighting factor for the similarity score. Not given in the sources.
*   $F$: Number of timesteps. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Empty Groups:** If a group receives no Gaussians during assignment, its SE(3) parameters cannot be updated. A guard should handle this case [G].
*   **Precision:** The similarity score involves standard deviation and mean of distances. Numerical stability should be considered for very small or very large distances [G].
*   **Boundary Conditions:** The SE(3) transformation assumes rigid body motion. Non-rigid deformations within a group may lead to artifacts if the group is too large [G].

## Subtopics

### Temporal Sensitivity Pruning (TSP)
Removes low-impact Gaussians via temporally aggregated sensitivity analysis. Chosen when reducing primitive count is critical for speed [S1, unofficial].

### Temporal Sensitivity Sampling (TSS)
Perturbs timestamps to suppress floaters and improve temporal coherence. Chosen when temporal artifacts are prevalent [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering complexity is reduced from $O(N)$ transformations per frame to $O(J)$ transformations per frame, where $J \ll N$ [S1, unofficial].
*   **Memory Complexity:** Stores $J$ sets of SE(3) parameters instead of $N$ neural network weights or deformation fields [S1, unofficial].
*   **Measured Performance:** On the MonoDyGauBench dataset (50 dynamic scenes), GroupFlow accelerates rendering by 13.71× and shortens training time by 2.53× compared to baselines [S1, unofficial].
*   **Strengths:** Substantial acceleration in training and rendering; regularizes the learned motion field, smoothing noisy trajectories; improves robustness in real-world dynamic scenes [S1, unofficial].
*   **Limitations:** Assumes locally coherent motion within groups; may not capture highly complex, non-rigid deformations if groups are too coarse [G].

## Background
GroupFlow was introduced in the paper "SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping" [S1, unofficial]. It is part of a suite of techniques including Temporal Sensitivity Pruning and Sampling, designed to address the computational inefficiency of neural deformation fields in dynamic 3D Gaussian Splatting.

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
