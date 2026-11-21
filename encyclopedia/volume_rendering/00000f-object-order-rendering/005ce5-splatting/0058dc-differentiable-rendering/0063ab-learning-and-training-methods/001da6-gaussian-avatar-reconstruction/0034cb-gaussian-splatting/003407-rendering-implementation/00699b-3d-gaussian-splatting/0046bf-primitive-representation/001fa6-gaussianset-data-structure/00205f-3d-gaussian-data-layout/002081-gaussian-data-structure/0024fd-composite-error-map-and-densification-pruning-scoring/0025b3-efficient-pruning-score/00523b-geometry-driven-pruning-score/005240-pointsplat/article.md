# PointSplat

Also called: Geometry-Driven Pruning and Transformer Refinement for 3D Gaussian Splatting

## Overview
PointSplat is a geometry-driven prune-and-refine framework for 3D Gaussian Splatting (3DGS) that enables high-quality reconstruction of compact Gaussian sets without requiring per-scene optimization at deployment [S1, unofficial]. It bridges the gap between Gaussian pruning and transformer refinement by using intrinsic 3D attributes for pruning and a dual-branch encoder for refinement [S1, unofficial]. Unlike image-driven pruning methods (e.g., LightGaussians, PUP-3DGS, Speedy-Splat) that rely on 2D images and per-scene optimization, PointSplat uses only 3D attributes for pruning [S1, unofficial]. Unlike SplatFormer, which concatenates all Gaussian parameters into a single feature vector causing appearance dominance due to high-dimensional spherical harmonics, PointSplat decouples geometry and appearance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ 3D Gaussians $\mathcal{G} = \{G_i\}_{i=1}^N$ from a trained 3DGS model. Each Gaussian $G_i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$, opacity $\alpha_i \in \mathbb{R}$, and spherical harmonics coefficients $\gamma^{(i)}$ [S1, unofficial].
- Output: A refined, compact subset of Gaussians with improved structural integrity and visual quality [S1, unofficial].

**Data Structures:**
Each Gaussian $G_i$ retains its standard 3DGS parameters:
- $\mu_i$: Mean position (3 floats).
- $\Sigma_i$: Covariance matrix, decomposed into rotation $R_i$ (from quaternion $q_i \in \mathbb{R}^4$) and scaling $S_i$ (from vector $s_i = [s_{xi}, s_{yi}, s_{zi}]$) [S1, unofficial].
- $\alpha_i$: Opacity (1 float).
- $\gamma^{(i)}$: Spherical harmonics coefficients. For degree $L=3$, this results in up to 48 dimensions for appearance coefficients [S1, unofficial].

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
from typing import List, Tuple

class Gaussian:
    def __init__(self, mu: np.ndarray, q: np.ndarray, s: np.ndarray, alpha: float, sh_coeffs: np.ndarray):
        self.mu = mu  # (3,)
        self.q = q    # (4,) quaternion
        self.s = s    # (3,) scaling vector
        self.alpha = alpha
        self.sh_coeffs = sh_coeffs  # (D,) e.g., 48 for L=3

    def get_covariance(self) -> np.ndarray:
        # Construct rotation matrix from quaternion
        R = quaternion_to_rotation_matrix(self.q)
        S = np.diag(self.s)
        return R @ S @ S.T @ R.T

    def compute_pruning_score(self) -> float:
        # Score based on intrinsic 3D attributes: opacity and volume
        # Volume is proportional to product of scaling factors
        volume = np.prod(self.s)
        # Simple heuristic: balance opacity and volume
        # Note: Exact formula not specified in sources; using product as proxy
        return self.alpha * volume

