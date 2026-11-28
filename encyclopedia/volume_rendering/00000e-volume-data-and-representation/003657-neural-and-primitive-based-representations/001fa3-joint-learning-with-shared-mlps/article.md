# Joint Learning with Shared MLPs

Also called: Context Model Architecture, Shared decoder training

## Overview
Joint Learning with Shared MLPs is a strategy for simultaneously training multiple volumetric visualization (VolVis) scenes using a single, lightweight neural network to decode scene attributes from compact anchor points [S1, unofficial]. It solves the problem of redundant computation and storage when handling large collections of similar volumetric datasets by exploiting inter-scene correlations [S1, unofficial]. Unlike separate per-scene optimization, this method groups geometrically similar scenes via hierarchical clustering and shares a context model (typically a hash-grid-based MLP) across all scenes in a group, significantly reducing training time, model size, and anchor count while maintaining reconstruction quality [S1, unofficial].

## How it works
**Inputs:** A set of $N$ VolVis scenes, each represented by a point cloud for geometric initialization and associated training images [S1, unofficial].
**Outputs:** A shared context model (binary hash grids + shallow MLP) and a set of anchor attributes (e.g., Gaussian parameters) for each scene, optimized jointly [S1, unofficial].

**Data Structures:**
1. **Anchor Point Cloud:** A subsampled set of points $\mathbf{x}_a$ representing the scene geometry, limited to at most 10,000 points for clustering and initialization [S1, unofficial].
2. **Context Model:** Based on BiRF, consisting of multiple binary hash grids with parameters restricted to $+1$ or $-1$, and a shallow MLP that decodes aggregated hash features [S1, unofficial].
3. **Scene Groups:** Disjoint partitions $\mathcal{G}_1, \dots, \mathcal{G}_k$ formed by computing pairwise Chamfer distances between subsampled point clouds and applying hierarchical clustering [S1, unofficial].

**Steps:**
1. **Scene Grouping:** Compute pairwise Chamfer distances between the subsampled point clouds of all $N$ scenes [S1, unofficial]. Apply hierarchical clustering with a threshold $\tau$ (Chamfer distance) to form groups [S1, unofficial].
2. **Initialization:** Initialize anchor points for each scene using the subsampled point clouds ($\le 10,000$ points) [S1, unofficial].
3. **Joint Optimization:** For each group $\mathcal{G}_g$:
   - Initialize the shared context model parameters (hash grid values and MLP weights) [S1, unofficial].
   - Initialize scene-specific anchor attributes for all scenes in the group [S1, unofficial].
   - Iterate over training images from all scenes in the group [S1, unofficial].
   - For each image, sample rays and evaluate the scene function using the shared context model [S1, unofficial]. The context model takes an anchor position $\mathbf{x}_a$ as input and outputs values modeling the probability distribution for the anchor attributes [S1, unofficial].
   - Render the image and compute the loss against the ground truth [S1, unofficial].
   - Backpropagate gradients to update both the shared context model and the scene-specific anchor attributes [S1, unofficial].
4. **Entropy Coding:** Continuous anchor attributes are quantized to discrete values (via noise injection for backpropagation during training) to enable neural entropy coding, further compressing the model [S1, unofficial].

**Parameters:**
- **Clustering Threshold ($\tau$):** Controls the maximum Chamfer distance for scenes to be grouped together [S1, unofficial]. Tested values: $0, 0.005, 0.01, 0.02, 0.04$ [S1, unofficial]. Larger $\tau$ reduces the number of groups but risks OOM errors if groups become too large [S1, unofficial].
- **Subsampling Limit:** Maximum 10,000 points for clustering and initialization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joint_learning_with_shared_mlps(scenes, clustering_threshold=0.02):
    # Step 1: Subsample point clouds for clustering efficiency [S1, unofficial]
    subsampled_clouds = []
    for scene in scenes:
        pts = scene.point_cloud
        if len(pts) > 10000:
            subsampled_clouds.append(pts[:10000])
        else:
            subsampled_clouds.append(pts)

    # Compute pairwise Chamfer distances [S1, unofficial]
    n = len(subsampled_clouds)
    dist_matrix = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(i + 1, n):
            d_ab = sum(min(dist(a, b) for b in subsampled_clouds[j]) for a in subsampled_clouds[i]) / len(subsampled_clouds[i])
            d_ba = sum(min(dist(b, a) for a in subsampled_clouds[i]) for b in subsampled_clouds[j]) / len(subsampled_clouds[j])
            dist_matrix[i][j] = (d_ab + d_ba) / 2.0
            dist_matrix[j][i] = dist_matrix[i][j]

    # Hierarchical clustering algorithm not detailed in sources [G]
    groups = [[i] for i in range(n)]

    # Step 2 & 3: Joint optimization for each group [S1, unofficial]
    # Context model: Binary hash grids (+1/-1) + shallow MLP [S1, unofficial]
    context_model = None # Not detailed in sources [G]
    all_anchors = []

    for group in groups:
        group_scenes = [scenes[i] for i in group]
        scene_anchors = []
        for scene in group_scenes:
            scene_anchors.append(None) # Not detailed in sources [G]

        # Joint training loop [S1, unofficial]
        for epoch in range(100):
            for scene_idx, scene in enumerate(group_scenes):
                for image in scene.training_images:
                    # Forward pass: context model takes anchor position x_a as input
                    # and outputs values modeling the probability distribution for anchor attributes [S1, unofficial]
                    rendered_image = []
                    for ray in image.rays:
                        points = [] # Not detailed in sources [G]
                        decoded_attrs = [context_model.decode(p) for p in points]
                        pixel = None # Not detailed in sources [G]
                        rendered_image.append(pixel)

                    loss = None # Not detailed in sources [G]
                    update_shared_model_and_anchors(context_model, scene_anchors[scene_idx], loss) # Not detailed in sources [G]

    return context_model, all_anchors

