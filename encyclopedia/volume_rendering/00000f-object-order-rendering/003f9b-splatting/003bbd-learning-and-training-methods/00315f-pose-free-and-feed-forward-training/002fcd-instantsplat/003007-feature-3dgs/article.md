# Feature-3DGS

Also called: N2F2

## Overview
Feature-3DGS is a method for open-vocabulary 3D segmentation that stores per-Gaussian language features, such as CLIP or LSeg embeddings [S1, unofficial][S2]. It is distinguished as the first method for feature field distillation based on 3D Gaussian Splatting, leveraging guidance from 2D foundation models to learn semantic features for each 3D Gaussian [S2]. It employs a low-dimensional feature field upsampled via a convolutional decoder to improve rendering efficiency [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse multi-view images, camera poses (or estimated via InstantSplat), and 2D foundation model features (e.g., CLIP, LSeg) extracted from input views.
*   **Outputs:** A set of 3D Gaussians, each associated with a low-dimensional semantic feature vector, and a convolutional decoder for upsampling.

**Data Structures:**
*   **3D Gaussian:** Standard 3DGS attributes (position, covariance, opacity, color) plus a new attribute: `feature_vector`.
*   **Feature Vector:** A low-dimensional embedding (dimension not specified in sources) representing the semantic content [S2].
*   **Convolutional Decoder:** A neural network module that upsamples the sparse feature field to a dense grid or directly to pixel space [S2].

**Steps:**
1.  **Initialization:** Initialize 3D Gaussians using geometry priors (e.g., from MASt3R/VGGT in InstantSplat) [S2].
2.  **Feature Learning:** Learn semantic features for each 3D Gaussian. This involves distilling features from 2D foundation models into the 3D Gaussian representation [S2].
3.  **Guidance:** Leverage guidance from 2D foundation models (e.g., CLIP, LSeg) during training to ensure semantic consistency [S2].
4.  **Rendering:** Employ a low-dimensional feature field. To render high-resolution semantic features, upsample the sparse feature field using a convolutional decoder [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Feature-3DGS rendering and training
# Note: Specific architectural details of the convolutional decoder and feature distillation loss are not provided in sources.

def forward_pass(gaussians, camera, decoder):
    """
    Render semantic features for a given view.
    """
    # 1. Project 3D Gaussians to 2D screen space
    projected_gaussians = project_gaussians(gaussians, camera)
    
    # 2. Gather low-dimensional feature vectors for each pixel
    # This involves alpha blending of the sparse feature field
    sparse_features = alpha_blend_features(projected_gaussians)
    
    # 3. Upsample sparse features using convolutional decoder
    # The decoder takes the sparse feature field and upsamples it
    dense_features = decoder(sparse_features)
    
    return dense_features

def train_step(gaussians, images, foundation_model_features, decoder, optimizer):
    """
    Train the 3D Gaussians and decoder using 2D foundation model guidance.
    """
    # 1. Render features for current view
    rendered_features = forward_pass(gaussians, camera, decoder)
    
    # 2. Compute loss against 2D foundation model features
    # The specific loss function (e.g., cosine similarity, MSE) is not detailed
    loss = compute_feature_loss(rendered_features, foundation_model_features)
    
    # 3. Backpropagate and update
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    return loss
```

**Parameters:**
*   **Feature Dimension:** Not given in the sources.
*   **Foundation Model:** CLIP or LSeg [S1, unofficial][S2].
*   **Decoder Architecture:** Convolutional decoder [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Low-dimensional feature fields may lose semantic detail; upsampling artifacts may occur if the decoder is not well-trained [S2].
*   **Boundaries:** Feature consistency across view boundaries must be maintained during distillation [S2].

## Subtopics

### Holistic Gaussian Initialization
Provides a robust initialization strategy for Gaussians, which can be combined with Feature-3DGS for better semantic alignment. [Holistic Gaussian Initialization](9776-holistic-gaussian-initialization.md)

### ReSplat
Focuses on re-splatting techniques for efficient rendering, potentially applicable to feature fields. [ReSplat](11511-resplat.md)

### AESplat
Uses adaptive entropy-based splatting, which could optimize the rendering of feature fields. [AESplat](14724-aesplat.md)

## Operating characteristics
*   **Time Complexity:** Rendering involves alpha blending of features and a convolutional upsampling step. The complexity depends on the number of Gaussians and the decoder's cost. Not given in the sources.
*   **Memory Complexity:** Stores a low-dimensional feature vector per Gaussian, which is more memory-efficient than storing high-dimensional RGB features. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables open-vocabulary 3D segmentation; efficient rendering via low-dimensional features and convolutional upsampling [S2].
*   **Limitations:** Relies on the quality of 2D foundation models; upsampling may introduce artifacts [S2].
*   **When to use:** When open-vocabulary segmentation is required and efficiency is a concern.
*   **When not to use:** When high-fidelity RGB rendering is the primary goal without semantic needs.

## Background
Feature-3DGS was introduced as the first method for feature field distillation based on 3D Gaussian Splatting [S2]. It builds on the work of 2D foundation models like CLIP and LSeg, adapting them to the 3D Gaussian representation [S1, unofficial][S2]. It is part of a series of works that leverage 2D priors for 3D tasks, following methods like SAGA and Gaussian Grouping [S2].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
