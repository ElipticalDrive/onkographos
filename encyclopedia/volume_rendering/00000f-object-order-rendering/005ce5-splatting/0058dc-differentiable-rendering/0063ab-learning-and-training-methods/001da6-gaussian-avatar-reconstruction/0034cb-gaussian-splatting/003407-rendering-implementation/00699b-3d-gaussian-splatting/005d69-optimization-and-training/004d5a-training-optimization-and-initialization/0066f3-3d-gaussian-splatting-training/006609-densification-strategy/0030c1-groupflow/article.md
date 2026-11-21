# GroupFlow

Also called: Groupwise SE(3) Motion Distillation, Temporal Motion Grouping

## Overview
GroupFlow is a motion representation technique for dynamic 3D Gaussian Splatting that replaces per-Gaussian neural deformation fields with shared SE(3) transformations for groups of Gaussians [S1, unofficial]. It solves the computational bottleneck of evaluating individual deformation networks for every primitive at every timestep, accelerating rendering and training while maintaining temporal coherence [S1, unofficial]. Unlike methods that retain dense neural fields or use global rigid transforms, GroupFlow captures locally coherent motion by clustering Gaussians based on trajectory similarity and fitting a separate SE(3) trajectory to each cluster [S1, unofficial].

## How it works
GroupFlow operates on a dynamic scene represented by a set of 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each Gaussian $g_i$ has a time-varying mean position $\mu_i^t$, rotation $r_i^t$, and scale $s_i^t$ [S1, unofficial]. The method distills the learned deformation field into grouped SE(3) transformations [S1, unofficial].

**Inputs:**
*   Set of $N$ Gaussians.
*   Number of timesteps $F$ (indices $t=0, \dots, F-1$).
*   Number of control trajectories (groups) $J$.
*   Weighting factor $\lambda_r$.

**Outputs:**
*   For each group $j \in \{1, \dots, J\}$:
    *   Initial control point position $h_j^0 \in \mathbb{R}^3$.
    *   Time-varying rotation matrices $R_j^t \in \mathbb{R}^{3 \times 3}$ for $t=0, \dots, F-1$.
    *   Time-varying translation vectors $T_j^t \in \mathbb{R}^3$ for $t=0, \dots, F-1$.
*   Assignment of each Gaussian $i$ to a group $j$.

**Data Structures:**
*   **Gaussian Trajectories:** Each Gaussian $i$ is represented as a sequence of mean positions $\mathcal{M}_i = \{\mu_i^t\}_{t=0}^{F-1}$ [S1, unofficial].
*   **Control Trajectories:** A set of $J$ control points $\{h_j^t\}_{t=0}^{F-1}$ for $j=1, \dots, J$ [S1, unofficial].

**Steps:**

1.  **Initialization:** Initialize $J$ control trajectories $\{h_j^0\}_{j=1}^J$ at $t=0$ using farthest point sampling on the initial mean positions $\{\mu_i^0\}_{i=1}^N$ [S1, unofficial].

2.  **Trajectory Similarity Calculation:** For each Gaussian $i$ and each control trajectory $j$, compute the similarity score $S_{i,j}$ using the temporal mean and standard deviation of the Euclidean distance between the Gaussian's trajectory and the control trajectory's trajectory [S1, unofficial]:
    $$
    S_{i,j} = \lambda_r \, \mathrm{std}_t(\| \mu_i^t - h_j^t \|) + (1-\lambda_r) \, \mathrm{mean}_t(\| \mu_i^t - h_j^t \|)
    $$
    where:
    *   $\mu_i^t$ is the mean position of Gaussian $i$ at time $t$.
    *   $h_j^t$ is the position of control trajectory $j$ at time $t$.
    *   $\| \cdot \|$ denotes the Euclidean norm.
    *   $\mathrm{mean}_t(\cdot)$ computes the average over $t=0, \dots, F-1$.
    *   $\mathrm{std}_t(\cdot)$ computes the standard deviation over $t=0, \dots, F-1$.
    *   $\lambda_r \in [0, 1]$ is a weighting factor balancing the importance of trajectory consistency (std) versus average proximity (mean) [S1, unofficial].

3.  **Group Assignment:** Assign each Gaussian $i$ to the group $j$ that minimizes the similarity score $S_{i,j}$ (or maximizes similarity, depending on convention; the source implies assignment based on this score) [S1, unofficial]. Let $\mathcal{M}^j$ be the set of Gaussians assigned to group $j$.

4.  **SE(3) Trajectory Fitting:** For each group $j$, fit a group-wise SE(3) trajectory by optimizing the parameters $\{h_j^0, R_j^t, T_j^t\}$ jointly with the scene [S1, unofficial]. The deformation of any Gaussian $\mu_i \in \mathcal{M}^j$ at timestep $t$ is predicted by applying the learned SE(3) transformation relative to its control point:
    $$
    \mu_i^t = R_j^t (\mu_i^0 - h_j^0) + h_j^0 + T_j^t
    $$
    where:
    *   $\mu_i^0$ is the initial mean position of Gaussian $i$.
    *   $h_j^0$ is the initial position of control trajectory $j$.
    *   $R_j^t$ is the rotation matrix for group $j$ at time $t$.
    *   $T_j^t$ is the translation vector for group $j$ at time $t$ [S1, unofficial].

5.  **Rendering/Training:** During rendering or training, instead of querying a neural network for each Gaussian's deformation, apply the corresponding group's SE(3) transformation to the initial mean $\mu_i^0$ to obtain $\mu_i^t$ [S1, unofficial].

**Parameters:**
*   $J$: Number of groups/control trajectories. Not given in the sources.
*   $\lambda_r$: Weighting factor for trajectory similarity. Not given in the sources.
*   $F$: Number of timesteps. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Farthest Point Sampling:** The initialization of control points via farthest point sampling at $t=0$ is critical for covering the scene [S1, unofficial].
*   **Temporal Coherence:** The method regularizes the learned motion field by smoothing noisy trajectories, which helps in real-world dynamic scenes [S1, unofficial].
*   **Precision:** The SE(3) transformation assumes rigid body motion within each group. If a group contains Gaussians with non-rigid relative motion, the approximation may introduce artifacts.

## Operating characteristics
*   **Time Complexity:** Reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group), where $J \ll N$ [S1, unofficial].
*   **Memory Complexity:** Stores $J$ sets of SE(3) parameters instead of $N$ neural network weights or deformation fields [S1, unofficial].
*   **Performance:** On the MonoDyGauBench dataset, integrating GroupFlow into DeformableGS accelerates rendering by 13.71× and shortens training by 2.53× compared to baselines [S1, unofficial].
*   **Strengths:** Substantial acceleration in training and rendering; maintains temporally coherent, high-quality motion reconstruction; distills dense deformations into a smaller set of parameters [S1, unofficial].
*   **Limitations:** Assumes locally coherent rigid motion within groups; may not capture complex non-rigid deformations if groups are too large or poorly defined [S1, unofficial].

## Background
GroupFlow was introduced in the paper "SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping" [S1, unofficial]. It is part of a pipeline that also includes Temporal Sensitivity Pruning (TSP) and Temporal Sensitivity Sampling (TSS) to further optimize dynamic 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
