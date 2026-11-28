# Consistent Anchor Guided Gaussian Splatting

Also called: CAG-GS

## Overview
Consistent Anchor Guided Gaussian Splatting (CAG-GS) is a large-scale scene rendering method that employs a divide-and-conquer strategy with semantic guidance to ensure global consistency across spatial blocks, thereby avoiding artifacts such as floaters and brightness inconsistencies [S1]. It distinguishes itself from sibling methods like Momentum-GS by directly incorporating consistent semantic information from a pre-trained segmentation model (SAM2) to guide feature learning, rather than relying solely on shared MLP decoders for regularization [S1].

## How it works
**Inputs:** Sparse point cloud $P$ from Structure-from-Motion (SfM); training views with ground truth images; pre-trained semantic segmentation model (SAM2) [S1].
**Outputs:** A set of 3D Gaussians defined by anchors, optimized for novel view synthesis [S1].

### Data Structures
**Anchor ($A_j$):** Each anchor is defined by:
*   **Level-of-Detail ($l_j$):** Integer level in the hierarchy [S1].
*   **Position ($p_j$):** 3D coordinate in space [S1].
*   **Learnable Feature ($f_j$):** Vector in $\mathbb{R}^d$ [S1].
*   **Spatial Offsets ($o_{j,1}, \dots, o_{j,K}$):** $K$ learnable 3D offset vectors in $\mathbb{R}^{3K}$ used to predict Gaussian positions [S1].

**Implicit Gaussians:** Generated per anchor [S1]. Each Gaussian $i$ has:
*   **Position ($\mu_i$):** Derived from anchor position and offsets [S1].
*   **Color ($c_i$):** Predicted by MLP [S1].
*   **Opacity ($\sigma_i$):** Predicted by MLP [S1].
*   **Covariance ($\Sigma_i$):** Parameterized by scaling and rotation (standard 3D-GS definition) [S1].

### Parameters
*   $L$: Number of Level-of-Detail (LoD) levels for voxelization [S1].
*   $\delta$: Base voxel size [S1].
*   $d$: Dimensionality of the anchor feature vector $f_j$ [S1].
*   $K$: Expected number of Gaussians generated per anchor [S1].
*   $\lambda$: Weighting coefficient for SSIM loss (standard 3D-GS parameter) [S1].

### Algorithm Steps

1.  **Scene Discretization (Anchor Generation):**
    The sparse point cloud $P$ is voxelized at $L$ different resolutions to generate anchor points $A$ [S1]. For each point $p \in P$ and level $l \in \{0, \dots, L-1\}$, the anchor position is calculated as [S1]:
    $$ A = \left\{ \left\lfloor \frac{p}{\delta/2^l} \right\rceil \cdot \frac{\delta}{2^l} \mid p \in P, l = 0, \dots, L-1 \right\} $$
    where $\lfloor \cdot \rceil$ denotes element-wise rounding and $\delta$ is the base voxel size [S1].

2.  **Coarse Global Training:**
    A coarse global model is trained using all available training views to provide global priors [S1].

3.  **Block Division:**
    The scene is divided into multiple spatial blocks [S1].

4.  **Semantic Mapping Module Initialization:**
    A semantic mapping module $\gamma: \mathbb{R}^3 \to \mathbb{R}^D$ is introduced [S1]. This module is trained to align its outputs with visual features extracted from the pre-trained SAM2 model [S1].

5.  **Block-wise Fine-Tuning (Iterative per Block):**
    For each block:
    a.  **View Assignment:** Training views are assigned to the block according to blending weights computed during rendering [S1].
    b.  **Anchor Selection:** During rendering, visible anchors are selected based on their LoD levels and distance from the target viewpoint [S1].
    c.  **Feature Fusion:** For each selected anchor $j$ with position $p_j$ and feature $f_j$, a semantic vector is generated and fused [S1]:
        $$ \tilde{f}_j = W \cdot \gamma(p_j) + f_j $$
        where $\gamma(p_j)$ is the semantic vector encoded by the mapping module, and $W$ is a learnable weight matrix [S1].
    d.  **Gaussian Prediction:** The enhanced feature $\tilde{f}_j$ is passed through MLP decoders to predict the attributes of $K$ implicit Gaussians associated with the anchor [S1].
    e.  **Rendering:** The predicted 3D Gaussians are projected to 2D [S1]. They are sorted by depth and composited using alpha blending [S1]:
        $$ C(x') = \sum_{i \in I(x')} c_i \alpha_i \prod_{k=1}^{i-1} (1 - \alpha_k) $$
        where $x'$ is the queried pixel, $I(x')$ is the set of indices of contributing Gaussians sorted by depth, and $\alpha_i = \sigma_i G'_i(x')$ is the transmittance/opacity contribution of Gaussian $i$ at pixel $x'$ [S1].
    f.  **Optimization:** The Gaussians are optimized by minimizing the photometric loss [S1]:
        $$ \mathcal{L}_{3DGS} = (1 - \lambda)\mathcal{L}_1(I, \hat{I}) + \lambda\mathcal{L}_{SSIM}(I, \hat{I}) $$
        where $I$ is the rendered image, $\hat{I}$ is the ground truth, $\mathcal{L}_1$ is the L1 loss, and $\mathcal{L}_{SSIM}$ is the SSIM loss [S1].

