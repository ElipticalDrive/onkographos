# VTC (Visibility-Aware Texture Consensus)

Also called: Visibility-Aware Texture Consensus

## Overview
VTC is a pre-training observation filtering and consensus mechanism designed to improve 3D Gaussian Splatting reconstruction in texture-degraded scenes by rejecting unreliable observations and decoupling geometry from appearance [S1, unofficial]. It differs from existing approaches that treat all visible observations as equally reliable by performing observation-level rejection of colors corrupted by occlusion, background mixing, or exposure differences, and by using a fixed robust consensus target computed before training rather than an online-updated uncertainty estimate [S1, unofficial].

## How it works
VTC operates on explicit Gaussian representations to determine visibility and depth consistency at the Gaussian granularity [S1, unofficial]. It constructs a cross-view color consensus and consensus weights from channel-wise median and Median Absolute Deviation (MAD) [S1, unofficial]. The consensus remains fixed during training to prevent feedback contamination [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   An initial set of $M$ 3D Gaussians $\mathcal{G} = \{\mathbf{g}_i\}_{i=1}^M$, where each Gaussian $\mathbf{g}_i$ has position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$ (appearance parameters) [S1, unofficial].
*   $N$ input images with known camera poses [S1, unofficial].

**Outputs:**
*   A binary mask $\mathbf{M}_{ij} \in \{0, 1\}$ indicating reliable observations for Gaussian $i$ in image $j$ [S1, unofficial].
*   A fixed color consensus vector $\mathbf{C}^{\text{cons}}_i$ for each Gaussian [S1, unofficial].
*   Consensus weights $\mathbf{W}_{ij}$ for each reliable observation [S1, unofficial].
*   Local PCA statistics for geometric anchoring [S1, unofficial].

### Data Structures
*   **Reliable Observation Set:** A sparse structure mapping $(i, j)$ pairs where $\mathbf{M}_{ij}=1$ [S1, unofficial].
*   **Color Consensus:** A tensor of shape $(M, 3)$ storing the channel-wise median color for each Gaussian [S1, unofficial].
*   **MAD:** A tensor of shape $(M, 3)$ storing the Median Absolute Deviation for each Gaussian [S1, unofficial].
*   **Consensus Weights:** A tensor of shape $(M, N)$ storing the weight for each Gaussian-image pair [S1, unofficial].

### Steps
1.  **Pre-training Filtering:** For each Gaussian $i$ and each input image $j$, compute the projected color $\mathbf{C}_{ij}$ and depth $D_{ij}$ [S1, unofficial]. Determine the reliability mask $\mathbf{M}_{ij}$ using four criteria [S1, unofficial]: (1) **Image Bounds:** The projection of Gaussian $i$ must lie within the image boundaries [S1, unofficial]. (2) **Foreground Membership:** The Gaussian must belong to the foreground object [S1, unofficial]. (3) **Projected Visibility:** The Gaussian must be occlusion-free in the view [S1, unofficial]. (4) **Depth Consistency:** The rendered depth $D_{ij}$ must be consistent with the expected depth from the camera pose and Gaussian geometry [S1, unofficial]. If any criterion fails, $\mathbf{M}_{ij} = 0$; otherwise, $\mathbf{M}_{ij} = 1$ [S1, unofficial].
2.  **Consensus Construction:** For each Gaussian $i$, collect all filtered observations $\{\mathbf{C}_{ij} \mid \mathbf{M}_{ij} = 1\}$ [S1, unofficial]. Compute the channel-wise median color $\mathbf{C}^{\text{cons}}_i$ across these observations [S1, unofficial]. This serves as the robust target color for Gaussian $i$ [S1, unofficial].
3.  **Weight Determination:** Compute the Median Absolute Deviation (MAD) for each color channel $k$ of Gaussian $i$:
    $$ \text{MAD}_{ik} = \text{median}_j \left( | \mathbf{C}_{ijk} - \mathbf{C}^{\text{cons}}_{ik} | \cdot \mathbf{M}_{ij} \right) $$
    where $\mathbf{C}_{ijk}$ is the $k$-th channel of $\mathbf{C}_{ij}$ [S1, unofficial]. Compute the consensus weight $\mathbf{W}_{ij}$ for each reliable observation. The weight is determined by the MAD and the effective sample count $K_i = \sum_j \mathbf{M}_{ij}$ [S1, unofficial]. A standard robust weighting formulation is:
    $$ \mathbf{W}_{ij} = \exp \left( - \frac{ \sum_k | \mathbf{C}_{ijk} - \mathbf{C}^{\text{cons}}_{ik} | }{ \epsilon + \sum_k \text{MAD}_{ik} + 1.0/K_i } \right) $$
    where $\epsilon$ is a small constant for numerical stability [G]. If $\mathbf{M}_{ij} = 0$, $\mathbf{W}_{ij} = 0$ [G].
4.  **Parameter Routing:** Restrict the parameter groups affected by each loss term to achieve geometry–appearance decoupling [S1, unofficial]. The VTC loss acts only on the appearance parameters $\mathbf{c}_i$ (spherical harmonic coefficients) [S1, unofficial]. Geometric parameters (position, covariance, opacity) are updated by other losses but are not directly influenced by the color consensus error [S1, unofficial].
5.  **Local PCA and Geometric Anchoring:** Estimate neighborhood structure for each Gaussian using local Principal Component Analysis (PCA) on the positions of neighboring Gaussians [S1, unofficial]. Compute the covariance matrix of neighbor positions [G]. Extract principal axes and normals [G]. Construct a geometric anchoring regularizer that imposes an adaptive prior in the orthogonal subspace according to structural confidence [S1, unofficial]. This regularizer uses a heavy-tailed generalized Gaussian prior [S1, unofficial].
6.  **Training Constraint:** The consensus $\mathbf{C}^{\text{cons}}_i$, MAD, and weights $\mathbf{W}_{ij}$ remain **fixed** during training [S1, unofficial]. This prevents the current rendering error from contaminating the supervision target through feedback [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_vtc_consensus(gaussians, images, camera_poses):
    """
    Pre-training step to compute VTC consensus and weights.
    """
    M = len(gaussians)
    N = len(images)
    reliable_mask = np.zeros((M, N), dtype=bool)
    colors = np.zeros((M, N, 3))
    
    for i in range(M):
        for j in range(N):
            # Standard differentiable rendering to get projected color and depth
            projected_color, depth, is_visible = render_gaussian(gaussians[i], images[j], camera_poses[j])
            
            # Criterion 1: Image Bounds
            in_bounds = is_within_image_bounds(projected_color)
            # Criterion 2: Foreground Membership
            is_foreground = check_foreground_membership(projected_color)
            # Criterion 3: Projected Visibility
            visible = is_visible and not is_occluded(projected_color, depth)
            # Criterion 4: Depth Consistency
            depth_consistent = check_depth_consistency(depth)
            
            if in_bounds and is_foreground and visible and depth_consistent:
                reliable_mask[i, j] = True
                colors[i, j] = projected_color
            else:
                reliable_mask[i, j] = False
                colors[i, j] = 0.0

    color_consensus = np.zeros((M, 3))
    mad = np.zeros((M, 3))
    weights = np.zeros((M, N))
    
    for i in range(M):
        reliable_colors = colors[i, reliable_mask[i]]
        K_i = len(reliable_colors)
        
        if K_i > 0:
            color_consensus[i] = np.median(reliable_colors, axis=0)
            deviations = np.abs(reliable_colors - color_consensus[i])
            mad[i] = np.median(deviations, axis=0)
            
            for j in range(N):
                if reliable_mask[i, j]:
                    diff = np.abs(colors[i, j] - color_consensus[i])
                    # Weight determined by MAD and effective sample count [G]
                    weights[i, j] = np.exp(-np.sum(diff) / (1e-6 + np.sum(mad[i]) + 1.0 / K_i))
                else:
                    weights[i, j] = 0.0
        else:
            color_consensus[i] = 0.0
            mad[i] = 0.0
            weights[i, :] = 0.0
            
    return reliable_mask, color_consensus, mad, weights

def vtc_loss(gaussians, images, camera_poses, color_consensus, weights):
    """
    Compute VTC loss during training. Only updates appearance parameters (SH coefficients).
    """
    loss = 0.0
    for i in range(len(gaussians)):
        for j in range(len(images)):
            if weights[i, j] > 0:
                current_color = render_gaussian_color(gaussians[i], images[j], camera_poses[j])
                diff = np.abs(current_color - color_consensus[i])
                loss += weights[i, j] * np.sum(diff)
    return loss

# Standard operations [G]
def render_gaussian(g, img, pose):
    return np.zeros(3), 0.0, True

def is_within_image_bounds(color):
    return True

def check_foreground_membership(color):
    return True

def is_occluded(color, depth):
    return False

def check_depth_consistency(depth):
    return True

def render_gaussian_color(g, img, pose):
    return np.zeros(3)
```

### Parameters
*   **MAD:** Median Absolute Deviation, used to determine consensus weights [S1, unofficial].
*   **Default Configuration:** Enables VTC, parameter routing, local-PCA statistics, and geometric anchoring [S1, unofficial].
*   **Directional Projection, Structure–Appearance Dual-Graph Regularizer, Residual Shrinkage:** Used only in mechanism experiments, not in the default configuration [S1, unofficial].

### Edge Cases and Pitfalls
*   **No Reliable Observations:** If a Gaussian has no reliable observations (e.g., in heavily occluded or texture-less regions), the consensus is undefined [G]. The pseudocode handles this by setting consensus to zero and weights to zero, effectively ignoring the Gaussian in the VTC loss [G].
*   **Precision:** Use double precision for median and MAD calculations to avoid numerical instability in robust statistics [G].
*   **Boundaries:** Ensure image bounds and foreground membership checks are robust to edge cases (e.g., Gaussians partially outside the image) [G].

## Operating characteristics
*   **Time Complexity:** Pre-training filtering is $O(M \cdot N)$, where $M$ is the number of Gaussians and $N$ is the number of views. Consensus construction is $O(M \cdot N)$ [G]. Training loss computation is $O(M \cdot N)$ but only affects appearance parameters [G].
*   **Memory Complexity:** Stores reliable mask, consensus, MAD, and weights: $O(M \cdot N)$ [G].
*   **Strengths:** Improves reconstruction in texture-degraded scenes by rejecting unreliable observations. Decouples geometry and appearance, preventing color errors from corrupting geometric parameters [S1, unofficial].
*   **Limitations:** Requires pre-training filtering, which adds overhead. Fixed consensus may not adapt to dynamic scenes [G].
*   **When to Use:** Texture-degraded scenes, scenes with occlusions, background mixing, or exposure differences [S1, unofficial].
*   **When Not to Use:** High-quality, well-textured scenes where standard 3DGS performs well [G].

## Background
VTC was proposed in "Visibility-Aware Texture Consensus and Local Structural Constraints for 3D Gaussian Reconstruction in Texture-Degraded Scenes" [S1, unofficial]. It addresses the identifiability of geometric supervision under texture degradation [S1, unofficial]. The method is based on the hypothesis that weakly textured regions and foreground-background mixed boundaries reduce the identifiability of photometric gradients with respect to geometric parameters [S1, unofficial].

## Sources
[S1] Visibility-Aware Texture Consensus and Local Structural Constraints for 3D Gaussian Reconstruction in Texture- Degraded Scenes, https://doi.org/10.21203/rs.3.rs-10727268/v1, fetched 2026-10-26 (unofficial)
