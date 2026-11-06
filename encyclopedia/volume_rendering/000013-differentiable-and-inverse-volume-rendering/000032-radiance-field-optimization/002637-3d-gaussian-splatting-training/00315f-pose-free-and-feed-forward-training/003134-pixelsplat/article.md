# pixelSplat

Also called: 3D Gaussian Splats from Image Pairs, Epipolar Transformer Gaussian Splatting

## Overview
pixelSplat is a feed-forward model that reconstructs a 3D radiance field parameterized by 3D Gaussian primitives from a pair of input images [S1]. It solves the problem of generalizable novel view synthesis by enabling real-time, memory-efficient rendering and fast 3D reconstruction at inference time [S1]. Unlike light-field transformers which produce black-box ray fields, pixelSplat generates an explicit, editable 3D representation [S1]. It is distinguished from its siblings by its specific use of an epipolar transformer to resolve scale ambiguity and a probabilistic sampling mechanism to avoid local minima in primitive regression [S1].

## How it works
**Inputs:** A pair of images $I_1, I_2$ and their associated camera parameters (poses and intrinsics) [S1].
**Outputs:** A set of 3D Gaussian primitives $\{G_i\}$, where each primitive is defined by mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and scale $S_i$ [S1]. The spherical harmonics coefficients $c_{i,lm}$ are not explicitly detailed in the source for the prediction head, but the parent article notes they are part of the general representation [6136-instantsplat.md][6464-pixelsplat.md].

**Data Structures:**
The model predicts parameters for pixel-aligned Gaussian primitives. The feature map $\mathbf{F}[\mathbf{u}]$ is generated for each pixel $\mathbf{u}$ [S1]. The prediction network $f$ outputs:
*   $\phi$: Probability distribution parameters (implied by the probabilistic prediction scheme) [S1].
*   $\boldsymbol{\delta}$: Depth offset or relative depth parameter [S1].
*   $\boldsymbol{\Sigma}$: Covariance matrix parameters [S1].
*   $\mathbf{S}$: Scale factor [S1].

**Steps:**
1.  **Image Encoding:** Each reference image is passed through a ResNet-50 and a ViT-B/8 vision transformer, both pre-trained using a DINO objective. Their pixel-wise outputs are summed to generate initial features [S1].
2.  **Epolar Feature Encoding:** The features from the two views are processed by a multi-view epipolar transformer. This involves two rounds of epipolar cross-attention. Frequency-based positional encoding of depth is used to resolve scale ambiguity [S1].
3.  **Gaussian Parameter Prediction:** For each pixel $\mathbf{u}$, a neural network $f$ conditioned on the feature $\mathbf{F}[\mathbf{u}]$ predicts the Gaussian parameters [S1]:
    $$ \phi, \boldsymbol{\delta}, \boldsymbol{\Sigma}, \mathbf{S} = f(\mathbf{F}[\mathbf{u}]) $$
    [S1]
4.  **Probabilistic Sampling:** A dense probability distribution over 3D space is predicted. Gaussian means are sampled from this distribution. The sampling operation is made differentiable via a reparameterization trick [S1].
5.  **Reparameterization Trick:** The density of a sampled Gaussian primitive is coupled to the probability of that location. This allows back-propagation of gradients through the Gaussian splatting representation. When a gradient would increase the opacity of a Gaussian at a 3D location, the model increases the probability that a Gaussian will be sampled at that location again in the future, implicitly spawning or deleting primitives during training [S1].
6.  **Rendering:** The predicted 3D Gaussians are rendered using differentiable volume rendering (standard Gaussian splatting rasterization) to produce novel views [S1][6136-instantsplat.md].

