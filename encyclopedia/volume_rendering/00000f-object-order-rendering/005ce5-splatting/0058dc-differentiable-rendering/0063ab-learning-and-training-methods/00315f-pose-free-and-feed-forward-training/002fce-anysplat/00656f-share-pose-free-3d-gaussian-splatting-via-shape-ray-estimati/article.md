# SHARE (Pose-free 3D Gaussian Splatting via Shape-Ray Estimation)

Also called: Pose-free 3D Gaussian Splatting via Shape-Ray Estimation

## Overview
SHARE is a pose-free, feed-forward Gaussian splatting framework that jointly estimates relative camera poses and reconstructs accurate 3D geometry by operating in latent space rather than explicit 3D space, mitigating pose-induced geometry misalignment [S1]. Unlike methods that transform 3D geometries from each viewpoint into a reference camera coordinate using predicted poses, SHARE constructs a unified cost volume in latent space, bypassing erroneous 3D space transformation [S1]. It sets itself apart from siblings like [FreeSplat](15163-freesplat.md) and [ZipSplat](17331-zipsplat.md) by using Plücker rays to guide multi-view feature fusion directly, avoiding the sequential estimation of pose and shape that leads to significant geometric errors [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $M$ unposed images $I = \{I_i\}_{i=1}^M$.
*   **Output:** A set of $N$ 3D Gaussian primitives $\{G_n\}_{n=1}^N$ and relative camera poses $\{R_i, T_i\}_{i=1}^M$. Each Gaussian $G_n$ is defined by position $\mu_n$, opacity $\alpha_n$, covariance matrix $\Sigma_n$, and color $c_n$ [S1].

**Data Structures:**
*   **Multi-view Features:** $F_i \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times C}$, extracted per view [S1].
*   **Plücker Rays:** $P_{li} \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times 6}$, defined per spatial patch $l$ for view $i$. Each ray consists of direction $d$ and momentum $m$, denoted $P = \langle d, m \rangle$ [S1].
*   **Cost Volumes:** $C_i \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times D}$ for each view $i$, where $D$ is the number of hypothetical depth candidates [S1].
*   **Spatial Weights:** $W_i \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4}}$ used to account for view-dependent effects [S1].
*   **Fused Feature:** $V_g$, the canonical volume representation [S1].
*   **Parameters:** $\Phi_\theta$ denotes the network mapping [S1].

**Numbered Steps:**

1.  **Joint Feature Extraction:** Estimate multi-view features $F_i \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times C}$ using a matching transformer backbone [S1].
2.  **Ray Estimation:** Estimate patch-wise Plücker rays $P_{li} \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times 6}$ through an additional two-layer lightweight transformer. Each ray is defined by direction $d$ and momentum $m$ [S1].
3.  **Cost Volume Construction:** For each viewpoint $i$, construct cost volumes $C_i$ by extending 2D features into 3D along hypothetical planes and projecting onto other views. Compute channel-wise correlations between the reference feature $F_i$ and warped features $F_{d_j \to i}$ for $D$ depth candidates. The cost volume $C_i$ is computed as:
    $$ C_i = \sum_{j \neq i} \frac{F_i \cdot F_{j \to i}}{\sqrt{C}} \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times D} $$
    where $D$ is the number of hypothetical depth candidates [S1].
4.  **Pose-aware Cost Aggregation:** Refine cost volumes through cost aggregation conditioned on predicted Plücker rays. Use patch-wise cross-attention to embed pose awareness into the cost volumes. Since rays are locally defined in spatial patches, they are seamlessly integrated with image features, introducing geometric bias into the cost volumes [S1].
5.  **Canonical Volume Formation:** Select one viewpoint as the canonical view. Align remaining views to the canonical space using the pose-aware cost volumes and spatial weights $W_i$ to account for occlusion and lighting, building a fused feature $V_g$ [S1].
6.  **Anchor Prediction:** Use fused features $V_g$ to predict pixel-aligned coarse geometry (anchors) [S1].
7.  **Anchor-aligned Gaussian Prediction:** Use fused features $V_f$ (derived from $V_g$) to predict offset vectors from the coarse anchors to spread to adjacent regions. Predict $k$ Gaussians per region for fine-grained reconstruction [S1].

**Formulas:**
The objective mapping function is:
$$ \Phi_\theta : \{I_i\}_{i=1}^M \rightarrow \{G_n\}_{n=1}^N, \{R_i, T_i\} = \Psi(\{P_{li}\}_{l=1}^{P_h \times P_w}) $$
where $G_n = \{\mu_n, \alpha_n, \Sigma_n, c_n\}$, $P = \langle d, m \rangle$, and $\Psi$ converts Plücker rays to camera parameters $(R, T)$ [S1].

**Parameters:**
*   $P_h, P_w$: Spatial patch resolution used for defining Plücker rays and cost volumes [S1].
*   $D$: Number of hypothetical depth candidates used in cost volume construction [S1].

