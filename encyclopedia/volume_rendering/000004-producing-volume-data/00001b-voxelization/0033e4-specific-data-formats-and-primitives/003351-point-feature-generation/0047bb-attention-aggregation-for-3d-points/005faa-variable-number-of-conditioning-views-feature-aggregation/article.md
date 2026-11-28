# Variable Number of Conditioning Views Feature Aggregation

Also called: Camera-aware Feature Aggregation, Multi-View Feature Aggregation, Permutation Invariant Feature Aggregation with Viewpoint Info

## Overview
This method aggregates pixel-aligned features from an arbitrary number of conditioning views into a single feature vector for a query 3D point, solving the problem of variable input dimensions at inference time [S2, unofficial]. It differs from fixed-view concatenation by being permutation invariant to the number of conditioning views, and it differs from simple mean pooling by using max pooling or camera-aware transformations to mitigate artifacts caused by inherent depth ambiguity [S1, unofficial][S2, unofficial].

## How it works
### Inputs and Outputs
- **Inputs:** A batch of $B$ query points, $n$ conditioning views per point. For each view $i \in \{0, \dots, n-1\}$: a feature map tensor $F^{(i)} \in \mathbb{R}^{C \times H \times W}$, projected 2D coordinates $P^{(i)} \in \mathbb{R}^{N \times 2}$, and optionally camera parameters $c_i \in \mathbb{R}^7$ (4D rotation quaternion + 3D camera position) [S1, unofficial][S2, unofficial].
- **Outputs:** Aggregated feature tensor $F_{agg} \in \mathbb{R}^{B \times C \times N}$ [S1, unofficial].

### Data Structures
- The aggregation function $G: \mathbb{R}^{n \times C} \to \mathbb{R}^C$ must satisfy $G([f^{(1)}, \dots, f^{(n)}]) = G([f^{\psi(1)}, \dots, f^{\psi(n)}])$ for any permutation $\psi$ [S2, unofficial].
- The camera-aware summarization network $N_{cf}: \mathbb{R}^{C+7} \to \mathbb{R}^C$ [S2, unofficial].

### Steps
1. **Projection & Sampling:** For each view $i$, sample features at the projected 2D coordinates using bilinear interpolation or grid sampling, yielding $f^{(i)} \in \mathbb{R}^{C \times N}$ [S1, unofficial][S3].
2. **Stacking:** Stack the sampled features across the view dimension to form $F_{stack} \in \mathbb{R}^{C \times N \times n}$ [S1, unofficial].
3. **Aggregation:** Apply a permutation-invariant reduction across the view dimension. The reference implementation uses max pooling [S1, unofficial]. Alternatively, for camera-aware aggregation, transform each view's feature with $N_{cf}$ before averaging [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aggregate_view_features(view_feats, points_proj, fusion='max'):
    # view_feats: [B, M, C, H, W], points_proj: [B, M, N, 2]
    sampled = []
    for i in range(view_feats.shape[1]):
        feat = view_feats[:, i, :, :, :]
        coords = points_proj[:, i, :, :]
        sampled.append(index_2d(feat, coords)) # Standard grid/bilinear sampling -> [B, C, N]
    stacked = torch.stack(sampled, dim=-1) # [B, C, N, M]
    if fusion == 'max':
        pooled = F.max_pool2d(stacked, kernel_size=(1, stacked.shape[-1]))
        return pooled.squeeze(-1) # [B, C, N]
    else:
        raise NotImplementedError("Only 'max' fusion is implemented.")
