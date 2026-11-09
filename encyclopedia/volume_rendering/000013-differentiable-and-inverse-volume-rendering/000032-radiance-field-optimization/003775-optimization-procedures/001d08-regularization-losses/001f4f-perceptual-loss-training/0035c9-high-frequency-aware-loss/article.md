# High-Frequency Aware Loss

Also called: LHF, Shallow VGG Loss

## Overview
High-Frequency Aware Loss (LHF) is a perceptual regularization term designed to recover high-frequency details in rendered images while avoiding the texture distortion common in deeper perceptual losses. It sets itself apart from standard VGG-based perceptual losses by restricting feature extraction to the first block of the network, before any max-pooling operations, thereby preserving fine-grained spatial information [S1]. It builds on [Perceptual Loss Training](2000-perceptual-loss-training.md) by specifically targeting the loss of high-frequency content that occurs when using pixel-wise losses on misaligned data.

## How it works
**Inputs and Outputs:**
*   **Input:** A rendered patch $R$ and a ground truth patch $G_i$ (aligned to the rendered view).
*   **Output:** A scalar loss value $L_{HF}$.

**Data Structures:**
*   $F$: The feature extractor, defined as the first block of a pre-trained VGG-19 model. This block consists of convolutional layers followed by ReLU activations but excludes any max-pooling layers.
*   Feature maps: The output of $F$ has dimensions $C \times W \times H$, where $C$ is the number of channels, and $W, H$ are the width and height of the feature map.

**Parameters:**
*   $w$: A weighting scalar for the loss term (typical values not given in sources, but used in combination with patch MSE [S1]).
*   $F$: Fixed pre-trained weights from VGG-19.

**Steps:**
1.  Extract features from the rendered patch $R$ using the shallow VGG block: $F_R = F(R)$.
2.  Extract features from the aligned ground truth patch $G_i$: $F_{GT} = F(G_i)$.
3.  Compute the squared Euclidean distance between the feature maps.
4.  Normalize by the total number of elements in the feature map ($C \times W \times H$).

**Formula:**
$$L_{HF}(G_i, R) = \frac{1}{2CW H} \|F(G_i) - F(R)\|_2^2$$

Where:
*   $\| \cdot \|_2^2$ denotes the squared L2 norm (sum of squared differences).
*   $C, W, H$ are the dimensions of the feature maps produced by $F$.
*   $G_i$ is the ground truth patch after alignment.
*   $R$ is the rendered patch.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def high_frequency_aware_loss(rendered_patch, ground_truth_patch, vgg_block):
    """
    Computes the High-Frequency Aware Loss.
    
    Args:
        rendered_patch (Tensor): Rendered image patch R.
        ground_truth_patch (Tensor): Aligned ground truth patch G_i.
        vgg_block (Module): The first block of a pre-trained VGG-19 model 
                            (convolutions + ReLUs, no pooling).
                            
    Returns:
        Tensor: Scalar loss value L_HF.
    """
    # Extract features using the shallow VGG block
    # F denotes the first block of VGG-19
    features_rendered = vgg_block(rendered_patch)
    features_gt = vgg_block(ground_truth_patch)
    
    # Compute squared L2 difference
    diff = features_rendered - features_gt
    squared_diff = diff ** 2
    
    # Sum over all elements
    sum_squared_diff = torch.sum(squared_diff)
    
    # Get dimensions C, W, H
    C, W, H = features_rendered.shape[1], features_rendered.shape[2], features_rendered.shape[3]
    
    # Normalize by 2 * C * W * H
    loss = sum_squared_diff / (2 * C * W * H)
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **Misalignment:** The loss assumes $G_i$ is properly aligned with $R$. Misalignment prevents the recovery of high-frequency details, as the features will not match even if the content is correct [S1].
*   **Texture Distortion:** While deeper VGG losses distort textures, this shallow variant preserves them. However, it is still sensitive to noise in rendered images, which can introduce optimization bias [S2].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Depends on the forward pass of the first VGG-19 block. Since it avoids pooling and deeper layers, it is computationally cheaper than full perceptual losses.
*   **Memory Complexity:** Requires storing feature maps from the first VGG block for both the rendered and ground truth patches.
*   **Strengths:** Improves high-frequency details without distorting real textures [S1]. Effective when combined with patch-based MSE to handle misalignment [S1].
*   **Limitations:** Highly sensitive to noise in rendered images [S2]. Requires pre-aligned ground truth patches [S1].
*   **When to use:** When optimizing neural radiance fields or similar renderers where high-frequency detail recovery is critical and misalignment is handled via patch-based alignment [S1].
*   **When not to use:** When computational resources are extremely limited and full VGG is too heavy (though this is lighter, it still requires a pre-trained model). When the input images are very noisy, as it may introduce bias [S2].

## Background
The High-Frequency Aware Loss was introduced in the AligNeRF paper [S1]. It was developed to address the limitations of standard perceptual losses, which, while good at capturing high-frequency details, often distort the actual texture of objects [S1]. The authors modified the original perceptual loss proposed by Johnson et al. [12] by using only the output of the first block of VGG-19 before max-pooling. This approach was integrated into the AligNeRF training pipeline, replacing per-pixel MSE loss with a combination of patch-based MSE and this shallow VGG feature-space loss [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
