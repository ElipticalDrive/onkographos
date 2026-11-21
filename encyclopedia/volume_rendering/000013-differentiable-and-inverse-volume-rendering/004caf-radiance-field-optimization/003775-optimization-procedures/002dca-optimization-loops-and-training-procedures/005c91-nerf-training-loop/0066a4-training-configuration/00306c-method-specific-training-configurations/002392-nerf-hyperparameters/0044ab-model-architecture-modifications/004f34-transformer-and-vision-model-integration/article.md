# Transformer and Vision Model Integration

Also called: Vision-Transformer NeRF, ViT-NeRF family

## Overview
This family integrates Vision Transformers (ViT) and other vision backbones into Neural Radiance Fields to improve feature extraction from input images. It solves the limitation of standard NeRF architectures, which rely on simple positional encoding and lack semantic understanding, by providing rich global or local image features to condition the radiance field . Unlike pure geometric methods, these models leverage the self-attention mechanism of Transformers to capture long-range dependencies, addressing issues like self-occlusion and inconsistent appearance in novel view synthesis .

## Shared principles
The core mechanism replaces or augments the standard positional encoding $\gamma(\mathbf{x})$ and directional encoding $\gamma(\mathbf{d})$ with features extracted from a pre-trained or fine-tuned vision model.

1.  **Feature Extraction:** An input image $I$ is processed by a vision backbone (ViT, CNN, or hybrid) to produce feature maps or tokens.
2.  **Spatial Alignment:** For a 3D point $\mathbf{x}$ sampled along a ray, its 2D projection $\pi(\mathbf{x})$ on the input image plane is computed.
3.  **Feature Query:** The feature value $F(\pi(\mathbf{x}))$ is sampled from the extracted feature maps at the projected location.
4.  **Conditioning:** The radiance field MLP takes the spatial position $\mathbf{x}$, viewing direction $\mathbf{d}$, and the sampled feature $F(\pi(\mathbf{x}))$ as input to predict density $\sigma$ and color $\mathbf{c}$ .

## Subtopics

### [ViT-NeRF](13878-vit-nerf.md)
Combines a ViT for global shape information and a 2D CNN for local appearance. It fuses these into a hybrid feature map to condition rendering. It is chosen when single-view synthesis requires both global semantic consistency and local detail, avoiding the inconsistent appearance of pure global methods.

### [VisionNeRF](13885-visionnerf.md)
Infers a 3D radiance field from a single *unposed* image. It uses ViT self-attention to retrieve features for occluded regions, mitigating self-occlusion degradation. It is chosen when camera poses are unknown and occlusion handling is critical.

### [TransNeRF](13907-transnerf.md)
A generalizable method that learns a generic scene representation from multiple source views without per-scene optimization. It uses an encoder-decoder Transformer to explore deep relationships among source views. It is chosen for cross-scene generalization where per-scene training is prohibitive.

### [NeRF Analogies](18133-nerf-analogies.md)
Transfers visual attributes (appearance) from a source NeRF to a target NeRF using dense semantic correspondences from ViT features. It is chosen for appearance transfer without requiring geometric alignment between scenes.

## Background
The integration of Transformers into NeRFs emerged to address the inability of standard MLPs to leverage high-level semantic information from input images. Early works like [ViT-NeRF](13878-vit-nerf.md) demonstrated that combining global ViT features with local CNN features improved rendering quality over pure global or local approaches . Subsequent works extended this to unposed single-view synthesis [VisionNeRF](13885-visionnerf.md) and generalizable multi-view settings [TransNeRF](13907-transnerf.md). The use of semantic features for cross-scene manipulation, such as in [NeRF Analogies](18133-nerf-analogies.md), further expanded the utility of vision models beyond direct rendering .

## Sources
[S1] Transformer and Vision Model Integration Notes, 2023
