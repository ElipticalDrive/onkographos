# AligNeRF Training Objective

Also called: Alignment-Aware Patch Matching Loss + High-Frequency Aware Loss

## Overview
AligNeRF replaces the standard per-pixel Mean Squared Error (MSE) photometric loss with a composite objective combining a patch-based MSE loss and a high-frequency-aware feature loss [S1]. This approach solves the problem of high-frequency detail loss caused by camera or scene motion misalignment between training views [S1]. Unlike standard NeRF training which assumes static scenes, AligNeRF explicitly accounts for misalignment by comparing local patches rather than single pixels, and by operating in a shallow VGG feature space to preserve texture [S1].

## How it works
The method modifies the photometric term $\mathcal{L}_{\text{photo}}$ in the total loss equation defined by [Specific Loss Formulations](10636-specific-loss-formulations.md).

**Inputs and Outputs:**
*   **Inputs:** Rendered image patch $I_{render}$, ground truth image patch $I_{gt}$, rendered feature map $F_{render}$, ground truth feature map $F_{gt}$.
*   **Outputs:** Scalar loss values $L_{PM}$ and $L_{HF}$.

**Data Structures:**
*   Patches are extracted from the rendered and ground truth images. The source does not specify the exact patch size or stride, only that they are "patch-based" [S1].
*   Feature maps are extracted from the first block of a pre-trained VGG-19 network [S1].

**Formulas:**
The total loss for this component is:
$$ L_{total} = L_{PM} + w \cdot L_{HF} $$

Where:
*   $L_{PM}$ is the Patch-based MSE loss.
*   $L_{HF}$ is the High-Frequency feature loss.
*   $w$ is the weight for the high-frequency loss.

**Parameters:**
*   $w$: Weight for the high-frequency loss. Typical value: 0.05 [S1].

**Steps:**
1.  **Patch Extraction:** Extract local patches from the rendered image and the corresponding ground truth image.
2.  **Patch MSE Calculation ($L_{PM}$):** Compute the MSE between the rendered patch and the ground truth patch. This accounts for misalignment by averaging error over a local neighborhood rather than a single pixel [S1].
3.  **Feature Extraction:** Pass the rendered image and ground truth image through the first block of a pre-trained VGG-19 model to obtain feature maps [S1].
4.  **Feature Loss Calculation ($L_{HF}$):** Compute the MSE between the rendered feature map and the ground truth feature map. Let $C, W, H$ be the dimensions of the extracted feature maps [S1].
5.  **Weighted Sum:** Combine the losses using the weight $w$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alignerf_loss(rendered_image, ground_truth_image, w=0.05):
    # 1. Patch-based MSE Loss (L_PM)
    # Note: Exact patch size and extraction logic are not specified in sources [S1]
    # Assuming a function extract_patches(image, patch_size, stride) exists
    rendered_patches = extract_patches(rendered_image, patch_size=7, stride=1) # [G]
    gt_patches = extract_patches(ground_truth_image, patch_size=7, stride=1) # [G]
    
    # Compute MSE between corresponding patches
    L_PM = mean_squared_error(rendered_patches, gt_patches)
    
    # 2. High-Frequency Feature Loss (L_HF)
    # Extract features from the first block of pre-trained VGG-19
    # Note: The specific VGG-19 layer indices for "first block" are not explicitly defined in sources [S1]
    # Assuming a function vgg_first_block_features(image) exists
    rendered_features = vgg_first_block_features(rendered_image) # [G]
    gt_features = vgg_first_block_features(ground_truth_image) # [G]
    
    # C, W, H are dimensions of the feature maps [S1]
    # Compute MSE between feature maps
    L_HF = mean_squared_error(rendered_features, gt_features)
    
    # 3. Combine losses
    L_total = L_PM + w * L_HF
    
    return L_total
```

## Subtopics

### 2D Mask Matching
Used in Instance-NeRF to ensure multi-view consistency of 2D instance masks aligned to a 3D instance detected by NeRF-RCNN [S2]. It uses a Mask2Former panoptic segmentation model pretrained on COCO to create initial 2D segmentation, filters background masks, projects 3D masks from NeRF-RCNN to image space, and assigns predicted masks to the 3D instance with the highest Intersection over Union (IoU) [S2]. Predicted masks with IoU $\le 0.05$ with any projected mask are treated as unlabeled areas and excluded from training [S2].

## Operating characteristics
*   **Strengths:** Improves recovery of high-frequency details in scenes with camera or scene motion by mitigating the blurring effects of misalignment [S1].
*   **Limitations:** Requires the use of a pre-trained VGG-19 network for feature extraction, adding computational overhead compared to pure pixel-wise MSE [S1].
*   **Performance:** Experiments on mip-NeRF 360 outdoor scenes show that AligNeRF recovers significantly more high-frequency detail than standard mip-NeRF 360, even when the standard model is scaled up with 4x more channels [S1].

## Background
AligNeRF was introduced in the paper "AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training" [S1]. It was developed to address the limitations of mip-NeRF 360 in unbounded real-world scenes with complex textures and subtle motion [S1]. The method serves as a plug-and-play modification to existing NeRF training pipelines [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
