# Nearest Neighbor Feature Matching Loss

Also called: NNFM Loss

## Overview
Nearest Neighbor Feature Matching (NNFM) is a style loss function that minimizes the distance between local feature vectors of a rendered image and the nearest neighbor feature vectors in a style image's feature map [S1]. It solves the problem of capturing distinctive local details, such as brushstrokes, which global statistics (e.g., Gram matrices) fail to preserve faithfully [S1]. Unlike [Perceptual Loss](13426-perceptual-loss.md) which typically minimizes the distance to a fixed reference feature, NNFM dynamically matches each rendered feature to its closest counterpart in the style feature space, enabling high-frequency detail transfer while maintaining multi-view consistency in 3D radiance fields [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $I_{render}$: The image rendered from the radiance field at a specific viewpoint.
    *   $I_{style}$: The 2D style image.
    *   $I_{content}$: The original content image (used for the content-preserving term).
    *   $\lambda$: Weight for the content-preserving loss.
*   **Outputs:**
    *   Scalar loss value $\ell$ to be minimized during optimization.

**Data Structures:**
Feature maps $F_{render}$, $F_{style}$, and $F_{content}$ are extracted using the `conv3` block of a pretrained VGG-16 network [S1].
*   **VGG-16 Structure:** Consists of 5 layer blocks (conv1 to conv5). Each block begins with a max-pooling layer that downsamples the feature map by a factor of 2. Inside each block, feature maps share the same spatial resolution [S1].
*   **Feature Map Layout:** $F_{render}$ is a tensor of shape $(H, W, C)$, where $H$ and $W$ are the height and width of the rendered image, and $C$ is the number of channels in the `conv3` block. The same dimensions apply to $F_{style}$ and $F_{content}$ [S1].
*   **Feature Vector:** $F(i, j)$ denotes the feature vector at pixel location $(i, j)$, with shape $(C,)$ [S1].

**Numbered Steps:**
1.  **Feature Extraction:** Extract feature maps $F_{render}$, $F_{style}$, and $F_{content}$ from their respective input images using the `conv3` block of a pretrained VGG-16 network [S1].
2.  **Nearest Neighbor Search:** For each pixel location $(i, j)$ in $F_{render}$, find the pixel location $(i', j')$ in $F_{style}$ that minimizes the cosine distance between their feature vectors [S1].
3.  **Distance Calculation:** Compute the cosine distance $D$ between the rendered feature vector $\boldsymbol{F}_{render}(i, j)$ and its nearest neighbor $\boldsymbol{F}_{style}(i', j')$ [S1].
4.  **Aggregation:** Average the minimum distances over all pixels $N$ in $F_{render}$ to compute the NNFM loss $\ell_{\textrm{nnfm}}$ [S1].
5.  **Content Preservation:** Compute the L2 difference between $F_{render}$ and $F_{content}$ [S1].
6.  **Final Loss:** Combine the NNFM loss and the content loss using the weight $\lambda$ [S1].

**Formulas:**
The NNFM loss is defined as [S1]:
$$
\ell _{\textrm {nnfm}}({\boldsymbol {F}_{\textrm {render}}}, \boldsymbol {F}_{\textrm {style}})=\frac {1}{N}\sum _{i, j} \min _{i', j'} D\big ({\boldsymbol {F}_{\textrm {render}}}(i, j),\boldsymbol {F}_{\textrm {style}}(i',j')\big )
$$
where $N$ is the number of pixels in $F_{render}$ [S1].

The cosine distance $D$ between two feature vectors $\boldsymbol{v}_1$ and $\boldsymbol{v}_2$ is [S1]:
$$
D(\boldsymbol {v}_1, \boldsymbol {v}_2)=1-\frac{\boldsymbol {v}_1^T\boldsymbol {v}_2}{\sqrt {\boldsymbol {v}_1^T\boldsymbol {v}_1 \boldsymbol {v}_2^T\boldsymbol {v}_2}}
$$

The final loss combines the NNFM loss with a content-preserving L2 loss [S1]:
$$
\ell =\ell _{\textrm {nnfm}}({\boldsymbol {F}_{\textrm {render}}}, \boldsymbol {F}_{\textrm {style}})+\lambda \cdot \ell _2({\boldsymbol {F}_{\textrm {render}}}, \boldsymbol {F}_{\textrm {content}})
$$
where $\ell _2({\boldsymbol {F}_{\textrm {render}}}, \boldsymbol {F}_{\textrm {content}})$ is the L2 difference between the feature maps [S1].

**Parameters:**
*   $\lambda$: Controls stylization strength. A larger $\lambda$ preserves more content, while a smaller $\lambda$ leads to stronger stylization [S1].
    *   Typical value: $\lambda = 0.001$ for forward-facing captures [S1].
    *   Typical value: $\lambda = 0.005$ for 360° captures [S1].
*   **Scale:** Stylization is applied only at the final scale (full resolution), not using a coarse-to-fine approach [S1].
*   **Augmentations:** Style image augmentations (rotation and/or scaling) are skipped [S1].

