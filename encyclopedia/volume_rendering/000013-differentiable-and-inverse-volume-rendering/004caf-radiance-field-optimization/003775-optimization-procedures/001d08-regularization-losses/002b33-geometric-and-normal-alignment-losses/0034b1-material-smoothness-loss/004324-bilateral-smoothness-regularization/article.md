# Bilateral Smoothness Regularization

Also called: Image gradient-aware smoothness loss, Phys3DGS Smoothness Regularizer

## Overview
Bilateral Smoothness Regularization is a geometric and normal alignment loss applied to material fields to enforce spatial smoothness of BRDF parameters (such as roughness and specular tint) weighted by image gradients [S1, unofficial][S2]. It alleviates the ambiguity between material and illumination by constraining material gradients to correspond with image edges, preventing noisy, inconsistent material reconstructions [S2]. It differs from standard smoothness losses by using the image gradient as a bilateral weight, preserving material boundaries where the image shows sharp transitions [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $x$: The BRDF parameter field (e.g., base color, roughness $\rho$, or specular tint $\alpha$) [S1, unofficial][S2].
    *   $I$: The ground truth RGB image [S1, unofficial][S2].
    *   $M$: A binary mask indicating pixels belonging to the object [S1, unofficial].
    *   $S_I$: The set of pixels on the object, used for normalization [S2].
*   **Outputs:**
    *   $L_{smooth}$: The scalar regularization loss value [S1, unofficial][S2].

**Data Structures:**
*   **Image Gradient:** Pre-computed gradient magnitude $\|\nabla_p I(p)\|$ for each pixel $p$ [S2]. The specific derivative operator is not explicitly defined in the sources, but standard finite differences are implied by the notation $\nabla_d$ [S1, unofficial].
*   **Material Field:** A tensor or array storing the scalar BRDF parameter $x$ for each surface point or pixel [S1, unofficial][S2].

**Steps:**
1.  **Pre-compute Image Gradient:** Calculate the gradient magnitude $\|\nabla_p I(p)\|$ for all pixels $p$ in the ground truth image $I$ before training [S2].
2.  **Compute Material Gradient:** For each BRDF parameter $x$ and each pixel $i$ in the mask $M$, compute the second-order derivative (Laplacian-like) in the x and y dimensions, denoted as $\nabla^2_d x(i)$ [S1, unofficial]. Note: Source [S2] uses first-order gradients $\|\nabla_{x_s} \alpha(x_s)\|$ for surface points $x_s$, while Source [S1, unofficial] uses second-order derivatives $\nabla^2_d$ for pixel indices $i$. The implementation must align with the specific source's definition of the gradient operator.
3.  **Calculate Bilateral Weight:** Compute the exponential weight $e^{\nabla_d I(i)}$ (Source [S1, unofficial]) or $e^{-\|\nabla_p I(p)\|}$ (Source [S2]). Source [S1, unofficial] uses the gradient itself in the exponent, while Source [S2] uses the negative magnitude. This discrepancy must be resolved based on the specific implementation variant.
4.  **Aggregate Loss:** Sum the product of the material gradient magnitude and the bilateral weight over all pixels in the mask [S1, unofficial][S2].
5.  **Normalize:** Divide by the number of pixels in $S_I$ if using the formulation from Source [S2] [S2].

**Formula:**
Source [S1, unofficial] defines the loss as:
$$ L_{smooth}(x) = \sum _{i\in M} \sum _{d\in x,y} |\nabla ^2_d x(i)| e^{\nabla _d I(i)} $$
where $d$ iterates over x and y dimensions [S1, unofficial].

Source [S2] defines the loss as:
$$ l_s = \frac{1}{|S_I|} \sum _{p\in S_I} (\|\nabla_{x_s} \alpha(x_s)\| + \|\nabla_{x_s} \rho(x_s)\|) e^{-\|\nabla_p I(p)\|} $$
where $x_s$ is the surface point corresponding to pixel $p$, and $\alpha, \rho$ are specular tint and roughness [S2].

**Parameters:**
*   **Mask $M$ / Set $S_I$:** Defines the region of interest (the object) [S1, unofficial][S2].
*   **Image Gradient:** Pre-computed, not learned [S2].

**Pitfalls:**
*   **Noise:** Without this regularization, the model generates noisy materials that are inconsistent across the same material, making it unsuitable for material editing applications [S2].
*   **Formula Discrepancy:** Sources [S1, unofficial] and [S2] use different gradient orders (second vs. first) and different signs in the exponential weight. Implementers must choose one variant or clarify the intended behavior [S1, unofficial][S2].
*   **Boundary Conditions:** The handling of image gradient computation at image boundaries is not specified [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilateral_smoothness_loss(material_field, ground_truth_image, mask):
    """
    Computes the Bilateral Smoothness Regularization loss.
    Implements the variant from Source [S2] for surface points, 
    as it explicitly handles normalization and distinct material parameters.
    """
    # 1. Pre-compute image gradient magnitude (assumed pre-computed in training loop)
    # Using Sobel operators for gradient magnitude
    grad_x = sobel_filter(ground_truth_image, axis=1)
    grad_y = sobel_filter(ground_truth_image, axis=0)
    image_grad_magnitude = np.sqrt(grad_x**2 + grad_y**2)
    
    # 2. Get material gradients for roughness and specular tint
    # Assuming material_field contains [roughness, specular_tint] channels
    roughness = material_field[..., 0]
    specular_tint = material_field[..., 1]
    
    # Compute first-order gradients for material parameters
    rough_grad_x = sobel_filter(roughness, axis=1)
    rough_grad_y = sobel_filter(roughness, axis=0)
    rough_grad_mag = np.sqrt(rough_grad_x**2 + rough_grad_y**2)
    
    spec_grad_x = sobel_filter(specular_tint, axis=1)
    spec_grad_y = sobel_filter(specular_tint, axis=0)
    spec_grad_mag = np.sqrt(spec_grad_x**2 + spec_grad_y**2)
    
    # 3. Calculate bilateral weight
    # Source [S2] uses e^(-||grad I||)
    weight = np.exp(-image_grad_magnitude)
    
    # 4. Aggregate loss
    # Apply mask
    masked_rough_grad = rough_grad_mag * mask
    masked_spec_grad = spec_grad_mag * mask
    masked_weight = weight * mask
    
    # Sum over pixels in S_I (mask)
    sum_grad = np.sum(masked_rough_grad + masked_spec_grad)
    sum_weight = np.sum(masked_weight)
    
    # 5. Normalize by |S_I|
    # Note: Source [S2] divides by |S_I|, which is the count of pixels in the mask
    num_pixels = np.sum(mask)
    if num_pixels == 0:
        return 0.0
        
    loss = sum_grad / num_pixels
    
    return loss
```

## Subtopics

### Material Smoothness Loss
The parent category. Enforces spatial smoothness of material fields. [Material Smoothness Loss](7299-material-smoothness-loss.md)

### Geometric and Normal Alignment Losses
The sibling category. Focuses on aligning geometry and normals. [Geometric and Normal Alignment Losses](7300-geometric-and-normal-alignment-losses.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the mask $|S_I|$ or $M$, as it involves a single pass over the pixels to compute gradients and sum the loss [S1, unofficial][S2].
*   **Memory Complexity:** Requires storing the pre-computed image gradient and the material field gradients [S2].
*   **Strengths:** Effectively reduces noise in reconstructed materials and resolves ambiguity between material and illumination [S2]. Improves suitability for material editing applications [S2].
*   **Limitations:** Requires pre-computation of image gradients [S2]. Performance may vary depending on the choice of gradient operator and exponential weight sign [S1, unofficial][S2].
*   **When to use:** When reconstructing physically-based materials (BRDF) from images, especially when noise or material-illumination ambiguity is observed [S2].
*   **When not to use:** When computational resources for pre-computing gradients are unavailable, or when the material is expected to be highly noisy by nature [S2].

## Background
*   **Origin:** The term "Bilateral Smoothness Regularization" is attributed to Yao et al. 2022 in Source [S2]. Source [S1, unofficial] refers to it as "image gradient-aware smoothness loss" and applies it in the context of Phys3DGS.
*   **History:** It was introduced to address the ambiguity between material and illumination in inverse rendering tasks [S2].
*   **Usage:** Used in Phys3DGS for BRDF parameter smoothing [S1, unofficial] and in NeIF (Neural Illumination Fields) for roughness and specular tint smoothing [S2].

## Sources
[S1] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S2] A Pre-convolved Representation for Plug-and-Play Neural Illumination Fields, https://doi.org/10.1609/aaai.v38i7.28618, fetched 2026-10-29