**Parameters:**
*   Training resolution: $256 \times 256$ pixels [S1].
*   Loss function: Combination of Mean Squared Error (MSE) and Perceptual Distance (LPIPS) [S1].
*   Optimizer: Adam [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixelsplat_forward(images, camera_params):
    # images: list of 2 images [H, W, 3]
    # camera_params: list of 2 camera parameter sets
    
    # 1. Image Encoding
    # ResNet-50 and ViT-B/8 features summed
    features = []
    for img in images:
        resnet_feat = resnet50_dino(img)
        vit_feat = vit_b8_dino(img)
        features.append(resnet_feat + vit_feat)
    
    # 2. Epipolar Transformer
    # Two rounds of epipolar cross-attention with frequency-based depth encoding
    fused_features = epipolar_transformer_cross_attention(
        features[0], features[1], 
        camera_params[0], camera_params[1]
    )
    
    # 3. Gaussian Parameter Prediction
    # Predict phi, delta, Sigma, S for each pixel
    # phi represents parameters for the probability distribution over depth
    phi, delta, sigma, scale = gaussian_prediction_head(fused_features)
    
    # 4. Probabilistic Sampling (Differentiable)
    # Sample depths from the predicted distribution
    # The reparameterization trick ensures gradients flow to phi
    sampled_depths = sample_from_distribution(phi, delta)
    
    # 5. Construct Gaussians
    # Convert sampled depths and features to 3D Gaussians
    # Mean mu is derived from camera ray and sampled depth
    # Covariance Sigma and Scale S are used to define the Gaussian shape
    gaussians = []
    for u in pixels:
        mu = ray_origin[u] + sampled_depths[u] * ray_direction[u]
        # Sigma and S are predicted directly or derived
        # Alpha is typically predicted or derived from opacity
        g = Gaussian(
            mean=mu,
            covariance=sigma[u],
            opacity=predict_opacity(u), # Assumed part of prediction or derived
            scale=scale[u]
        )
        gaussians.append(g)
        
    return gaussians

def train_step(images, gt_images, camera_params):
    gaussians = pixelsplat_forward(images, camera_params)
    rendered_images = render_gaussians(gaussians, camera_params)
    
    # Loss: MSE + LPIPS
    loss = mse_loss(rendered_images, gt_images) + lpips_loss(rendered_images, gt_images)
    
    # Backpropagate through the differentiable sampling
    loss.backward()
    optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** SfM poses are scaled by an arbitrary unknown scalar $s_i$. The model handles this by inferring the scale via the epipolar transformer [S1]. Without it, ghosting artifacts occur [S1].
*   **Local Minima:** Direct regression of Gaussian means suffers from local minima because Gaussians have local support and gradients vanish if the distance to the correct location is large. The probabilistic sampling avoids this [S1].
*   **Out-of-Distribution Views:** Reflective surfaces may appear transparent, and Gaussians may appear billboard-like when viewed from out-of-distribution camera poses [S1].

## Subtopics

### MVSplat
A method that builds upon PixelSplat by combining multi-view feature extraction through transformers and cost volumes, which are then processed by a U-Net [S3]. It extends the two-view approach to multi-view settings.

## Operating characteristics
*   **Time Complexity:** Inference cost is approximately 650 times less than the next-fastest baseline for encoding one scene and rendering 100 images [S1]. It accelerates rendering by 2.5 orders of magnitude compared to state-of-the-art light field transformers [S1].
*   **Memory Complexity:** Memory-efficient rendering and scalable training [S1]. Uses significantly less memory per ray at training time compared to baselines [S1].
*   **Performance:** Outperforms baselines (pixelNeRF, GPNR, Du et al.) on RealEstate10k and ACID datasets, with significant improvements in perceptual distance (LPIPS) [S1].
*   **Strengths:** Real-time rendering, editable 3D representation, fast inference, handles scale ambiguity, avoids local minima.
*   **Limitations:** Struggles with reflective surfaces (transparency) and out-of-distribution views (billboarding) [S1]. Requires known camera poses (unlike NoPoSplat) [S2, unofficial].

## Background
pixelSplat was introduced by Charatan, Li, Tagliasacchi, and Sitzmann from MIT and Simon Fraser University/University of Toronto [S1]. It was presented at CVPR 2024 [S1]. The method addresses the limitations of previous generalizable novel view synthesis approaches, specifically the high computational cost of differentiable rendering and the black-box nature of light-field transformers [S1]. It builds on the success of 3D Gaussian Splatting for single-scene optimization [19] but adapts it for feed-forward, generalizable inference [S1].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
