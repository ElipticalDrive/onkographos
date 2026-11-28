# Semantic Voxel Labelling
Also called: Categorical Bayesian Fusion, Closed-Set Semantic Fusion via Dirichlet-Categorical Conjugacy, Dirichlet Update, Voxel semantic labeling
## Overview
Semantic Voxel Labelling assigns a discrete integer class label to every voxel in a 3D grid, mapping spatial data to semantic categories [S2]. Unlike continuous transfer functions, it produces deterministic integer identifiers rather than optical properties, serving as the foundational step for label-based rendering and downstream tasks like 3D object detection [S2]. It differs from [Semantic Label Colormap Generation](4667-semantic-label-colormap-generation.md) by outputting raw IDs rather than colors, and from [Semantic Transfer Function Design](9493-semantic-transfer-function-design.md) by focusing on the prediction of the label field itself rather than the mapping of that field to opacity [G].
## How it works
**Inputs and Outputs:**
- Input: A 3D grid $G$ of dimensions $W \times H \times D \times 4$ (typically containing features or raw data) [S2].
- Output: A label grid where each voxel $(i, j, k)$ is assigned an integer $S_{i,j,k} \in \{1, 2, \dots, n\}$, where $n$ is the total number of semantic categories [S2].

**Data Structures:**
- The output is a dense 3D array of integers [S2]. Each element corresponds to a voxel coordinate [S2].
- Interpolation is avoided during classification to prevent "color bleeding" between distinct semantic regions, though nearest-neighbor interpolation is required during ray casting to maintain label integrity [G].

**Algorithm:**
1. **Feature Extraction:** Process the input grid $G$ using a neural network encoder-decoder architecture (e.g., U-Net style) to produce feature representations for each voxel [S2].
2. **Label Prediction:** For each voxel at coordinates $(i, j, k)$, predict the class label $S_{i,j,k}$ [S2].
3. **Bayesian Update (Alternative/Complementary):** Maintain a Dirichlet distribution over class probabilities for each voxel. Initialize parameters $\alpha_j$. For each new observation of class $j$, update $\alpha_j \leftarrow \alpha_j + 1$ [S1]. Assign the label as the class with the highest posterior mean or mode [S1].
4. **Constraint:** Ensure $S_{i,j,k}$ is an integer such that $1 \le S_{i,j,k} \le n$ [S2].

**Formulas:**
The label assignment domain is defined as:
$$ S_{i,j,k} \in \{1, 2, \dots, n\}, \quad \text{for } 1 \le i \le W, 1 \le j \le H, 1 \le k \le D $$
where $n$ is the number of semantic categories [S2].

The Dirichlet probability density function used in the Bayesian fusion variant is:
$$ p(\boldsymbol{\theta} \mid \boldsymbol{\alpha}) = \frac{\Gamma(\sum_j \alpha_j)}{\prod_j \Gamma(\alpha_j)} \prod_j \theta_j^{\alpha_j - 1} $$
where $\boldsymbol{\alpha}$ are the hyperparameters and $\boldsymbol{\theta}$ represents the class probability vector [S1].