6.  **Merging:**
    The optimized blocks are cropped along their division boundaries and seamlessly merged to obtain the final reconstruction [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class MLP:
    """Standard feedforward network for Gaussian attribute prediction [G]."""
    def __init__(self, in_dim, out_dim):
        self.W = [[0.0]*out_dim for _ in range(in_dim)]
    def __call__(self, x):
        return [sum(self.W[i][j]*x[i] for i in range(len(x))) for j in range(len(self.W[0]))]

def round_element_wise(vec, scale):
    """Element-wise rounding to grid centers [G]."""
    return [round(v / scale) * scale for v in vec]

def generate_anchors(point_cloud, delta, L, d, K):
    """Discretize point cloud into anchors at L levels of detail [S1]."""
    anchors = []
    for p in point_cloud:
        for l in range(L):
            grid_size = delta / (2 ** l)
            pos = round_element_wise(p, grid_size)
            anchors.append({
                'level': l, 'position': pos,
                'feature': [0.0]*d, 'offsets': [[0.0]*3 for _ in range(K)]
            })
    return anchors

def semantic_mapping(position, gamma_net):
    """Encode 3D position into semantic vector via SAM2-aligned module [S1]."""
    return gamma_net(position)

def fuse_features(anchor, semantic_vec, W):
    """Fuse semantic information with anchor's learnable feature [S1]."""
    return [sum(W[i][j]*semantic_vec[i] for i in range(len(W))) + anchor['feature'][j] for j in range(len(anchor['feature']))]

def predict_gaussians(enhanced_feat, mlp_dec):
    """Predict attributes for K Gaussians from enhanced feature [S1]."""
    return mlp_dec(enhanced_feat)

def render_block(anchors, viewpoint, gamma_net, W, mlp_dec, K):
    """Render a single block using selected anchors [S1]."""
    visible = [a for a in anchors if math.dist(a['position'], viewpoint.cam_pos) < viewpoint.max_dist]
    gaussians = []
    for a in visible:
        sem_vec = semantic_mapping(a['position'], gamma_net)
        fused = fuse_features(a, sem_vec, W)
        attrs = predict_gaussians(fused, mlp_dec)
        for k in range(K):
            gaussians.append({
                'pos': [a['position'][i]+a['offsets'][k][i] for i in range(3)],
                'color': attrs[k*4:k*4+3], 'opacity': attrs[k*4+3]
            })
    gaussians.sort(key=lambda g: math.dist(g['pos'], viewpoint.cam_pos))
    img = [[0.0]*3 for _ in range(viewheight)]
    for g in gaussians:
        alpha = g['opacity'] * math.exp(-0.5 * math.dist(g['pos'], viewpoint.pixel_proj))
        img = [[img[r][c]*(1-alpha) + g['color'][c]*alpha for c in range(3)] for r in range(viewheight)]
    return img

def train_cag_gs(point_cloud, views, gamma_net, W, mlp_dec, delta, L, K, d):
    """Main training loop with coarse-to-fine strategy [S1]."""
    anchors = generate_anchors(point_cloud, delta, L, d, K)
    # Coarse global training: optimize all anchors using all views [S1]
    for epoch in range(num_epochs_coarse):
        for v in views:
            img = render_block(anchors, v, gamma_net, W, mlp_dec, K)
            loss = photometric_loss(img, v.gt_image)
            update_parameters(loss)
    # Block division: partition anchors into spatial blocks [S1]
    blocks = divide_scene(anchors)
    for block in blocks:
        # View assignment: assign views based on blending weights [S1]
        block_views = assign_views(block, views)
        # Block-wise fine-tuning
        for epoch in range(num_epochs_fine):
            for v in block_views:
                img = render_block(block, v, gamma_net, W, mlp_dec, K)
                loss = photometric_loss(img, v.gt_image)
                update_parameters(loss)
    # Merging: crop blocks at boundaries and merge [S1]
    final_scene = merge_blocks(blocks)
    return final_scene
```

### Edge Cases and Pitfalls
*   **Precision:** The rounding operation $\lfloor \cdot \rceil$ in anchor generation must be handled carefully to ensure consistent anchor placement across different resolutions [S1].
*   **Boundaries:** When merging blocks, cropping along division boundaries is required to avoid double-counting or gaps, though the specific blending function for the merge is not detailed beyond "seamlessly merged" [S1].
*   **Global Priors:** The coarse global model provides global priors during the initial training phase [S1].

## Operating characteristics
*   **Time Complexity:** The method involves a coarse global training phase followed by block-wise fine-tuning [S1]. The complexity is dominated by the rendering and optimization steps per block [S1]. The use of LoD levels allows for efficient culling of distant anchors [S1].
*   **Memory Complexity:** Memory usage is reduced compared to explicit 3D-GS by using implicit Gaussian prediction from anchors [S1]. The number of anchors is determined by the voxelization of the SfM point cloud [S1].
*   **Strengths:** Handles large-scale scenes efficiently; avoids floaters and brightness inconsistencies through semantic guidance; preserves global consistency via coarse-to-fine training [S1].
*   **Limitations:** Requires a pre-trained semantic segmentation model (SAM2); performance depends on the quality of the initial SfM point cloud [S1].
*   **When to use:** Large-scale scene rendering where global consistency is critical and explicit Gaussian storage is prohibitive [S1].
*   **When not to use:** Small-scale scenes where explicit 3D-GS is sufficient; scenes with poor SfM reconstruction [S1].

## Background
CAG-GS was proposed to address the computational and storage burdens of 3D Gaussian Splatting (3D-GS) when applied to large-scale scenes [S1]. It builds on the implicit scene representation approach where anchors predict Gaussian attributes [S1]. The method was introduced to overcome the limitations of divide-and-conquer strategies that often lead to limited performance due to inconsistent lighting and floaters [S1]. The hierarchical representation follows OctreeGS [S1]. The method integrates semantic guidance from SAM2 to improve feature learning [S1].

## Sources
[S1] CAG-GS: Consistent Anchor Guided Gaussian Splatting for Large-scale Scene Rendering, https://doi.org/10.1609/aaai.v40i14.38120, fetched 2026-10-29
