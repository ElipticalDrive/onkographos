# Transformer and Vision Model Integration

Also called: Vision Transformer NeRF, TransNeRF, VisionNeRF

## Overview
This family integrates Vision Transformers (ViT) and general Transformer architectures into Neural Radiance Fields to leverage global semantic understanding and long-range dependencies for novel view synthesis. It solves the limitations of standard MLP-based NeRFs, which lack explicit global scene modeling and struggle with single-image inputs or occlusions [S1]. Unlike sibling methods that rely on per-scene optimization or canonical coordinate systems, these models either generalize across scenes without per-scene tuning or explicitly model self-occlusion via attention mechanisms to retrieve features from distant parts of the source view [S1, unofficial].

## Shared principles
The core mechanism replaces or augments the standard NeRF input pipeline with vision-language or vision-only transformer encoders to extract conditioning features.

1.  **Global Feature Extraction:** Input images are processed by a ViT encoder. The image is split into patches, projected to tokens, and passed through self-attention layers to capture global context [S1]. The class token is typically discarded as it lacks spatial meaning for feature maps [S1].
2.  **Feature Map Construction:** Latent tokens are unflattened and resampled into multi-level 2D feature maps (coarse-to-fine) to preserve spatial hierarchy [S1].
3.  **Local Feature Extraction:** A 2D CNN branch extracts local appearance details to complement the global ViT features, preventing blurriness and color inconsistency near the input view [S1].
4.  **Feature Fusion:** Global and local features are fused (e.g., via convolution) into a hybrid feature map [S1]. For multi-view methods, features from multiple source views are aggregated via attention [S1, unofficial].
5.  **Conditioned Rendering:** A NeRF MLP takes 3D coordinates, viewing directions, and the sampled hybrid features to predict density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].

## Subtopics

### [ViT-NeRF](13878-vit-nerf.md)
Combines a ViT for global shape and a 2D CNN for local appearance, conditioning volume rendering on hybrid per-pixel features. It avoids the need for a canonical coordinate system required by methods like CodeNeRF and produces sharper edges than SRN and PixelNeRF.

### [VisionNeRF](13885-visionnerf.md)
Infers 3D representation from a single unposed image by combining global ViT features and local CNN features. It distinguishes itself by explicitly addressing self-occlusion in single-image settings through long-range dependency modeling, allowing it to retrieve relevant features from distant parts of the source view for occluded regions, whereas standard models query features along the ray corresponding to the visible surface.

### [TransNeRF](13907-transnerf.md)
A generalizable method that learns a generic radiance field using an encoder-decoder Transformer framework, avoiding per-scene optimization. It differs from pooling-based multi-view representations by exploring deep relationships amongst multiple views and aggregating multi-view information into a coordinate-based scene representation via attention in a single Transformer-based NeRF.

## Background
The integration of Transformers into NeRFs emerged to address the inductive bias of CNNs and the lack of global context in MLPs. ViT-NeRF [S1] and VisionNeRF [S1] were developed to improve single-image novel view synthesis by leveraging pre-trained vision models (e.g., DINO) for robust feature extraction. TransNeRF [S1, unofficial] extended this to multi-view generalizable NeRFs, enabling zero-shot synthesis on unseen scenes by aggregating multi-view information via attention mechanisms.

## Sources
[S1] ViT-NeRF: Vision Transformer for Neural Radiance Fields, URL, 2023-10-01
[S1] TransNeRF: Transforming Radiance Fields into Transformers, URL, 2023-10-01 (unofficial)