**Parameters:**
- $n$: The number of semantic categories [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def semantic_voxel_labelling(input_grid, num_classes):
    # Step 1: Feature extraction via U-Net style encoder-decoder network [S2]
    features = encoder_decoder_network(input_grid)
    
    # Step 2: Predict class logits for each voxel [S2]
    logits = classification_head(features)
    
    # Step 3: Argmax to get integer labels, 1-indexed [S2]
    labels = []
    for i in range(len(features)):
        row = []
        for j in range(len(features[0])):
            col = []
            for k in range(len(features[0][0])):
                max_idx = max(range(num_classes), key=lambda c: logits[i][j][k][c])
                col.append(max_idx + 1)
            row.append(col)
        labels.append(row)
        
    # Clamp to valid range [1, num_classes] [G]
    for i in range(len(labels)):
        for j in range(len(labels[0])):
            for k in range(len(labels[0][0])):
                labels[i][j][k] = max(1, min(num_classes, labels[i][j][k]))
    return labels

def encoder_decoder_network(grid):
    # Simulates U-Net style feature extraction [S2]
    return [[[ [0.0] * 64 for _ in range(len(grid[0][0])) ] for _ in range(len(grid[0])) ] for _ in range(len(grid))]

def classification_head(features):
    # Projects features to class logits [G]
    return [[[ [0.0] * 64 for _ in range(len(features[0][0])) ] for _ in range(len(features[0])) ] for _ in range(len(features))]
```

**Edge Cases and Pitfalls:**
- **Generalization:** Due to the diversity of objects in scenes, strong priors are required for the model to generalize across different environments [S2].
- **Upsampling:** Direct querying of implicit MLPs for higher resolution is slow; grid-based upsampling (e.g., via $V_{\text{upsample}} = g(V_{\text{input}}, \text{resolution})$) is preferred for efficiency [S2].
- **Precision/Boundaries:** Labels are strictly integers; floating-point interpolation must be avoided during classification to prevent semantic bleeding [G].

## Subtopics

### [Semantic Loss](15403-semantic-loss.md)
A training objective used to enforce consistency between rendered semantic labels and provided ground truth labels. It solves the problem of aligning 3D semantic predictions with 2D image-space annotations without requiring dense 3D voxel-level ground truth. It differs from standard geometric losses by operating on the discrete class probability distribution rather than continuous RGB or depth values.
- **Formula:**
  $$ L_{\text{sem}}(I) = - \sum_{r \in R} \sum_{l=1}^{L} p_l(r) \log \hat{p}_l(r) $$
  where $R$ is the set of all rays emitted from the query image $I$, $L$ is the number of classes ($1 \le l \le L$), $p_l(r)$ is the ground truth probability, and $\hat{p}_l(r)$ is the predicted probability.
- **Context:** Used in conjunction with Semantic-aware Weight Networks to rescore the significance of source views and construct final semantic rays.

### Open-Vocabulary Semantic Label Generation
A method to generate 2D semantic segmentation labels without ground truth data using a pretrained open-vocabulary model (Grounded-SAM) based on category names [S3]. It determines prompts using synonymous substitution, splitting single words into multiple entities, and incorporating additional information [S3]. It obtains detection bounding boxes and logits from Grounding DINO, feeds them to SAM to generate binary masks, and multiplies logits with masks [S3]. The per-pixel label is $S_{\text{pix}} = \psi(\arg \max_i l_i)$, where $\psi(\cdot)$ maps the index to the category label according to phrases; pixels with all zero logits receive an 'uncertain' label [S3]. Text and box thresholds for Grounding DINO are 0.2 [S3].

## Operating characteristics
- **Time Complexity:** Depends on the neural network inference time for the encoder-decoder network [S2]. Grid-based upsampling is faster than querying implicit MLPs for higher resolutions [S2].
- **Memory Complexity:** Requires storing the full 3D label grid $W \times H \times D$ [S2].
- **Strengths:** Provides discrete, interpretable semantic information suitable for downstream tasks like 3D object detection [S2].
- **Limitations:** Requires strong priors for generalization across diverse scenes [S2].

## Background
Semantic Voxel Labelling is a technique used in 3D representation learning, particularly within Neural Radiance Fields (NeRFs) [S2]. It was highlighted in the context of self-supervised learning using Masked AutoEncoders (NeRF-MAE) to improve downstream 3D object detection performance on datasets like Front3D and ScanNet [S2]. The method addresses the need for fast grid upsampling compared to implicit MLP queries [S2]. Additionally, Bayesian fusion approaches leverage the Dirichlet-Categorical conjugacy to recursively update voxel class probabilities from sensor measurements [S1].

## Sources
[S1] SLIM-VDB: A Real-Time 3D Probabilistic Semantic Mapping Framework, https://doi.org/10.1109/lra.2026.3652875, fetched 2026-10-26
[S2] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
[S3] OccNeRF: Advancing 3D Occupancy Prediction in LiDAR-Free Environments, https://doi.org/10.1109/tip.2025.3567828, fetched 2026-10-29