**Pitfalls:**
*   Sequential estimation of camera pose and shape leads to significant geometry misalignment if pose estimates are inaccurate. SHARE avoids this by joint estimation in latent space [S1].
*   Inaccurate pose estimates in traditional methods cause errors when transforming 3D geometries into a reference coordinate system. SHARE bypasses this by fusing features in latent space [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def share_forward(images):
    """
    Input: images - List of M unposed images I_i.
    Output: Gaussians {G_n}, Camera Poses {R_i, T_i}
    """
    # Step 1: Joint Feature Extraction
    # F_i in R^(H/4 x W/4 x C)
    features = []
    for img in images:
        # Using a matching transformer backbone
        F_i = matching_transformer_backbone(img)
        features.append(F_i)
    
    # Step 2: Ray Estimation
    # P_li in R^(H/4 x W/4 x 6)
    # Plucker rays P = <d, m>
    rays = []
    for F_i in features:
        # Using a two-layer lightweight transformer
        P_i = lightweight_transformer(F_i)
        rays.append(P_i)
        
    # Step 3: Cost Volume Construction
    # C_i in R^(H/4 x W/4 x D)
    cost_volumes = []
    for i in range(len(images)):
        C_i = torch.zeros((H/4, W/4, D))
        for j in range(len(images)):
            if i != j:
                # Warp features from view j to view i based on hypothetical depth
                # F_{j->i} in R^(H/4 x W/4 x C)
                F_j_to_i = warp_features(features[j], hypothetical_depths, rays[j])
                # Channel-wise correlation
                # Ci += (Fi . Fj->i) / sqrt(C)
                correlation = torch.sum(features[i] * F_j_to_i, dim=-1) / math.sqrt(C)
                C_i += correlation
        cost_volumes.append(C_i)
        
    # Step 4: Pose-aware Cost Aggregation
    # Refine cost volumes using patch-wise cross-attention with rays
    refined_cost_volumes = []
    for i in range(len(images)):
        # Cross-attention between cost volume and Plucker rays
        C_prime_i = patchwise_cross_attention(cost_volumes[i], rays[i])
        refined_cost_volumes.append(C_prime_i)
        
    # Step 5: Canonical Volume Formation
    # Select view 0 as canonical
    canonical_view_index = 0
    # Align other views to canonical space using refined cost volumes and weights
    # W_i in R^(H/4 x W/4) accounts for view-dependent effects
    # V_g is the fused feature
    V_g = align_to_canonical(refined_cost_volumes, weights, canonical_view_index)
    
    # Step 6: Anchor Prediction
    # Predict coarse geometry anchors from V_g
    anchors = predict_anchors(V_g)
    
    # Step 7: Anchor-aligned Gaussian Prediction
    # Predict offsets from anchors to spread to adjacent regions
    # Predict k Gaussians per region
    offsets = predict_offsets(V_g, anchors)
    gaussians = spread_gaussians(anchors, offsets)
    
    # Convert Plucker rays to camera parameters
    # Psi converts P = <d, m> to R, T
    poses = []
    for P_i in rays:
        R_i, T_i = convert_plucker_to_pose(P_i)
        poses.append((R_i, T_i))
        
    return gaussians, poses
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On real-world datasets including DTU, BlendedMVS, RealEstate10K, and ACID, SHARE achieves robust performance compared to existing generalizable pose-free approaches [S1]. It outperforms these baselines even when compared to pose-dependent Gaussian splatting methods [S1]. In specific comparisons with MVSplat, SHARE shows higher PSNR (e.g., 18.46dB vs 11.88dB in noisy pose scenarios) [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Robust to pose noise; avoids geometry misalignment caused by inaccurate pose estimates; efficient and scalable pose-free novel view synthesis via a single feed-forward pipeline [S1].
*   **Limitations:** Not given in the sources.
*   **When to use it:** When precise camera poses are unavailable or noisy, and robust 3D reconstruction is required from unposed image collections [S1].
*   **When not:** When precise camera poses are already available and explicit 3D transformation is preferred [S1].

## Background
SHARE was introduced to address the challenge of obtaining accurate camera poses in real-world scenarios, which often leads to noisy pose estimates and geometric misalignments [S1]. It was developed as a pose-free, feed-forward Gaussian splatting framework that overcomes these ambiguities by joint shape and camera rays estimation [S1]. The method builds on the principles of multi-view stereo (MVS) techniques, specifically cost volume construction and aggregation, but adapts them to latent space for Gaussian splatting [S1]. It is used for novel view synthesis and 3D scene reconstruction from sparse unposed images [S1].

## Sources
[S1] Pose-free 3D Gaussian splatting via shape-ray estimation, https://arxiv.org/abs/2505.22978, fetched 2026-10-26
