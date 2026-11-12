# GroupFlow

Also called: Groupwise SE(3) Motion Distillation

## Overview
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that distills per-Gaussian neural deformation fields into shared SE(3) transformations for groupwise motion [S1, unofficial]. It solves the computational bottleneck of evaluating individual neural networks for every Gaussian primitive at every timestep, reducing rendering and training costs by replacing $N$ per-Gaussian transformations with $J$ per-group transformations [S1, unofficial]. It sets itself apart from sibling methods like [Gaussian Densification](4314-gaussian-densification.md) by focusing on temporal motion efficiency rather than spatial density management, though it often operates in conjunction with densification strategies to maintain scene fidelity.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ dynamic 3D Gaussians $\mathcal{G}$, each with a sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ across $F$ timesteps.
*   **Output:** $J$ control trajectories $\{h_j^t\}_{j=0}^{J-1}$ and associated SE(3) parameters $\{h_j^0, R_j^t, T_j^t\}$ that define the motion for each group.

**Data Structures:**
*   **Gaussian Means:** A tensor of shape $(N, F, 3)$ storing $\mu_i^t$.
*   **Control Trajectories:** A tensor of shape $(J, F, 3)$ storing $h_j^t$.
*   **Group Assignments:** A vector of length $N$ mapping each Gaussian index $i$ to a group index $j$.

**Steps:**
1.  **Initialization:** Initialize $J$ control trajectories $\{h_j^0\}_{j=0}^{J-1}$ at $t=0$ using Farthest Point Sampling (FPS) on the initial Gaussian means $\{\mu_i^0\}_{i=0}^{N-1}$ [S1, unofficial].
2.  **Similarity Scoring:** For each Gaussian $i$ and control trajectory $j$, compute the trajectory similarity score $S_{i,j}$ using the formula:
    $$
    S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
    $$
    where $\lambda_r$ is a weighting factor (typically between 0 and 1), $\mathrm{std}_t$ computes the standard deviation over time $t$, and $\mathrm{mean}_t$ computes the mean over time $t$ [S1, unofficial].
3.  **Group Assignment:** Assign each Gaussian $i$ to the group $j$ that minimizes the similarity score $S_{i,j}$ [S1, unofficial]. This partitions the Gaussians into $J$ groups $\mathcal{M}^j$.
4.  **SE(3) Fitting:** For each group $j$, fit a group-wise SE(3) trajectory by optimizing the parameters $\{h_j^0, R_j^t, T_j^t\}$ jointly with the scene. The deformation of any Gaussian $\mu_i \in \mathcal{M}^j$ at timestep $t$ is predicted by:
    $$
    \mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
    $$
    where $R_j^t$ is the rotation matrix and $T_j^t$ is the translation vector for group $j$ at time $t$ [S1, unofficial].
5.  **Optimization:** The shared flow parameters $\{h_j^0, R_j^t, T_j^t\}$ are optimized end-to-end alongside the scene representation, regularizing the motion field and smoothing noisy trajectories [S1, unofficial].

**Parameters:**
*   $J$: Number of control trajectories/groups. Not given in the sources.
*   $\lambda_r$: Weighting factor for the similarity score. Not given in the sources.
*   $F$: Number of timesteps. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The SE(3) parameters must be maintained in a valid rotation space (e.g., via quaternion normalization or Rodrigues parameters) to ensure $R_j^t$ remains a valid rotation matrix [G].
*   **Boundary Conditions:** The formula assumes $\mu_i^0$ and $h_j^0$ are aligned at $t=0$. If the initial assignment is poor, the relative transformation may drift [G].

## Subtopics

### Temporal Sensitivity Pruning (TSP)
Removes low-impact Gaussians via temporally aggregated sensitivity analysis. Chosen when reducing primitive count is critical for speed. [Link to TSP article if available, otherwise note as sibling concept].

### Temporal Sensitivity Sampling (TSS)
Perturbs timestamps to suppress floaters and improve temporal coherence. Chosen when visual artifacts in motion are prevalent. [Link to TSS article if available, otherwise note as sibling concept].

## Operating characteristics
*   **Time Complexity:** Reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group), where $J \ll N$ [S1, unofficial].
*   **Memory Complexity:** Stores $J$ SE(3) trajectories instead of $N$ neural network weights or deformation fields [S1, unofficial].
*   **Measured Performance:** On the MonoDyGauBench dataset, integrating GroupFlow with TSP and TSS into DeformableGS accelerates rendering by 13.71× and reduces training time by 2.53× compared to baselines [S1, unofficial].
*   **Strengths:** Substantial acceleration in training and rendering; maintains temporally coherent, high-quality motion reconstruction; regularizes the learned motion field [S1, unofficial].
*   **Limitations:** Requires the assumption of locally coherent motion within groups; performance depends on the choice of $J$ [S1, unofficial].

## Background
GroupFlow was introduced in the context of dynamic 3D Gaussian Splatting to address the high computational cost of neural deformation fields. It is part of the SpeeDe3DGS framework, which also includes Temporal Sensitivity Pruning and Temporal Sensitivity Sampling [S1, unofficial]. The method distills dense, per-Gaussian neural deformations into a smaller set of groupwise SE(3) motions, reducing the number of transformations per frame from $N$ to $J$ [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