**Edge Cases and Pitfalls:**
*   **Over-stylization:** Using the NNFM loss alone can lead to overly strong stylization, making the content harder to recognize. The content-preserving loss term is necessary to balance this [S1].
*   **Memory Constraints:** Volumetric radiance field rendering consumes significant memory. Computing style losses on full-resolution rendered images is memory-intensive. A deferred back-propagation method is required: disable auto-differentiation to render the full-resolution image, compute the loss, and then back-propagate [S1].
*   **View-Dependence:** To ensure consistent stylization, view-dependent appearance modeling should be discarded (e.g., by zeroing out view directions in MLPs or keeping only the first spherical harmonic component) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def cosine_distance(v1, v2):
    """
    Computes cosine distance between two vectors or batches of vectors.
    v1, v2: tensors of shape (..., C)
    """
    # Normalize vectors
    v1_norm = torch.nn.functional.normalize(v1, p=2, dim=-1)
    v2_norm = torch.nn.functional.normalize(v2, p=2, dim=-1)
    # Cosine similarity
    sim = torch.sum(v1_norm * v2_norm, dim=-1)
    # Cosine distance = 1 - similarity
    return 1.0 - sim

def nnfm_loss(F_render, F_style, F_content, lambda_weight):
    """
    Computes the Nearest Neighbor Feature Matching Loss.
    
    Args:
        F_render: Tensor of shape (H, W, C) or (B, H, W, C) containing rendered features.
        F_style: Tensor of shape (H_s, W_s, C) or (B, H_s, W_s, C) containing style features.
        F_content: Tensor of shape (H, W, C) or (B, H, W, C) containing content features.
        lambda_weight: Float, weight for the content loss.
        
    Returns:
        Scalar loss value.
    """
    # Flatten spatial dimensions to get list of feature vectors
    # Handle both single image and batched inputs
    if F_render.dim() == 3:
        H, W, C = F_render.shape
        F_render_flat = F_render.view(-1, C) # Shape (N, C)
        F_content_flat = F_content.view(-1, C)
        # Style features are typically from a single reference image
        H_s, W_s, C_s = F_style.shape
        F_style_flat = F_style.view(-1, C_s) # Shape (N_s, C)
    else:
        B, H, W, C = F_render.shape
        F_render_flat = F_render.view(B, -1, C) # Shape (B, N, C)
        F_content_flat = F_content.view(B, -1, C)
        B_s, H_s, W_s, C_s = F_style.shape
        F_style_flat = F_style.view(B_s, -1, C_s) # Shape (B, N_s, C)
        
        # Ensure batch sizes match or handle single style image
        if B != B_s:
            # Broadcast style features to match render batch if needed
            # Assuming F_style is a single reference for all renders in batch
            F_style_flat = F_style_flat.expand(B, -1, -1)

    # Compute pairwise cosine distances
    # dist[i, j] = distance between F_render_flat[i] and F_style_flat[j]
    # Using broadcasting: (N, 1, C) and (1, N_s, C) -> (N, N_s)
    dist = cosine_distance(F_render_flat.unsqueeze(1), F_style_flat.unsqueeze(0))
    
    # Find minimum distance for each rendered feature vector
    min_dists, _ = torch.min(dist, dim=1) # Shape (N,) or (B, N)
    
    # Average over all pixels
    nnfm_loss_val = torch.mean(min_dists)
    
    # Compute L2 content loss
    content_loss_val = torch.mean((F_render_flat - F_content_flat) ** 2)
    
    # Combine losses
    total_loss = nnfm_loss_val + lambda_weight * content_loss_val
    
    return total_loss
```

## Subtopics

### Deferred Back-Propagation
A technique to handle memory-intensive style losses on full-resolution rendered images by disabling auto-differentiation during rendering and enabling it only for loss computation and back-propagation [S1].

### VGG-16 conv3 Block
The specific feature extractor used for NNFM, chosen empirically for capturing style details better than other blocks [S1].

## Operating characteristics
*   **Time Complexity:** The nearest neighbor search requires computing distances between all rendered feature vectors and all style feature vectors, resulting in $O(N \cdot N_s)$ complexity per iteration, where $N$ is the number of pixels in the rendered image and $N_s$ is the number of pixels in the style image [S1].
*   **Memory Complexity:** Requires storing full-resolution feature maps for the rendered image, style image, and content image simultaneously during the loss computation phase [S1]. The deferred back-propagation method is necessary to avoid out-of-memory errors during the rendering phase [S1].
*   **Strengths:** Captures high-frequency visual details and local structures (e.g., brushstrokes) more effectively than global statistics-based losses like Gram matrices [S1]. Maintains multi-view consistency when applied to radiance fields [S1].
*   **Limitations:** Computationally expensive due to the nearest neighbor search. Can lead to overly strong stylization if not balanced with a content loss [S1].
*   **When to use:** When high-fidelity transfer of local stylistic details (like texture and brushstrokes) is required, and computational resources allow for full-resolution feature matching [S1].
*   **When not to use:** When real-time performance is critical, or when memory constraints prevent storing full-resolution feature maps [S1].

## Background
The Nearest Neighbor Feature Matching (NNFM) loss was introduced in the "ARF: Artistic Radiance Fields" paper by K. Zhang et al. [S1]. It was proposed to address the limitations of previous VGG-based style losses (which relied on Gram matrices) that produced blurry results and failed to capture faithful brushstrokes in 3D stylization [S1]. The method was adapted from 2D style transfer techniques [S1] but optimized for 3D radiance fields by applying stylization only at the final scale and skipping style image augmentations [S1].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
