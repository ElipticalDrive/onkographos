# VolSplat Architecture and Training Pipeline

Also called: VolSplat

## Overview
VolSplat is a feed-forward 3D Gaussian Splatting framework that regresses scene parameters from sparse multi-view images in a single pass, bypassing per-scene optimization [S1]. It distinguishes itself by lifting 2D features into a sparse 3D voxel grid via plane-sweep depth prediction and predicting exactly one Gaussian primitive per occupied voxel, ensuring scene-adaptive allocation with stable sparse decoding [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** $N_{\text{in}} = 6$ input views at $256 \times 256$ resolution [S1].
- **Outputs:** A set of 3D Gaussian primitives $\mathcal{G}$, each defined by a 38-dimensional parameter vector, for novel view synthesis [S1].

**Data Structures:**
1. **Multi-scale Feature Maps:** Extracted per view via a weight-sharing ResNet backbone [S1].
2. **Cost Volume:** Constructed via plane-sweep with $D = 128$ inverse-depth candidates per reference view [S1].
3. **Sparse Voxel Grid:** World-coordinate representation where only occupied cells are materialized, processed via MinkowskiEngine [S1].
4. **Gaussian Primitive Parameter Vector $\mathbf{p} \in \mathbb{R}^{38}$:** Comprises opacity $\alpha$, center offset $\Delta \mathbf{x}$, anisotropic scale $\mathbf{s}$, quaternion rotation $\mathbf{q}$, and degree-2 spherical-harmonic color coefficients $\mathbf{c}$ [S1].

**Numbered Steps:**
1. Extract multi-scale feature maps from each input view using a weight-sharing ResNet backbone [S1].
2. Refine features via a cross-view interaction module using local window attention to enhance multi-view consistency [S1].
3. Construct a cost volume for each reference view using a plane-sweep strategy with $D = 128$ inverse-depth candidates [S1].
4. Perform local neighbor matching on the cost volume to predict robust depth maps [S1].
5. Lift the 2D features into 3D space using the predicted depth maps, building the representation in world coordinates [S1].
6. Process the lifted features as a sparse voxel set where only occupied cells are materialized, applying sparse 3D refinement via MinkowskiEngine [S1].
7. Predict one Gaussian primitive per occupied voxel, outputting the 38-dimensional parameter vector $\mathbf{p}$ [S1].

**Formulas:**
The parameter vector $\mathbf{p}$ is defined as:
$$\mathbf{p} = [\alpha, \Delta x, \Delta y, \Delta z, s_x, s_y, s_z, q_w, q_x, q_y, q_z, c_0, \dots, c_{37}]$$
where $\alpha \in [0,1]$ is opacity, $(\Delta x, \Delta y, \Delta z)$ is the center offset, $(s_x, s_y, s_z)$ is the anisotropic scale, $(q_w, q_x, q_y, q_z)$ is the quaternion rotation, and $c_0 \dots c_{37}$ are the degree-2 spherical-harmonic color coefficients [S1].

**Parameters:**
- Inverse-depth candidates: 128 per reference view [S1].
- Gaussian parameter vector dimension: 38 [S1].
- Input views per training sample: 6 [S1].
- Target views per training sample: 8 [S1].
- Input resolution: $256 \times 256$ pixels [S1].
- Optimizer: AdamW with decoupled weight decay [S1].
- Learning rate: Two separate groups for the pretrained monocular branch and the remaining trainable parameters [S1].
- Stability mechanism: Gradient clipping is enabled during training [S1].

**Edge Cases and Pitfalls:**
- Sparse grid initialization: Only occupied cells are materialized to avoid computational redundancy of a dense global grid [S1].
- Depth prediction errors: Robust depth prediction via local neighbor matching is critical for accurate feature lifting [S1].
- Anchor gap selection: The progressive expansion of the anchor-gap range during early training iterations is required for stable convergence [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class VolSplatPipeline:
    def __init__(self, num_depth_candidates=128, input_resolution=(256, 256)):
        self.num_depth_candidates = num_depth_candidates
        self.input_resolution = input_resolution
        # Weight-sharing ResNet backbone for multi-scale feature extraction [S1]
        self.backbone = nn.Sequential(
            nn.Conv2d(3, 64, 7, stride=2, padding=3, bias=False),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True)
        )
        # Cross-view interaction via local window attention [S1]
        self.cross_view_attn = nn.MultiheadAttention(embed_dim=256, num_heads=8, batch_first=True)
        # Depth prediction head [S1]
        self.depth_head = nn.Sequential(
            nn.Conv2d(256, 128, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(128, 1, 3, padding=1)
        )
        # Sparse 3D refinement layers [S1]
        self.sparse_refine = nn.Sequential(
            nn.Conv3d(256, 256, 3, padding=1),
            nn.BatchNorm3d(256),
            nn.ReLU(inplace=True)
        )
        # Gaussian parameter predictor [S1]
        self.gaussian_head = nn.Conv3d(256, 38, 1)

    def forward(self, input_views):
        # input_views: Tensor of shape (N_in, 3, 256, 256)
        # Step 1: Feature Extraction
        features = self.backbone(input_views) # Shape: (N_in, C, H, W)

        # Step 2: Cross-View Interaction
        B, C, H, W = features.shape
        feat_flat = features.permute(0, 2, 3, 1).reshape(B, H*W, C)
        refined_feat, _ = self.cross_view_attn(feat_flat, feat_flat, feat_flat)
        refined_features = refined_feat.reshape(B, H, W, C).permute(0, 3, 1, 2)

        # Step 3 & 4: Cost Volume Construction and Depth Prediction
        # Construct cost volume with 128 inverse-depth candidates [S1]
        # Perform local neighbor matching to predict depth maps [S1]
        depth_maps = []
        for i in range(B):
            # Sources do not specify the exact cost volume aggregation and matching logic [S1]
            cost_vol = self._build_cost_volume(refined_features[i:i+1])
            depth_map = self._local_neighbor_matching(cost_vol)
            depth_maps.append(depth_map)
        depth_maps = torch.stack(depth_maps, dim=0) # Shape: (N_in, H, W)

        # Step 5: Feature Lifting
        # Lift 2D features to 3D space using predicted depth maps in world coordinates [S1]
        # Sources do not specify the exact lifting/ray-casting logic [S1]
        lifted_features = self._lift_features(refined_features, depth_maps)

        # Step 6: Sparse 3D Refinement
        # Create sparse voxel set, materializing only occupied cells [S1]
        # Sources do not specify the exact sparse tensor creation logic [S1]
        sparse_tensor = self._create_sparse_tensor(lifted_features)
        refined_sparse = self.sparse_refine(sparse_tensor)

        # Step 7: Gaussian Parameter Prediction
        # Predict one Gaussian primitive per occupied voxel [S1]
        gaussian_params = self.gaussian_head(refined_sparse)
        return gaussian_params

    def _build_cost_volume(self, ref_feat):
        # Plane-sweep cost volume construction with 128 candidates [S1]
        # Sources do not specify the exact implementation [S1]
        return torch.zeros(1, 1, 256, 256, 128)

    def _local_neighbor_matching(self, cost_volume):
        # Local neighbor matching on cost volume [S1]
        # Sources do not specify the exact implementation [S1]
        return torch.zeros(1, 256, 256)

    def _lift_features(self, features, depth_maps):
        # Lift 2D features to 3D using depth maps [S1]
        # Sources do not specify the exact implementation [S1]
        return torch.zeros(1, 256, 16, 16, 16)

    def _create_sparse_tensor(self, lifted_features):
        # Create sparse voxel set from lifted features [S1]
        # Sources do not specify the exact implementation [S1]
        return torch.zeros(1, 256, 16, 16, 16)
```

**Training Data Sampling:**
- Each sample uses $N_{\text{in}} = 6$ input views and $N_{\text{target}} = 8$ target views [S1].
- Two input views are selected as boundary anchors with a randomly sampled frame gap within a predefined range [S1].
- The remaining input views are sampled between the anchors [S1].
- The anchor-gap range is progressively expanded during early training iterations [S1].
- Target novel views are chosen from disjoint camera positions not included in the inputs [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Evaluated on a single NVIDIA H20 GPU with 6 input views [S1]. VolSplat achieves a superior balance between reconstruction quality and resource consumption compared to state-of-the-art baselines on RealEstate10K [S1].
- **Scaling:** Not given in the sources.
- **Strengths:** Scene-adaptive primitive allocation, stable sparse 3D decoding, efficient sparse 3D refinement using MinkowskiEngine [S1].
- **Limitations:** Not given in the sources.
- **When to use:** When a feed-forward approach for 3DGS is required, especially for scenes with sparse input views [Parent](9213-mvsplat.md).
- **When not to use:** When per-scene optimization is acceptable and higher quality is needed [Parent](9213-mvsplat.md).

## Background
VolSplat was introduced by Choy, JunYoung Gwak, and Silvio Savarese in the paper "VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction" [S1]. The method builds on previous work in feed-forward 3DGS, such as MVSplat [Parent](9213-mvsplat.md), and incorporates techniques from 4D spatio-temporal convnets (MinkowskiEngine) for efficient sparse 3D processing [S1]. Training is performed on RealEstate10K and fine-tuned on ScanNet, with zero-shot evaluation on ACID [S1].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
