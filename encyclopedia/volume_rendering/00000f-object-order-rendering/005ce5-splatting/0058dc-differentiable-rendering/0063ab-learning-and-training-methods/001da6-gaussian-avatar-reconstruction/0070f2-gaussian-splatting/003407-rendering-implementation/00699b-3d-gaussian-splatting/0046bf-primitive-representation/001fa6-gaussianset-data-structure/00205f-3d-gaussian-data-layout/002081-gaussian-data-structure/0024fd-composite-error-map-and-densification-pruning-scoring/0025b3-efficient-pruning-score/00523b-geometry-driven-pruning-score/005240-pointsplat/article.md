# PointSplat


## Overview
PointSplat is a geometry-driven prune-and-refine framework for 3D Gaussian Splatting that reconstructs compact Gaussian sets without requiring per-scene optimization at deployment [S1, unofficial]. It bridges Gaussian pruning and transformer refinement by ranking Gaussians using only intrinsic 3D attributes and refining them via a dual-branch encoder [S1, unofficial]. Unlike image-driven pruning methods (e.g., LightGaussians, PUP-3DGS, Speedy-Splat) that require 2D images and per-scene optimization, PointSplat operates solely on 3D geometry [S1, unofficial]. Unlike SplatFormer, which concatenates all parameters into a single vector causing appearance dominance from high-dimensional spherical harmonics, PointSplat decouples geometry and appearance to prevent feature imbalance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ 3D Gaussians $\mathcal{G} = \{G_i\}_{i=1}^N$ from a trained 3DGS model [S1, unofficial]. Each Gaussian $G_i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$, opacity $\alpha_i \in \mathbb{R}$, and spherical harmonics coefficients $\gamma^{(i)}$ [S1, unofficial].
- Output: A refined, compact subset of Gaussians with improved structural integrity and visual quality [S1, unofficial].

**Data Structures:**
Each Gaussian $G_i$ retains its standard 3DGS parameters:
- $\mu_i$: Mean position (3 floats) [S1, unofficial].
- $\Sigma_i$: Covariance matrix, decomposed into rotation $R_i$ (from quaternion $q_i \in \mathbb{R}^4$) and scaling $S_i$ (from vector $s_i = [s_{xi}, s_{yi}, s_{zi}]$) [S1, unofficial].
- $\alpha_i$: Opacity (1 float) [S1, unofficial].
- $\gamma^{(i)}$: Spherical harmonics coefficients [S1, unofficial]. For degree $L=3$, this results in up to 48 dimensions for appearance coefficients [S1, unofficial].

**Method Steps:**

1. **Geometry-Driven Pruning Score Calculation:**
   Compute a pruning score for each Gaussian $G_i$ based solely on intrinsic 3D attributes (opacity and volume/spatial coverage) [S1, unofficial]. The score balances spatial coverage and opacity to select a compact yet informative subset [S1, unofficial]. This avoids complex image-ray interactions [S1, unofficial].
   - Formula for Gaussian definition:
     $$ G_i(x) \propto \exp\left(-\frac{1}{2} (x - \mu_i)^T \Sigma_i^{-1} (x - \mu_i)\right) $$
     where $\Sigma_i = R_i S_i S_i^T R_i^T$ [S1, unofficial].
   - Select the top $K$ Gaussians with the highest scores [S1, unofficial].

2. **Dual-Branch Encoder Initialization:**
   Separate the geometric attributes ($\mu_i, R_i, S_i$) and appearance attributes ($\alpha_i, \gamma^{(i)}$) of the retained Gaussians into two distinct branches [S1, unofficial]. This explicit separation prevents feature imbalance where high-dimensional appearance features (e.g., 48D SH coefficients for $L=3$) dominate lower-dimensional geometric features [S1, unofficial].

3. **Feature Reweighting with Positional Encoding:**
   Reweight the separated geometric and appearance features using positional encoding [S1, unofficial]. This step ensures balanced learning and prevents appearance dominance during transformer refinement [S1, unofficial].

