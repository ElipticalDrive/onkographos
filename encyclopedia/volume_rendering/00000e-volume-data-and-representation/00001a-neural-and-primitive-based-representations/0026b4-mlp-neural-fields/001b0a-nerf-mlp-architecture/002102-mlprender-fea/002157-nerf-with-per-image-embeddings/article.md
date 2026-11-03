# NeRF with Per-Image Embeddings

Also called: GLO Model Variant, NeRF with Global Lighting Optimization Embeddings

## Overview
A model variant that learns a per-image embedding for each scene and injects it into the NeRF MLP during color prediction [S1]. Unlike Stable View Synthesis (SVS), which modifies scene brightness and color to explain camera variation, this method neutralizes photometric variation during training, yielding consistent global brightness and color across views [S1]. It builds on [MLPRender_Fea](2414-mlprender-fea.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images with known camera poses; a set of learnable per-image embedding vectors.
*   **Outputs:** RGB color values for rendered pixels.

**Data Structures:**
*   **Per-Image Embeddings:** A learnable vector $e_i$ for each image $i$ in the training set. The length of this vector is not specified in the primary source [S1], but the variant described in [S2] uses a length of 128.
*   **MLP:** The standard NeRF MLP structure defined in [MLPRender_Fea](2414-mlprender-fea.md), which takes position, view direction, and features as input.

**Steps:**
1.  **Initialization:** Allocate a unique embedding vector $e_i$ for each training image $i$.
2.  **Training:** During training, for a ray sampled from image $i$, retrieve the corresponding embedding $e_i$.
3.  **Injection:** Inject $e_i$ into the NeRF MLP during the color prediction phase. The specific mechanism of injection (e.g., concatenation, addition, or modulation) is not detailed in [S1], but [S2] describes an affine modulation of the bottleneck vector.
4.  **Color Prediction:** The MLP predicts the RGB color based on the ray origin, direction, and the injected embedding.
5.  **Test-Time Evaluation:** To produce a reconstruction corresponding to the photometric average of all input cameras, set the embedding vector to zero ($e_i = \mathbf{0}$) for all images [S1].

**Formulas:**
*   **Test-Time Rendering:** $C(x, d) = \text{NeRF}(x, d, e_i = \mathbf{0})$ [S1].
*   **Affine Modulation (Variant):** If using the affine modulation variant [S2], let $z$ be the bottleneck vector of the MLP. Let $MLP_{mod}(e_i)$ output two vectors: scale $s$ and shift $t$. The modulated bottleneck is $z' = s \odot z + t$, where $\odot$ is element-wise multiplication. The scale $s$ is computed as $\exp(MLP_{mod\_scale}(e_i))$ [S2].

**Parameters:**
*   **Embedding Length:** Not given in [S1]. 128 in [S2].
*   **MLP Hidden Units (for modulation):** 128 [S2].

**Edge Cases and Pitfalls:**
*   **Evaluation Metrics:** Standard metrics like PSNR, SSIM, and LPIPS are difficult to interpret for this model on datasets with significant photometric variation (e.g., Tanks and Temples) because the model neutralizes these variations [S1]. The "w/GLO" variant may perform poorly on these metrics compared to models that preserve photometric variation, even if the visual result is more consistent [S1].
*   **Test-Time Embedding:** Setting the embedding to zero at test time is crucial for obtaining a consistent, average-brightness reconstruction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming NeRF_MLP is defined as per [MLPRender_Fea](2414-mlprender-fea.md)
# and includes a bottleneck layer before the final color prediction.

def render_with_per_image_embedding(ray_origin, ray_direction, image_index, embeddings, use_affine_modulation=False):
    """
    Renders a ray using the NeRF model with per-image embeddings.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        image_index: int, index of the training image this ray belongs to.
        embeddings: list of vectors, one per training image.
        use_affine_modulation: bool, whether to use the affine modulation variant from [S2].
        
    Returns:
        RGB color value.
    """
    # Get the embedding for the current image
    e_i = embeddings[image_index]
    
    # If using affine modulation variant [S2]
    if use_affine_modulation:
        # MLP for modulation: 2-layer MLP with 128 hidden units, ReLU activation
        # Outputs scale and shift vectors of same length as bottleneck
        scale, shift = modulation_mlp(e_i)
        
        # Get the bottleneck vector from the main NeRF MLP
        # Note: The exact internal structure of NeRF_MLP is assumed to have a bottleneck
        bottleneck = get_bottleneck(ray_origin, ray_direction, e_i)
        
        # Apply affine modulation
        # Scale is exp of the scale output
        scale = exp(scale)
        modulated_bottleneck = scale * bottleneck + shift
        
        # Predict color using the modulated bottleneck
        # The rest of the MLP processes this modulated vector
        color = predict_color_from_bottleneck(modulated_bottleneck)
    else:
        # Standard injection (mechanism not specified in [S1], assumed to be concatenation or addition)
        # This is a placeholder for the unspecified injection method
        color = NeRF_MLP(ray_origin, ray_direction, e_i)
        
    return color

def modulation_mlp(e_i):
    """
    Two-layer MLP for affine modulation [S2].
    Input: e_i (128-dim vector)
    Output: scale (128-dim), shift (128-dim)
    """
    # Layer 1: Linear(128, 128), ReLU
    h = relu(linear(e_i, 128, 128))
    # Layer 2: Linear(128, 256) -> split into scale (128) and shift (128)
    out = linear(h, 128, 256)
    scale = out[:, :128]
    shift = out[:, 128:]
    return scale, shift

def get_bottleneck(ray_origin, ray_direction, e_i):
    """
    Placeholder for the bottleneck extraction from the main NeRF MLP.
    The exact implementation depends on the specific NeRF architecture.
    """
    # This function is not defined in the sources. It is assumed to be part of the NeRF_MLP.
    # [G] In a typical NeRF, this would be the output of the hidden layer before the final color prediction.
    pass

def predict_color_from_bottleneck(bottleneck):
    """
    Placeholder for the final color prediction layer.
    """
    # [G] Typically a linear layer mapping the bottleneck to RGB.
    pass
```

## Subtopics

### Affine Generative Latent Optimization
A variant that allocates a 128-length vector for each image and uses these vectors as input to a two-layer MLP with 128 hidden units [S2]. The MLP outputs two vectors (scale and shift) of the same length as the bottleneck vector [S2]. The internal activation is ReLU, and the final activation for the scaling output is exp [S2]. The scale and shift are applied to the bottleneck vector before evaluating the view-dependent MLP [S2]. This technique is only used for video results and does not improve quantitative performance on metrics [S2].

## Operating characteristics
*   **Performance:** On the Tanks and Temples dataset (color-corrected metrics), the model achieves the following PSNR scores: bicycle (24.37), flowers (21.73), Outdoor garden (26.98), stump (26.40), treehill (22.87), room (31.63), counter (29.55), Indoor kitchen (32.23), bonsai (33.46) [S1]. The variant with GLO (w/GLO) achieves: bicycle (23.95), flowers (21.60), Outdoor garden (25.09), stump (25.98), treehill (21.99), room (28.24), counter (28.40), Indoor kitchen (30.81), bonsai (30.27) [S1].
*   **Strengths:** Neutralizes photometric variation during training, yielding consistent global brightness and color across views [S1]. Produces pleasing reconstructions that correspond to the photometric average of all input cameras when the embedding is set to zero at test time [S1].
*   **Limitations:** Performs poorly on standard metrics (PSNR, SSIM, LPIPS) on datasets with significant photometric variation because it does not preserve the variation that these metrics might otherwise capture [S1]. The affine modulation variant does not improve quantitative performance [S2].
*   **When to use:** When consistent global appearance across views is desired, and photometric variation in the input data is considered noise or undesirable [S1].
*   **When not to use:** When preserving photometric variation (e.g., lighting changes) is important, or when standard metrics are the primary evaluation criterion on datasets with such variation [S1].

## Background
Originated in the Mip-NeRF 360 paper [S1]. The method was motivated by the difficulty of drawing meaningful conclusions from standard metrics on datasets with photometric variation, such as the Tanks and Temples dataset [S1]. The authors constructed their own dataset with careful control over photometric variation to better evaluate their method [S1]. The affine modulation variant was introduced in Zip-NeRF [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
