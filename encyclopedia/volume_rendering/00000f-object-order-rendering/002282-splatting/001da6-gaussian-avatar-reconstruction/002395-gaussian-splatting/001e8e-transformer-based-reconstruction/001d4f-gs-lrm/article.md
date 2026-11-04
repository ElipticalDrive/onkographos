# GS-LRM

Also called: Gaussian Large Reconstruction Model

## Overview
GS-LRM is a feed-forward transformer-based method that reconstructs 3D Gaussian Splatting (3DGS) representations from a small set of input views. It solves the sparse-view reconstruction problem by directly predicting Gaussian parameters without iterative optimization. It is distinguished from its siblings, such as MVSplat and NoPoSplat, by its specific tokenization strategy of concatenating images with camera parameters before processing them through a transformer block [S1, unofficial].

## How it works
**Inputs:** A set of 2 to 4 input images and their corresponding camera parameters (extrinsics and intrinsics).
**Outputs:** A set of 3D Gaussian parameters (position, covariance, color, opacity) aligned along camera-to-3D pixel rays.

**Data Structures:**
The method processes tokens representing visual and geometric information.
1.  **Image Tokens:** Features extracted from the input images.
2.  **Camera Tokens:** Embeddings representing the camera pose and intrinsics.
3.  **Concatenated Sequence:** The image and camera tokens are concatenated into a single sequence.

**Steps:**
1.  **Tokenization:** Extract features from the 2–4 input images. Generate embeddings for the associated camera parameters.
2.  **Concatenation:** Concatenate the image features and camera parameter embeddings into a unified token sequence [S1, unofficial].
3.  **Transformer Processing:** Pass the concatenated sequence through a transformer block. The transformer attends to the relationships between views and camera poses to infer 3D structure [S1, unofficial].
4.  **Decoding:** Decode the output tokens from the transformer block into the final Gaussian parameters (mean, covariance, color, opacity) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gs_lrm_forward(images, camera_params):
    """
    Reconstructs 3D Gaussians from 2-4 images and camera parameters.
    
    Args:
        images: List of 2-4 input images (H, W, 3).
        camera_params: List of 2-4 camera parameter sets (extrinsics, intrinsics).
        
    Returns:
        gaussians: List of 3D Gaussian parameters (means, covariances, colors, opacities).
    """
    # Step 1: Extract image features
    # Note: Specific feature extractor architecture (e.g., ViT encoder) is not detailed in sources.
    image_features = extract_image_features(images)
    
    # Step 2: Generate camera embeddings
    # Note: Specific embedding method for camera parameters is not detailed in sources.
    camera_embeddings = embed_camera_params(camera_params)
    
    # Step 3: Concatenate tokens
    # Concatenate image features and camera embeddings along the sequence dimension.
    combined_tokens = concatenate(image_features, camera_embeddings)
    
    # Step 4: Process through transformer block
    # The transformer block attends to the combined tokens.
    transformer_output = transformer_block(combined_tokens)
    
    # Step 5: Decode Gaussian parameters
    # Decode the output tokens into 3D Gaussian parameters.
    gaussians = decode_gaussians(transformer_output)
    
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **View Count:** The method is designed for 2 to 4 images. Performance on significantly fewer or more views is not specified [S1, unofficial].
*   **Camera Accuracy:** The method relies on known camera parameters. Unlike NoPoSplat, it does not handle unknown poses or intrinsics [S1, unofficial].

## Subtopics

### Large 4D Gaussian Model (L4GM)
L4GM is a feed-forward 4D large reconstruction model that outputs per-frame 3D Gaussian Splatting (3DGS) from a single-view video in one forward pass of approximately 1 second. It upsamples the output to higher frame rates for temporal smoothness using temporal self-attention. It is built upon LGM and trained on a large Objaverse-derived multiview-video dataset containing 44K objects, 12M videos, and 300M frames. It is marked as a 2026 SOTA update [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is feed-forward, enabling fast inference compared to iterative optimization methods. L4GM, a related variant, completes reconstruction in ~1 second [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Fast, direct reconstruction from sparse views; leverages transformer attention for multi-view consistency [S1, unofficial].
*   **Limitations:** Requires known camera parameters; limited to 2-4 input views as per the base definition [S1, unofficial].
*   **When to use:** Sparse-view reconstruction where camera poses are known and fast inference is required.
*   **When not to use:** When camera poses are unknown (use NoPoSplat) or when dense multi-view input is available (other methods may be more robust).

## Background
GS-LRM was introduced as part of the evolution of feed-forward 3D reconstruction models. It builds upon earlier methods like PixelSplat and MVSplat by combining multi-view feature extraction through transformers and cost volumes. It is cited in the context of Gaussian Splatting reconstruction tutorials and surveys [S1, unofficial]. L4GM, a 4D variant, is noted as a 2026 SOTA update, trained on a massive Objaverse-derived dataset [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
