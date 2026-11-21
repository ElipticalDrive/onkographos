# FreGS

Also called: Frequency-aware regularization for 3DGS, Progressive frequency regularization

## Overview
FreGS is a training optimization method for 3D Gaussian Splatting that prevents overfitting and high-frequency artifacts by applying progressive frequency regularization to the primitives [S1, unofficial][S2]. It addresses the problem of over-reconstruction, where the model captures noise or fine details that do not generalize, by constraining the spectral content of the rendered images to match the ground truth in the frequency domain [S2]. Unlike sibling methods that focus on geometric shape constraints or depth supervision, FreGS operates exclusively on the frequency discrepancy between rendered and target images [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Current 3D Gaussian primitives (position, covariance, opacity, color), camera parameters, ground truth images, and a frequency regularization weight $\lambda$.
*   **Output:** Updated Gaussian parameters after one training step.

**Data Structures:**
*   **Gaussian Cloud:** Standard 3DGS structure.
*   **Frequency Domain Representation:** 2D Fourier Transform of the rendered and ground truth images.

**Method Steps:**
1.  **Render:** Render the current Gaussian cloud to obtain image $I_{render}$.
2.  **Transform:** Compute the 2D Fourier Transform of $I_{render}$ to get $F_{render}$ and of the ground truth $I_{gt}$ to get $F_{gt}$ [S2].
3.  **Compute Discrepancy:** Calculate the frequency discrepancy $\mathcal{L}_{freq}$ between $F_{render}$ and $F_{gt}$ [S2].
4.  **Regularize:** Add $\lambda \mathcal{L}_{freq}$ to the total loss [S1, unofficial][S2].
5.  **Backpropagate:** Compute gradients with respect to Gaussian parameters and update them [S1, unofficial].

**Formulas:**
The total loss is:
$$
\mathcal{L}_{total} = \mathcal{L}_{photometric} + \lambda \mathcal{L}_{freq}
$$
where $\mathcal{L}_{freq}$ measures the difference in the frequency domain [S2]. The specific formulation of $\mathcal{L}_{freq}$ is not detailed in the provided sources, but it is described as regularizing frequency discrepancies [S2].

**Parameters:**
*   $\lambda$: Weight for frequency regularization. Typical value not given in sources.

**Edge Cases and Pitfalls:**
*   **Over-smoothing:** Excessive regularization may remove valid high-frequency details [S1, unofficial].
*   **Precision:** Fourier transforms require careful handling of numerical precision [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fregs_step(gaussians, camera, gt_image, lambda_freq):
    # Render current state
    rendered_image = render(gaussians, camera)
    
    # Compute Fourier transforms
    F_rendered = fft2(rendered_image)
    F_gt = fft2(gt_image)
    
    # Compute frequency loss
    # Specific formula for L_freq not provided in sources
    L_freq = compute_frequency_discrepancy(F_rendered, F_gt)
    
    # Total loss
    L_total = L_photometric(rendered_image, gt_image) + lambda_freq * L_freq
    
    # Backpropagate and update
    gradients = compute_gradients(L_total, gaussians)
    update(gaussians, gradients)
```

## Subtopics

### Frequency Regularization
General approach to frequency domain constraints. [Frequency Regularization](6130-frequency-regularization.md)

### Smoothed 2D Supervision
Blurs target supervision for gradient stability. [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)

### 3D Half-Gaussian Splatting Covariance Transformation and Regularization
Enforces surface-like properties via covariance. [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)

### Hard and Soft Depth Regularization
Uses depth estimates for geometric constraints. [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)

### Cross-scale geometric adaptation
Adapts geometry across scales. [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)

### Geometry Regularization
General geometric constraints. [Geometry Regularization](14222-geometry-regularization.md)

## Operating characteristics
*   **Time Complexity:** Additional cost of Fourier transforms per step. Not given in sources.
*   **Memory Complexity:** Storage for frequency domain representations. Not given in sources.
*   **Performance:** Prevents overfitting and high-frequency artifacts [S1, unofficial]. Improves visual quality by addressing over-reconstruction [S2].
*   **Strengths:** Effective at reducing noise and artifacts [S1, unofficial].
*   **Limitations:** May remove valid details if $\lambda$ is too high. Specific performance metrics not given in sources.
*   **When to use:** When over-reconstruction or high-frequency artifacts are observed [S2].
*   **When not to use:** When fine detail recovery is critical and not smoothed out [S1, unofficial].

## Background
FreGS was introduced to address over-reconstruction in 3D Gaussian Splatting by regularizing frequency discrepancies [S2]. It applies progressive frequency regularization to primitives to prevent overfitting and high-frequency artifacts during training [S1, unofficial]. The method is cited in surveys on 3DGS rasterization and compression [S1, unofficial][S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