def dist(a, b):
    return sum((x - y)**2 for x, y in zip(a, b))**0.5
```

**Edge Cases and Pitfalls:**
*   **OOM Errors:** Setting the clustering threshold too high (e.g., 0.04) can group >20 scenes together, causing Out-Of-Memory (OOM) errors during training because anchor/optimizer parameters and training images cannot fit in GPU memory [S1, unofficial]. Peak memory occurs during joint training; inference memory is significantly lower (<2 GB) [S1, unofficial].
*   **Quality Degradation:** Joint learning across fundamentally unrelated datasets (e.g., beetle and argon bubble) degrades quality because the shared MLPs lack the capacity to simultaneously encode different volume structures [S1, unofficial]. Scene grouping based on Chamfer distance is essential [S1, unofficial].

## Subtopics

### [Joint-Hash Global Modeling](16257-joint-hash-global-modeling.md)
A pipeline for large-scale scenes using a shared VQC with global parameters $\theta$, conditioned on concatenated hash encodings of Gaussian positions and viewing directions. It extends the joint learning framework to handle global modeling by leveraging hash encodings for efficient conditioning.

### [Triplane-MLP Network Architecture](17335-triplane-mlp-network-architecture.md)
A model architecture using lightweight MLPs to predict appearance, geometry, and deformation parameters of individual Gaussians from triplane features, enabling efficient training and fast rendering (60 FPS) by discarding triplane/MLP models after predicting canonical avatar and deformation parameters. It sets itself apart by decoupling triplane/MLP inference from final rendering, optimizing for interactive speed.

### Fast Local Shape Updating
A strategy for interactive editing that performs test-time optimization only on the patch feature volumes of edited patches, fixing all other patch feature volumes and the shared MLP decoder $f$ [S2]. Updates the learned neural representation locally when the geometry is edited, avoiding full retraining [S2]. Pretrain the shared MLP decoder $f$ with various surface patches to instill a local shape prior. During pretraining, train the decoder $f$ and patch feature volume $FV_p$ to reconstruct patch $S_p$ using Eq. 4 [S2]. During editing, fix the shared decoder $f$ and all unedited patch feature volumes. Optimize only the patch feature volumes of the edited patches [S2]. Identify edited patches and connected patches affected by merge constraint updates. Update feature volumes only for these patches. Keep all other patch feature volumes fixed. Retrain until fitting accuracy is reached [S2]. Update time is less than 2 seconds for local updates [S2]. Takes an average of 1.52 seconds to reach fitting accuracy, halving the time compared to full Patch-Grid-TS training [S2].

### MLP Attribute Decoders
Predicts opacity, color, scale, and rotation for $k$ neural Gaussians per anchor using small MLPs conditioned on the view-adaptive feature [S3]. Feed the view-adaptive feature $\hat{f}_v$, relative viewing direction $\vec{d}_{vc}$, and distance $\delta_{vc}$ into each MLP branch [S3]. The output formulas are: $\{c_0, ..., c_{k-1}\} = \text{Sigmoid}(F_c)$ where $F_c$ is the color MLP [S3]; $\{s_0, ..., s_{k-1}\} = \text{Sigmoid}(F_s) \cdot s_v$ where $F_s$ is the scale MLP and $s_v$ is the base scaling of the anchor [S3]; $\{q_0, ..., q_{k-1}\} = \text{Normalize}(F_q)$ where $F_q$ is the rotation MLP, ensuring valid quaternions [S3]. Hidden dimension for all MLPs is 32 with architecture LINEAR → RELU → LINEAR [S3]. Opacity uses Tanh activation where 0 serves as the natural threshold for selecting valid samples, mapping to $[0, 1)$ [S3]. Color uses Sigmoid activation to constrain the output range to $(0, 1)$ [S3]. Rotation uses normalization activation to guarantee valid quaternion outputs [S3].

## Operating characteristics
*   **Time Complexity:** Clustering overhead scales quadratically with the number of scenes but is reduced by subsampling to 10,000 points [S1, unofficial]. For 41 scenes, clustering takes 0.49s, which is less than 0.02% of total training time [S1, unofficial]. Joint learning reduces total training time by 35% to 54% compared to separate optimization [S1, unofficial].
*   **Memory Complexity:** Model size is reduced by 44% to 60% compared to separate optimization [S1, unofficial]. Peak memory usage is high during joint training due to loading all anchors and optimizer states for a group, but inference memory is low (less than 2 GB) [S1, unofficial].
*   **Performance:** On the multivariate combustion dataset (41 scenes), joint learning reduced training time from 156.9 to 72.0 min, model size from 45.75 to 15.99 MB, and anchors from 1,049,074 to 415,275 [S1, unofficial]. Reconstruction quality (PSNR, LPIPS) remains nearly unchanged [S1, unofficial].
*   **Strengths:** Significant reductions in training time, model size, and anchor count. Savings grow monotonically with the number of basic scenes [S1, unofficial].
*   **Limitations:** Requires geometric similarity between scenes for effective grouping. Cannot handle fundamentally unrelated scenes in the same group without quality loss [S1, unofficial].

## Background
The method was introduced in the paper "ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization" [S1, unofficial]. It leverages the BiRF architecture as the context model, which uses binary hash grids and a shallow MLP [S1, unofficial]. The approach is designed for Volume Visualization (VolVis) scenes, addressing the redundancy in representing multiple similar volumetric datasets [S1, unofficial].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
[S2] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
[S3] Scaffold-GS: Structured 3D Gaussians for View-Adaptive Rendering, https://doi.org/10.1109/cvpr52733.2024.01952, fetched 2026-10-26