def point_splat(gaussians: List[Gaussian], k: int) -> List[Gaussian]:
    """
    PointSplat: Geometry-Driven Pruning and Transformer Refinement.
    
    Args:
        gaussians: List of N Gaussians from trained 3DGS.
        k: Number of Gaussians to retain after pruning.
        
    Returns:
        Refined list of K Gaussians.
    """
    # Step 1: Geometry-Driven Pruning
    scores = [g.compute_pruning_score() for g in gaussians]
    # Select top K Gaussians
    sorted_indices = np.argsort(scores)[::-1][:k]
    retained_gaussians = [gaussians[i] for i in sorted_indices]
    
    # Step 2 & 3: Dual-Branch Encoder and Feature Reweighting
    # Separate geometric and appearance features
    geometric_features = []
    appearance_features = []
    
    for g in retained_gaussians:
        # Geometric: position, rotation (quaternion), scale
        geo_feat = np.concatenate([g.mu, g.q, g.s])
        geometric_features.append(geo_feat)
        
        # Appearance: opacity, spherical harmonics
        app_feat = np.concatenate([[g.alpha], g.sh_coeffs])
        appearance_features.append(app_feat)
    
    geometric_features = np.array(geometric_features)
    appearance_features = np.array(appearance_features)
    
    # Apply positional encoding and reweighting
    # Note: Exact reweighting formula not specified in sources [G]
    # Assuming standard positional encoding for 3D points
    pos_encoding = positional_encoding(geometric_features[:, :3])
    geometric_features = geometric_features + pos_encoding
    
    # Normalize or scale features to prevent dominance [G]
    geometric_features = normalize(geometric_features)
    appearance_features = normalize(appearance_features)
    
    # Step 4: Transformer Refinement
    # Concatenate reweighted features for transformer input
    # Note: Transformer architecture details (e.g., number of layers, heads) not specified [G]
    combined_features = np.concatenate([geometric_features, appearance_features], axis=1)
    
    # Process through transformer
    refined_features = transformer_refine(combined_features)
    
    # Extract refined parameters
    refined_gaussians = []
    for i, g in enumerate(retained_gaussians):
        # Split refined features back into geometric and appearance
        # Note: Exact split indices depend on feature dimensions [G]
        geo_refined = refined_features[i, :geometric_features.shape[1]]
        app_refined = refined_features[i, geometric_features.shape[1]:]
        
        # Reconstruct Gaussian
        # Note: Mapping refined features back to quaternion/scale/etc. not specified [G]
        # Assuming identity mapping for simplicity in this pseudocode
        refined_g = Gaussian(
            mu=geo_refined[:3],
            q=geo_refined[3:7],
            s=geo_refined[7:10],
            alpha=app_refined[0],
            sh_coeffs=app_refined[1:]
        )
        refined_gaussians.append(refined_g)
        
    return refined_gaussians

def quaternion_to_rotation_matrix(q: np.ndarray) -> np.ndarray:
    # Standard quaternion to rotation matrix conversion
    w, x, y, z = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])

def positional_encoding(features: np.ndarray) -> np.ndarray:
    # Placeholder for positional encoding function
    # Typically involves sine/cosine functions of position
    return features

def normalize(features: np.ndarray) -> np.ndarray:
    # Standard L2 normalization per feature vector
    norms = np.linalg.norm(features, axis=1, keepdims=True)
    return features / (norms + 1e-8)

def transformer_refine(features: np.ndarray) -> np.ndarray:
    # Placeholder for transformer refinement
    # In practice, this would be a neural network forward pass
    return features
```

**Edge Cases and Pitfalls:**
- **Feature Imbalance:** If geometry and appearance are not properly separated and reweighted, high-dimensional spherical harmonics coefficients (up to 48D for $L=3$) can dominate the learning process, leading to suboptimal refinement [S1, unofficial].
- **Degenerate Input:** Pruning based solely on opacity and volume may remove Gaussians critical for fine details if their opacity is low but they cover unique spatial regions. The transformer refinement helps mitigate this by globally coordinating parameters [S1, unofficial].
- **Per-Scene Optimization:** PointSplat is designed to avoid per-scene optimization, so it may not achieve the same level of quality as methods that allow fine-tuning on specific scenes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Pruning is $O(N)$ to compute scores and select top $K$. Transformer refinement depends on the transformer architecture but is generally efficient for compact sets [S1, unofficial].
- **Memory Complexity:** Reduced by pruning $N$ to $K$ Gaussians. No additional per-scene optimization memory overhead [S1, unofficial].
- **Measured Performance:** Experiments on ScanNet++ and Replica datasets demonstrate competitive rendering quality and superior efficiency across varying sparsity levels [S1, unofficial].
- **Strengths:** No reliance on 2D images for pruning; no per-scene optimization required; avoids feature imbalance through dual-branch encoder [S1, unofficial].
- **Limitations:** May not achieve the same quality as methods with per-scene optimization; relies on intrinsic attributes which may not capture all visual importance [S1, unofficial].
- **When to Use:** When efficient compression of 3DGS models is needed without per-scene optimization, and when feature imbalance is a concern [S1, unofficial].
- **When Not to Use:** When maximum possible visual quality is required and per-scene optimization is acceptable [S1, unofficial].

## Background
PointSplat was introduced in the paper "PointSplat: Efficient Geometry-Driven Pruning and Transformer Refinement for 3D Gaussian Splatting" [S1, unofficial]. It addresses limitations in existing 3DGS compression methods, which either rely on image-driven pruning with per-scene optimization or suffer from feature imbalance in transformer-based refinement [S1, unofficial]. The method builds on the 3D Gaussian Splatting framework [S1, unofficial] and integrates concepts from point cloud processing refinements [S1, unofficial].

## Sources
[S1] PointSplat: Efficient Geometry-Driven Pruning and Transformer Refinement for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2604.09903, fetched 2026-10-29 (unofficial)