```

### Formulas
- **Permutation Invariance:** $G([f^{(1)}, \dots, f^{(n)}]) = G([f^{\psi(1)}, \dots, f^{\psi(n)}])$ [S2, unofficial].
- **Camera-Aware Update:** $f_X^{0(i)} = N_{cf}(f_X^{(i)}, c_i)$ [S2, unofficial].
- **Camera-Aware Aggregation:** $f_X = \frac{1}{n} \sum_{i=1}^n f_X^{0(i)}$ [S2, unofficial].
- **Weighted Pooling Variant:** $f_X = \sum_{i=1}^n w_i f_i'$, where $w_i = \text{Softmax}(\text{MLP}([\mu, \nu]))$ and $\mu, \nu$ are the mean and variance of the concatenated feature-pixel vector [S3].

### Parameters
- Camera parameters $c_i$: 4D rotation quaternion + 3D camera position (7D vector) [S2, unofficial].
- Typical values: Not given in the sources.

### Edge Cases and Pitfalls
- Simple mean aggregation produces artifacts due to inherent depth ambiguity when points are projected onto the feature image before sampling [S2, unofficial].
- The reference code only implements 'max' fusion; other modes raise `NotImplementedError` [S1, unofficial].
- Projected coordinates falling outside the valid sampling bounds require clamping or padding, which is not specified in the sources [G].

## Subtopics

### Camera-Aware Feature Summarization
This variant explicitly injects viewpoint information into the feature aggregation to handle depth ambiguity [S2, unofficial]. For each view $i$, it computes a camera-summarized feature $f_X^{0(i)} = N_{cf}(f_X^{(i)}, c_i)$, where $N_{cf}$ is a learned network and $c_i$ is the 7D camera parameter vector [S2, unofficial]. The final feature is the average across all views: $f_X = \frac{1}{n} \sum_{i=1}^n f_X^{0(i)}$ [S2, unofficial]. This accounts for likely occlusions before averaging [S2, unofficial].

### Multi-View Feature Aggregation via Weighted Pooling
This variant computes dynamic weights for each view based on geometric consistency [S3]. For each view $i$, it transforms the 3D point $x$ and reference direction $g$ to view coordinates ($x_i = R_i x + t_i$, $g_i = R_i g$), projects them using intrinsic $K_i$, and samples pixel $p_i$ and feature $f_i$ via bilinear interpolation [S3]. It concatenates $f_i$ and $p_i$, computes mean $\mu$ and variance $\nu$, feeds them into a lightweight MLP to extract perceptual features $f_i'$, and uses a Softmax over $\mu, \nu$ to generate weights $w_i$ [S3]. Finally, it performs weighted pooling: $f_X = \sum w_i f_i'$ [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of conditioning views $n$, requiring one sampling pass and one aggregation pass per view [S2, unofficial].
- **Memory Complexity:** Linear with respect to $n$ to store intermediate sampled features before reduction [S2, unofficial].
- **Strengths:** Handles arbitrary numbers of input views at inference without architectural changes; reduces artifacts caused by depth ambiguity by leveraging permutation-invariant reduction or explicit camera geometry [S1, unofficial][S2, unofficial].
- **Limitations:** Requires accurate camera intrinsics/extrinsics and, for the camera-aware variant, training a separate network $N_{cf}$ [S2, unofficial]. Performance degrades if projected coordinates are poorly aligned or occluded [S2, unofficial].

## Background
Originated in the PVA (Pixel-aligned Volumetric Avatars) paper [S2, unofficial]. The work addresses the limitation of previous methods that required a fixed number of conditioning views or suffered from artifacts when using simple mean pooling in multi-view settings with depth ambiguity [S2, unofficial]. It builds on permutation-invariant aggregation principles used in prior point-cloud and multi-view networks [S2, unofficial].

## Sources
[S1] DIF-Gaussian, https://github.com/xmed-lab/DIF-Gaussian, fetched 2026-10-27 (unofficial)
[S2] PVA: Pixel-aligned Volumetric Avatars, https://doi.org/10.48550/arxiv.2101.02697, fetched 2026-10-30 (unofficial)
[S3] Multi-View Stereo Network Based on Attention Mechanism and Neural Volume Rendering, https://doi.org/10.3390/electronics12224603, fetched 2026-10-26
