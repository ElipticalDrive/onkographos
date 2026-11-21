# Template-Based Correspondence Module


## Overview
The Template-Based Correspondence Module (TCM) is a method for non-photorealistic radiance field optimization that broadcasts style from a reference view to novel views by enforcing semantic correspondence consistency [S1]. It addresses the limitation of Reference Ray Registration (R3), which fails to register reference rays to occluded regions in 360° scenes, by using a content-style mapping template to guide stylization across the entire scene [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $\omega_P$: Photorealistic radiance field.
    *   $\omega_{NP}$: Non-photorealistic radiance field (to be optimized).
    *   $I_R$: Content reference image.
    *   $S_R$: Style reference image.
    *   $\phi$: Camera pose for a content domain view.
    *   $\phi_R$: Reference camera pose.
*   **Outputs:**
    *   Optimized $\omega_{NP}$.
    *   Stylized novel views rendered from $\omega_{NP}$.

**Data Structures:**
*   **Feature Maps:** High-level semantic feature maps $F_I$, $F_{IR}$, $F_{SR}$, $F_G$, and $\hat{F}_S$. These are 2D arrays of feature vectors. The specific extractor is VGG16 [S1]. Features used for loss calculation are concatenated from stages 3 and 4 (relu 3*, relu 4*) [S1].
*   **Color Patches:** Average color $\bar{C}$ for patches at feature-level indices.

**Parameters:**
*   $\lambda_0$: Balancing factor for feature loss, typically $5 \times 10^{-3}$ [S1].
*   $\lambda_c$: Balancing factor for color loss, typically $5$ [S1].
*   $\lambda_r, \lambda_f$: Balancing factors for reference and feature terms in the total objective, typically $1$ [S1].
*   $dist(a, b)$: Distance metric between feature vectors, implemented as cosine distance [S1].

**Steps:**
1.  **Extract Reference Features:** Compute semantic feature maps $F_{IR}$ from content reference $I_R$ and $F_{SR}$ from style reference $S_R$ using a pre-trained VGG16 extractor [S1].
2.  **Render and Extract Content Features:** For a content domain view $I$ rendered from $\omega_P$ under camera pose $\phi$, compute its semantic feature map $F_I$ using the same VGG16 extractor [S1].
3.  **Construct Guidance Feature ($F_G$):** Perform a search-and-replace process to map the reference style to the current content view. For each 2D position $(i, j)$ in $F_I$:
    *   Find the index $(i^*, j^*)$ in $F_I$ that minimizes the distance to the reference feature $F_{IR}$ at $(i, j)$:
        $$ (i^*, j^*) = \arg \min_{i_0, j_0} dist(F_I(i_0, j_0), F_{IR}(i, j)) $$
    *   Set the guidance feature at $(i, j)$ to the style reference feature at the matched index:
        $$ F_G(i, j) = F_{SR}(i^*, j^*) $$
    [S1]
4.  **Render Stylized View:** Render a stylized view $S$ from the non-photorealistic radiance field $\omega_{NP}$ under the same camera pose $\phi$.
5.  **Extract Stylized Features:** Compute the semantic feature map $\hat{F}_S$ for the stylized view $S$ using the VGG16 extractor [S1].
6.  **Compute Feature Loss ($L_{feat}$):** Enforce implicit feature-level supervision by forcing $\hat{F}_S$ to imitate $F_G$ and match the content structure $F_I$:
    $$ L_{feat} = \frac{1}{N} \sum_{i,j} dist(F_G(i, j), \hat{F}_S(i, j)) + \lambda_0 \| F_I(i, j) - \hat{F}_S(i, j) \|_2^2 $$
    where $N$ is the number of pixels/elements [S1].
7.  **Compute Color Loss ($L_{color}$):** Address color mismatches that cosine distance optimization cannot eliminate by transferring average color in patches. For each position $(i, j)$, let $\bar{C}_{NP}$ be the average color of the patch in the stylized view at $(i, j)$, and $\bar{C}_R(i^*, j^*)$ be the average color of the reference patch matched by the feature distance minimization in Step 3:
    $$ L_{color} = \frac{1}{N} \sum_{i,j} \| \bar{C}_{NP}(i, j) - \bar{C}_R(i^*, j^*) \|_2^2 $$
    [S1]
8.  **Optimize:** Minimize the total objective $L_{NP} = \lambda_f L_{feat} + \lambda_r L_{ref} + \lambda_c L_{color}$ to update $\omega_{NP}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def template_based_correspondence_module(
    omega_P, omega_NP, I_R, S_R, phi, phi_R, 
    lambda_0=5e-3, lambda_c=5, lambda_f=1, lambda_r=1
):
    # 1. Extract Reference Features
    F_IR = vgg16_extract(I_R)
    F_SR = vgg16_extract(S_R)
    
    # 2. Render and Extract Content Features
    I = render(omega_P, phi)
    F_I = vgg16_extract(I)
    
    # 3. Construct Guidance Feature F_G
    H, W, C = F_I.shape
    F_G = zeros_like(F_I)
    # Precompute distances to find matches for all (i,j)
    # dist is cosine distance
    for i in range(H):
        for j in range(W):
            target = F_IR[i, j]
            # Find (i*, j*) minimizing distance to F_IR[i,j] within F_I
            min_dist = infinity
            best_idx = (0, 0)
            for i0 in range(H):
                for j0 in range(W):
                    d = cosine_distance(F_I[i0, j0], target)
                    if d < min_dist:
                        min_dist = d
                        best_idx = (i0, j0)
            i_star, j_star = best_idx
            F_G[i, j] = F_SR[i_star, j_star]
            
    # 4. Render Stylized View
    S = render(omega_NP, phi)
    
    # 5. Extract Stylized Features
    F_S_hat = vgg16_extract(S)
    
    # 6. Compute Feature Loss
    # dist(F_G, F_S_hat) is cosine distance
    # ||F_I - F_S_hat||_2^2 is MSE
    feat_dist = cosine_distance(F_G, F_S_hat)
    content_dist = squared_L2_norm(F_I - F_S_hat)
    L_feat = mean(feat_dist) + lambda_0 * mean(content_dist)
    
    # 7. Compute Color Loss
    # Get average color for patches at (i,j) and matched (i*,j*)
    # Note: In implementation, patches are defined by the feature grid resolution
    C_NP = get_patch_average_color(S, F_I.shape)
    C_R = get_patch_average_color_from_reference(I_R, S_R, F_I, F_IR, F_SR)
    # C_R[i,j] corresponds to the color at the matched index (i*, j*)
    L_color = mean(squared_L2_norm(C_NP - C_R))
    
    # 8. Total Loss and Optimization
    # L_ref is assumed to be computed separately by R3 module
    L_ref = compute_reference_loss(omega_NP, phi_R) # Placeholder for R3 specific loss
    L_total = lambda_f * L_feat + lambda_r * L_ref + lambda_c * L_color
    
    # Update omega_NP using gradient descent on L_total
    # Note: Optimize patch-wise following gradient cache strategy to save memory [S1]
    return omega_NP, L_total
```

**Edge Cases and Pitfalls:**
*   **Memory Limitation:** Semantic features are extracted at the image level, which can cause memory overflow during back-propagation. The optimization of $\omega_{NP}$ should be performed patch-wise following the gradient cache strategy [S1].
*   **Color Mismatch:** Cosine distance optimization alone cannot eliminate color mismatches. The coarse color-matching loss $L_{color}$ is required to address this [S1].
*   **Occlusion:** TCM is specifically designed to handle occluded regions in 360° scenes where R3 fails, by leveraging semantic consistency across the entire scene [S1].

## Operating characteristics
*   **Time Complexity:** The search-and-replace process involves finding the nearest neighbor in the feature map for each pixel, which is $O(H \cdot W \cdot H \cdot W)$ if naive, or $O(H \cdot W \log(H \cdot W))$ with optimized spatial indexing. The feature extraction and loss computation are linear in the number of pixels $N$ [S1].
*   **Memory Complexity:** High, due to storing feature maps for multiple views and the need for gradient caching during patch-wise optimization [S1].
*   **Strengths:** Overcomes occlusion issues in 360° scenes by broadcasting style based on semantic correspondence rather than just geometric ray registration [S1].
*   **Limitations:** Relies on the assumption that semantic correspondence is consistent before and after stylization, which may not hold for all scenes or styles [S1]. Computationally expensive due to feature matching and VGG16 extraction [S1].
*   **When to use:** When stylizing 360° scenes with significant occlusions where reference-based ray registration fails to cover all regions [S1].
*   **When not to use:** For scenes where semantic consistency is not a valid assumption or when computational resources are severely limited [S1].

## Background
The Template-Based Correspondence Module was introduced in the Ref-NPR paper [S1]. It builds on the Reference Ray Registration (R3) method, addressing its inability to handle occluded regions in 360° scenes [S1]. The method leverages the common assumption in scene stylization that semantic correspondence remains consistent across stylization [S1]. It uses VGG16 as the semantic feature extractor, following practices in prior work [S1]. The implementation is based on the ARF codebase and uses Plenoxels for scene representation [S1].

## Sources
[S1] Ref-NPR: Reference-Based Non-Photorealistic Radiance Fields for Controllable Scene Stylization, https://doi.org/10.1109/cvpr52729.2023.00413, fetched 2026-10-26
