# Hierarchical Submap Alignment

Also called: HIGS Alignment, Hierarchical Implicit Grids Submap Alignment

## Overview
Hierarchical Submap Alignment (HIGS) is a global alignment and fusion method that eliminates estimation drift in locally optimized submaps to achieve globally consistent neural fields [S1, unofficial]. It sets itself apart from sibling methods like iMAP and RNR-Map Construction by using a hierarchical feature grid to progressively align submaps from coarse to fine levels, making it robust to severe initial misalignments where SDF-only methods fail [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $n_s$ locally optimized submaps. Each submap $u$ is represented as a hierarchical feature grid. Initial noisy base poses $\{T^{bu}_w\}_u$ for each submap in the world frame.
- **Outputs:** Optimized global base poses $\{T^{uw}_u\}_u$ for all submaps.

**Data Structures:**
- **Hierarchical Feature Grid:** Each submap $u$ contains features at multiple levels $l=1, \dots, L$. The features $f_{1:l}^u$ are concatenated features up to level $l$.
- **Overlap Indices:** For each pair of submaps $(u, v)$, the set $I_{luv}$ contains indices of level-$l$ vertices in submap $u$ that lie within the overlap region with submap $v$.
- **Vertex Positions:** $z_{l,i}$ denotes the 3D position of the $i$-th vertex at level $l$.

**Parameters:**
- **Iteration Allocation:** $k_{f,l}$ iterations for feature alignment at level $l$, and $k_s$ iterations for final prediction-based alignment. Typical values: $k_{f,1}=45, k_{f,2}=45, k_s=10$ [S1, unofficial].
- **Trust-Region Radius ($\tau$):** Scaled based on the initial submap alignment error to regularize pose updates [S1, unofficial].
- **Feature Levels ($L$):** Determined by the grid resolution and feature dimension settings (e.g., Section 4.1.3 in source) [S1, unofficial].

**Algorithm Steps:**
1. **Initialize:** Set initial submap poses $\{T^{uw}_u\}_u$ to the noisy base poses $\{T^{bu}_w\}_u$ [S1, unofficial].
2. **Hierarchical Feature Alignment:** For each level $l$ from 1 to $L$:
   - Update $\{T^{uw}_u\}_u$ by solving the feature-based alignment problem for $k_{f,l}$ iterations.
   - The optimization minimizes the cost function $c_{feat}^l(T_u, T_v)$ plus trust-region regularization:
     $$ \min_{\{T_u\}_u \subset SE(3)} \sum_{(u,v) \in E} w_{uv} c_{feat}^l(T_u, T_v) + \sum_{u=1}^{n_s} \rho(T^{bu}_w, T_u) $$
     where $E$ is the set of edges between overlapping submaps, $w_{uv}$ are weights, and $\rho$ is the trust-region regularization term [S1, unofficial].
   - The feature cost $c_{feat}^l$ is defined as:
     $$ c_{feat}^l(T_u, T_v) = \sum_{i \in I_{luv}} d(f_{1:l}^u(z_{l,i}), f_{1:l}^v((T_{vw})^{-1} T_u z_{l,i})) $$
     where $d$ is a distance metric (e.g., L1 or cosine distance) [S1, unofficial].
3. **Final Prediction-Based Alignment:** After processing all feature levels, update $\{T^{uw}_u\}_u$ by solving the prediction-based alignment problem for $k_s$ iterations. This step uses SDF or vision-language feature predictions to refine the alignment [S1, unofficial].
4. **Return:** The optimized poses $\{T^{uw}_u\}_u$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_submap_alignment(submaps, initial_poses, levels, k_f, k_s, tau):
    """
    submaps: List of submap objects, each containing hierarchical feature grids.
    initial_poses: List of initial SE(3) poses for each submap.
    levels: List of feature levels [1, 2, ..., L].
    k_f: List of iterations for each feature level.
    k_s: Number of iterations for final alignment.
    tau: Trust-region radius.
    """
    # Initialize poses
    current_poses = list(initial_poses)
    n_submaps = len(submaps)
    
    # Step 1: Hierarchical Feature Alignment
    for l, level in enumerate(levels):
        iterations = k_f[l]
        for _ in range(iterations):
            # Compute feature-based cost and gradients for all overlapping pairs
            # This involves optimizing the poses to minimize the feature distance
            # in the overlap regions, regularized by the trust region.
            current_poses = optimize_feature_alignment(
                submaps, current_poses, level, tau
            )
            
    # Step 2: Final Prediction-Based Alignment
    for _ in range(k_s):
        # Optimize using SDF or vision-language feature predictions
        current_poses = optimize_prediction_alignment(
            submaps, current_poses, tau
        )
        
    return current_poses

def optimize_feature_alignment(submaps, poses, level, tau):
    """
    Optimizes poses to minimize feature distance at a specific level.
    Uses PyTorch optimization as per source.
    """
    # Define cost function based on equation (29)
    # c_feat_l(Tu, Tv) = sum_{i in I_luv} d(f_{1:l}^u(z_l,i), f_{1:l}^v((Tvw)^-1 Tu z_l,i))
    # Plus trust region regularization rho(Tbu, Tu)
    
    # Initialize gradients for poses
    # Perform gradient descent/optimization step
    # Return updated poses
    pass # Implementation details depend on specific optimizer setup

def optimize_prediction_alignment(submaps, poses, tau):
    """
    Optimizes poses using SDF or vision-language feature predictions.
    Based on equation (31).
    """
    # Define cost function based on prediction distance
    # Plus trust region regularization
    # Perform gradient descent/optimization step
    # Return updated poses
    pass # Implementation details depend on specific optimizer setup
```

**Edge Cases and Pitfalls:**
- **Initial Error Sensitivity:** SDF-only prediction methods (like MIPS) are highly sensitive to initial guess errors. HIGS mitigates this by using hierarchical features [S1, unofficial].
- **Trust-Region Scaling:** The trust-region radius $\tau$ must be scaled appropriately to the initial error to ensure stable convergence [S1, unofficial].
- **Overlap Detection:** Accurate identification of overlap regions ($I_{luv}$) is critical for the feature cost calculation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of submaps, overlap size, and feature levels. In experiments on ScanNet scene 0011 (159 poses, 5 submaps), HIGS(SDF) took 12.15s, HIGS(CLIP) took 6.29s, and HIGS(SDF+CLIP) took 21.22s [S1, unofficial].
- **Memory Complexity:** Requires storing hierarchical feature grids for all submaps and overlap indices.
- **Strengths:** Robust to severe misalignments (e.g., 17 deg rotation, 0.5 m translation errors) compared to MIPS and VFPP [S1, unofficial].
- **Limitations:** Requires hierarchical feature grids, which may increase memory usage. Performance depends on the quality of local SLAM initialization.
- **When to Use:** When local SLAM produces submaps with significant drift or misalignment, and global consistency is required for neural field fusion.
- **When Not to Use:** If computational resources are extremely limited and initial alignment is already very accurate.

## Background
Hierarchical Submap Alignment was introduced in the HIGS (Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding) paper by Cao et al. [S1, unofficial]. It addresses the problem of global alignment and fusion of local submaps in neural scene representation, particularly in the context of joint geometric and semantic understanding. The method builds on the MIPS-Fusion approach for submap splitting [S1, unofficial].

## Sources
[S1] HIGS: Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding, https://doi.org/10.48550/arxiv.2609.38620, fetched 2026-10-26 (unofficial)
