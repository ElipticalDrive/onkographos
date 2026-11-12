# GroupFlow

Also called: Groupwise SE(3) Motion Distillation, GroupFlow Motion Grouping

## Overview
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that replaces per-Gaussian neural deformation fields with shared SE(3) transformations for groups of Gaussians [S1, unofficial]. It solves the computational bottleneck of evaluating individual deformation networks for every primitive at every timestep, reducing the number of transformations from $N$ (per Gaussian) to $J$ (per group) [S1, unofficial]. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on temporal motion efficiency rather than spatial density management, though it often operates in conjunction with densification strategies to manage the underlying Gaussian count [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ dynamic 3D Gaussians $\mathcal{G}$, each with a sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ across $F$ timesteps, and a hyperparameter $\lambda_r$ for trajectory similarity weighting.
- **Output:** $J$ control trajectories $\{h_j^t\}_{j=0}^{J-1}$ and associated SE(3) parameters $\{R_j^t, T_j^t\}$ that define the motion for each group.

**Data Structures:**
- **Gaussian Mean Sequence:** For each Gaussian $i$, store the 3D mean position $\mu_i^t \in \mathbb{R}^3$ for each timestep $t \in [0, F-1]$.
- **Control Trajectories:** A list of $J$ control points. Each control trajectory $j$ has an initial position $h_j^0 \in \mathbb{R}^3$ and time-varying rotation $R_j^t \in SO(3)$ and translation $T_j^t \in \mathbb{R}^3$.
- **Group Assignment:** A mapping from each Gaussian index $i$ to a group index $j \in [0, J-1]$.

**Numbered Steps:**
1. **Initialization:** Initialize $J$ control trajectories $\{h_j^t\}$ using farthest point sampling on the initial mean positions $\{\mu_i^0\}$ at $t=0$ [S1, unofficial].
2. **Trajectory Similarity Calculation:** For each Gaussian $i$ and each control trajectory $j$, compute the similarity score $S_{i,j}$ using the formula:
   $$
   S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
   $$
   where $\mathrm{std}_t$ and $\mathrm{mean}_t$ are computed over the time dimension $t$, and $\lambda_r$ is a weighting factor (typically between 0 and 1) [S1, unofficial].
3. **Group Assignment:** Assign each Gaussian $i$ to the group $j$ that minimizes the similarity score $S_{i,j}$ (i.e., the most similar control trajectory) [S1, unofficial]. This partitions the Gaussians into groups $\mathcal{M}^j$.
4. **SE(3) Trajectory Fitting:** For each group $j$, fit a group-wise SE(3) trajectory by optimizing the parameters $\{h_j^0, R_j^t, T_j^t\}$ jointly with the scene reconstruction. The deformation of any Gaussian $\mu_i \in \mathcal{M}^j$ at timestep $t$ is predicted by:
   $$
   \mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
   $$
   where $\mu_i^0$ is the initial mean position of Gaussian $i$, $h_j^0$ is the initial position of the control trajectory, $R_j^t$ is the rotation matrix, and $T_j^t$ is the translation vector [S1, unofficial].
5. **Rendering:** During rendering, apply the shared SE(3) transformation for the assigned group to each Gaussian instead of evaluating a per-Gaussian neural network [S1, unofficial].

**Parameters:**
- $\lambda_r$: Weighting factor for trajectory similarity. Not given in the sources.
- $J$: Number of control trajectories/groups. Not given in the sources.
- $F$: Number of timesteps. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The SE(3) parameters must be optimized carefully to ensure smooth motion. The distillation process regularizes the motion field, smoothing noisy trajectories [S1, unofficial].
- **Boundaries:** Farthest point sampling is used for initialization, which helps in covering the scene extent but may require sufficient $J$ to capture complex local motions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def groupflow_assign_groups(means_sequence, control_positions, lambda_r):
    """
    Assigns Gaussians to groups based on trajectory similarity.
    
    Args:
        means_sequence: np.array of shape (N, F, 3), mean positions for N Gaussians over F timesteps.
        control_positions: np.array of shape (J, F, 3), positions of J control trajectories over F timesteps.
        lambda_r: float, weighting factor for std vs mean in similarity score.
        
    Returns:
        group_assignments: np.array of shape (N,), group index for each Gaussian.
    """
    N, F, _ = means_sequence.shape
    J, _, _ = control_positions.shape
    
    # Initialize similarity scores
    # S[i, j] stores the score for Gaussian i and control trajectory j
    S = np.zeros((N, J))
    
    for i in range(N):
        for j in range(J):
            # Compute distance over time
            dists = np.linalg.norm(means_sequence[i] - control_positions[j], axis=1) # Shape (F,)
            
            # Compute std and mean over time
            std_t = np.std(dists)
            mean_t = np.mean(dists)
            
            # Compute similarity score
            S[i, j] = lambda_r * std_t + (1 - lambda_r) * mean_t
            
    # Assign to the group with the minimum score
    group_assignments = np.argmin(S, axis=1)
    return group_assignments

def groupflow_predict_deformation(means_initial, control_initial, rotation_matrices, translation_vectors, group_assignments, t):
    """
    Predicts the mean position of a Gaussian at timestep t using SE(3) transformation.
    
    Args:
        means_initial: np.array of shape (N, 3), initial mean positions.
        control_initial: np.array of shape (J, 3), initial control positions.
        rotation_matrices: np.array of shape (J, F, 3, 3), rotation matrices for each group and timestep.
        translation_vectors: np.array of shape (J, F, 3), translation vectors for each group and timestep.
        group_assignments: np.array of shape (N,), group index for each Gaussian.
        t: int, timestep to predict.
        
    Returns:
        predicted_means: np.array of shape (N, 3), predicted mean positions at timestep t.
    """
    N = means_initial.shape[0]
    predicted_means = np.zeros((N, 3))
    
    for i in range(N):
        j = group_assignments[i]
        mu_i_0 = means_initial[i]
        h_j_0 = control_initial[j]
        R_j_t = rotation_matrices[j, t]
        T_j_t = translation_vectors[j, t]
        
        # Apply SE(3) transformation
        # mu_i^t = R_j^t (mu_i^0 - h_j^0) + h_j^0 + T_j^t
        predicted_means[i] = R_j_t @ (mu_i_0 - h_j_0) + h_j_0 + T_j_t
        
    return predicted_means
```

## Operating characteristics
- **Time Complexity:** Reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group), where $J \ll N$ [S1, unofficial].
- **Memory Complexity:** Stores $J$ sets of SE(3) parameters instead of $N$ neural network weights or deformation fields [S1, unofficial].
- **Measured Performance:** On the MonoDyGauBench, integrating GroupFlow with Temporal Sensitivity Pruning (TSP) and Temporal Sensitivity Sampling (TSS) accelerates rendering by 13.71× and reduces training time by 2.53× compared to baselines [S1, unofficial].
- **Strengths:** Substantial acceleration during training and rendering; maintains temporally coherent, high-quality motion reconstruction; regularizes the learned motion field [S1, unofficial].
- **Limitations:** Requires grouping, which may not capture highly heterogeneous motion if $J$ is too small [S1, unofficial].
- **When to use:** When rendering speed is critical and motion is locally coherent [S1, unofficial].
- **When not to use:** When motion is highly complex and heterogeneous, requiring per-Gaussian deformation [S1, unofficial].

## Background
GroupFlow was introduced in "SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping" [S1, unofficial]. It is part of a pipeline that includes Temporal Sensitivity Pruning (TSP) and Temporal Sensitivity Sampling (TSS) to optimize dynamic 3D Gaussian Splatting. The method distills per-Gaussian neural motion fields into shared SE(3) transformations to reduce computational cost [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
