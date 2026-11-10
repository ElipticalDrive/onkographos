# LSeg Feature Encoding

Also called: LSeg

## Overview
LSeg Feature Encoding is a method for generating semantic feature maps from 3D scenes by distilling knowledge from the 2D vision-language model LSeg into a neural feature field. It solves the problem of open-vocabulary semantic understanding in 3D reconstruction by leveraging pretrained CLIP-based encoders as teacher networks. It differs from sibling methods like [3D Gaussian Splatting with Feature Vectors](8060-3d-gaussian-splatting-with-feature-vectors.md) by relying on external 2D segmentation models rather than learning features directly from multi-view images, and from [Text Embedding Generation with Directional Views](8239-text-embedding-generation-with-directional-views.md) by using a unified language-segmentation model rather than directional text embeddings.

## How it works
The method uses LSeg as a teacher network to provide target feature maps for distilling features into a NeRF or Gaussian Splatting scene. The process involves encoding text queries and image regions using LSeg's internal encoders, then using the resulting features as supervision targets.

**Inputs and Outputs:**
- **Input:** A 3D scene representation (e.g., NeRF or 3D Gaussians), a set of camera poses, and a text query string.
- **Output:** A 2D feature map $\mathbf{F}$ aligned with the rendered image, which is then used for segmentation or further 3D feature field training.

**Data Structures:**
- **Text Encoder:** CLIP’s ViT-B/32. Takes a text string and outputs a text embedding vector.
- **Image Encoder:** ViT-L/16. Takes an image patch or full image and outputs a feature map.
- **Feature Map:** A 2D grid of feature vectors, resized to the original image resolution via interpolation if necessary.

**Steps:**
1. **Text Encoding:** Encode the input text query using the CLIP ViT-B/32 text encoder to obtain a text embedding vector $\mathbf{t}$.
2. **Image Preprocessing:** Resize the input image to a scale factor $s$. The scale factors depend on the dataset:
   - For Replica: $s \in [0.75, 0.83, 0.92, 1.0, 1.08, 1.17, 1.25]$.
   - For other datasets: $s \in [0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.25]$.
3. **Multi-scale Inference:** For each scale $s$:
   a. Resize the image to $s \times$ original resolution.
   b. Pass the resized image through the ViT-L/16 image encoder to get a feature map.
   c. Compute the similarity between the image features and the text embedding $\mathbf{t}$ to generate a segmentation map.
4. **Feature Aggregation:** Resize the resulting feature maps from all scales back to the original image resolution using interpolation. Average these maps to produce the final feature map $\mathbf{F}$.
5. **Distillation:** Use $\mathbf{F}$ as a target to train the 3D feature field (e.g., NeRF or Gaussian attributes) via backpropagation.

**Parameters:**
- **Image Encoder:** ViT-L/16.
- **Text Encoder:** CLIP ViT-B/32.
- **Multi-scale Inference Scales:** See Step 2.
- **Interpolation:** Bilinear or nearest-neighbor (not specified, but standard for resizing).