4. **Transformer Refinement:**
   Process the reweighted features through a 3D transformer network (based on PointTransformerV3 architecture) to globally coordinate Gaussian parameters [S1, unofficial]. This captures fine-grained structural details, addressing artifacts such as blurred textures and broken geometric edges that may result from pruning alone [S1, unofficial]. The output is a set of compact, well-aligned ellipsoids [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import List

class Gaussian:
    def __init__(self, mu: np.ndarray, q: np.ndarray, s: np.ndarray, alpha: float, sh_coeffs: np.ndarray):
        self.mu = mu  # (3,) mean position
        self.q = q    # (4,) quaternion
        self.s = s    # (3,) scaling vector
        self.alpha = alpha
        self.sh_coeffs = sh_coeffs  # (D,) spherical harmonics coefficients

    def get_covariance(self) -> np.ndarray:
        R = quaternion_to_rotation_matrix(self.q)
        S = np.diag(self.s)
        return R @ S @ S.T @ R.T

    def compute_pruning_score(self) -> float:
        # Score based on intrinsic 3D attributes: opacity and volume [S1, unofficial]
        volume = np.prod(self.s)
        return self.alpha * volume

def point_splat(gaussians: List[Gaussian], k: int) -> List[Gaussian]:
    # Step 1: Geometry-Driven Pruning
    scores = [g.compute_pruning_score() for g in gaussians]
    sorted_indices = np.argsort(scores)[::-1][:k]
    retained_gaussians = [gaussians[i] for i in sorted_indices]

    # Step 2 & 3: Dual-Branch Encoder and Feature Reweighting
    geometric_features = []
    appearance_features = []

    for g in retained_gaussians:
        geo_feat = np.concatenate([g.mu, g.q, g.s])
        geometric_features.append(geo_feat)
        app_feat = np.concatenate([[g.alpha], g.sh_coeffs])
        appearance_features.append(app_feat)

    geometric_features = np.array(geometric_features)
    appearance_features = np.array(appearance_features)

    # Apply positional encoding and reweighting [G]
    pos_encoding = positional_encoding(geometric_features[:, :3])
    geometric_features = geometric_features + pos_encoding

    # Normalize features to prevent dominance [G]
    geometric_features = geometric_features / (np.linalg.norm(geometric_features, axis=1, keepdims=True) + 1e-8)
    appearance_features = appearance_features / (np.linalg.norm(appearance_features, axis=1, keepdims=True) + 1e-8)

    # Step 4: Transformer Refinement
    # Note: The exact transformer architecture and forward pass are not specified in the sources [S1, unofficial]
    combined_features = np.concatenate([geometric_features, appearance_features], axis=1)
    # refined_features = transformer_refine(combined_features)

    # Extract refined parameters
    refined_gaussians = []
    for i, g in enumerate(retained_gaussians):
        # Note: Mapping refined features back to quaternion/scale/etc. is not specified in the sources [S1, unofficial]
        # Assuming identity mapping for structural completeness
        refined_g = Gaussian(
            mu=geometric_features[i, :3],
            q=geometric_features[i, 3:7],
            s=geometric_features[i, 7:10],
            alpha=appearance_features[i, 0],
            sh_coeffs=appearance_features[i, 1:]
        )
        refined_gaussians.append(refined_g)

    return refined_gaussians

def quaternion_to_rotation_matrix(q: np.ndarray) -> np.ndarray:
    w, x, y, z = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])

def positional_encoding(features: np.ndarray) -> np.ndarray:
    # Standard positional encoding using sine/cosine functions [G]
    scales = np.logspace(-1, 4, features.shape[1])
    scaled_features = features[:, None, :] * scales[None, :, None]
    return np.sin(scaled_features)
```

**Edge Cases and Pitfalls:**
- **Feature Imbalance:** If geometry and appearance are not properly separated and reweighted, high-dimensional spherical harmonics coefficients (up to 48D for $L=3$) can dominate the learning process, leading to suboptimal refinement [S1, unofficial].
- **Degenerate Input:** Pruning based solely on opacity and volume may remove Gaussians critical for fine details if their opacity is low but they cover unique spatial regions [S1, unofficial]. The transformer refinement helps mitigate this by globally coordinating parameters [S1, unofficial].
- **Deployment Constraints:** PointSplat is designed to eliminate reliance on per-scene optimization at deployment, so it trades absolute peak quality for rapid compression and inference efficiency [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Pruning is $O(N)$ to compute scores and select top $K$ [S1, unofficial]. Transformer refinement depends on the transformer architecture but is generally efficient for compact sets [S1, unofficial].
- **Memory Complexity:** Reduced by pruning $N$ to $K$ Gaussians [S1, unofficial]. No additional per-scene optimization memory overhead is incurred [S1, unofficial].
- **Measured Performance:** Experiments on ScanNet++ and Replica datasets demonstrate competitive rendering quality and superior efficiency across varying sparsity levels [S1, unofficial].
- **Strengths:** No reliance on 2D images for pruning [S1, unofficial]. No per-scene optimization required [S1, unofficial]. Avoids feature imbalance through dual-branch encoder [S1, unofficial].
- **Limitations:** May not achieve the same quality as methods with per-scene optimization [S1, unofficial]. Relies on intrinsic attributes which may not capture all visual importance [S1, unofficial].
- **When to Use:** When efficient compression of 3DGS models is needed without per-scene optimization, and when feature imbalance is a concern [S1, unofficial].

## Background
PointSplat was introduced in the paper "PointSplat: Efficient Geometry-Driven Pruning and Transformer Refinement for 3D Gaussian Splatting" [S1, unofficial]. It addresses limitations in existing 3DGS compression methods, which either rely on image-driven pruning with per-scene optimization or suffer from feature imbalance in transformer-based refinement [S1, unofficial]. The method builds on the 3D Gaussian Splatting framework [S1, unofficial] and integrates concepts from point cloud processing refinements [S1, unofficial].

## Sources
[S1] PointSplat: Efficient Geometry-Driven Pruning and Transformer Refinement for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2604.09903, fetched 2026-10-29 (unofficial)