**Edge Cases and Pitfalls:**
- **Out-of-Distribution (OOD) Data:** LSeg performs poorly on OOD regions and queries (e.g., "fossils", "T-Rex") due to bias in its training datasets (e.g., traffic scenes). Small objects or parts are often not discriminative [S1].
- **Large Scale Artifacts:** Large scale inference (zoomed/cropped images) increases effective resolution but loses context information, potentially producing noisy features with wrong semantic understanding [S1].
- **Training Bias:** The mixed dataset bias (especially towards traffic scenes) limits zero-shot ability in various domains [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lseg_feature_encoding(image, text_query, camera_pose, dataset_type):
    # 1. Text Encoding
    text_encoder = get_clip_vit_b_32()
    text_embedding = text_encoder.encode(text_query)
    
    # 2. Define scales based on dataset
    if dataset_type == "Replica":
        scales = [0.75, 0.83, 0.92, 1.0, 1.08, 1.17, 1.25]
    else:
        scales = [0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.25]
        
    # 3. Multi-scale Inference
    original_height, original_width = image.shape[:2]
    aggregated_features = None
    
    for scale in scales:
        # Resize image
        scaled_image = resize(image, scale=scale)
        
        # Image Encoding
        image_encoder = get_vit_l_16()
        # Note: LSeg official demo model handles the text-image similarity internally
        # or we compute it manually. The source implies using the official demo model.
        # Assuming the demo model returns a feature map or segmentation map.
        # If it returns a feature map, we need to compute similarity with text_embedding.
        # The source says "Inference follows the official script", which likely handles
        # the text query and returns a segmentation map or feature map.
        
        # For the purpose of this pseudocode, we assume the model returns a feature map
        # or a similarity map. Let's assume it returns a feature map F_scaled.
        # In reality, LSeg computes pixel-wise similarity between image features and text features.
        
        # Using the official demo model's inference logic:
        # The model takes the image and text query and outputs a segmentation map.
        # However, for feature distillation, we might need the raw feature map.
        # The source mentions "feature maps are of reduced sizes... resized to original".
        # This implies we are using the image encoder's output.
        
        # Let's assume we get the image features from the ViT-L/16 encoder.
        # The ViT-L/16 encoder outputs a feature map of size (H/scale, W/scale, D).
        image_features = image_encoder.encode(scaled_image)
        
        # Compute similarity between image features and text embedding
        # This step is implicit in LSeg's segmentation head, but for feature distillation,
        # we might use the image features directly or the similarity map.
        # The source says "fimg (I, r), defined in Equation 4".
        # Assuming we get a feature map F_scaled of size (H/scale, W/scale, D).
        
        # Resize to original size
        resized_features = resize(image_features, scale=1.0/scale)
        
        if aggregated_features is None:
            aggregated_features = resized_features
        else:
            aggregated_features = aggregated_features + resized_features
            
    # Average the features
    final_feature_map = aggregated_features / len(scales)
    
    return final_feature_map
```

## Subtopics

### LSeg Teacher Network for Feature Distillation
Uses LSeg as a teacher network to distill features into NeRF, leveraging pretrained CLIP models as text and image feature encoders. Chosen for simplicity of implementation. Other concurrent models or text-unconditional encoders can be used. LSeg is not robust to out-of-distribution data (e.g., 'fossils', 'T-Rex', 'Triceratops') due to overfitting to bias in its training datasets (e.g., traffic scenes). LSeg by Li et al. appeared anonymously at OpenReview on September 29th, 2021. Other concurrent studies appeared at arXiv on November or December, 2021.

## Operating characteristics
- **Time Complexity:** Depends on the number of scales and the size of the ViT-L/16 model. Multi-scale inference increases computational cost but stabilizes features.
- **Memory Complexity:** Requires storing the feature maps for all scales during inference.
- **Strengths:** Leverages powerful 2D vision-language models for open-vocabulary queries. Simple to implement using existing LSeg models.
- **Limitations:** Poor performance on out-of-distribution data. Large scale inference can lose context information.
- **When to Use:** When open-vocabulary semantic understanding is required and the scene contains objects similar to those in the training datasets (e.g., traffic scenes).
- **When Not to Use:** When the scene contains out-of-distribution objects or when context information is critical for semantic understanding.

## Background
LSeg [44] by Li et al. appeared anonymously at OpenReview on September 29th, 2021. Other concurrent studies [52, 99, 103, 72] appeared at arXiv on November or December, 2021. LSeg uses pretrained CLIP models as a text encoder and an image feature encoder in freezing or finetuning manners to train zero-shot image semantic segmentation models. The model is trained on seven different datasets: ADE20K, BDD, Cityscapes, COCO-Panoptic, IDD, Mapillary Vistas, and SUN RGBD.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
