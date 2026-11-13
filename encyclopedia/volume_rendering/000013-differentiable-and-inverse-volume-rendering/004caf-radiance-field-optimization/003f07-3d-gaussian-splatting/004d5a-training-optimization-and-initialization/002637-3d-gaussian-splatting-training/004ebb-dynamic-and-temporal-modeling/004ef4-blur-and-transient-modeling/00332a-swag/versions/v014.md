# SWAG
Also called: SWAG: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG extends 3D Gaussian Splatting (3DGS) by conditioning Gaussians on learned appearance embeddings to handle dynamic scenes [S1][S2, unofficial]. It specifically addresses the problem of transient occluders (e.g., pedestrians, moving vehicles) by training unsupervised transient Gaussians that learn to ignore these distractors, yielding state-of-the-art results on diverse photo scenes [S1][S2, unofficial]. It is distinguished from siblings like Neural Transient Field (NeTF) by its reliance on explicit Gaussian primitives with appearance conditioning rather than implicit neural fields for transient modeling [S1][S2, unofficial].

## How it works
SWAG modifies the standard 3DGS representation to separate static geometry from transient appearance variations.

**Inputs and Outputs:**
*   **Input:** A set of uncalibrated or calibrated images $\{I_i\}$ of a scene with transient objects.
*   **Output:** A set of 3D Gaussians $\mathcal{G}$, each defined by position $\mathbf{c}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and a base color $\mathbf{c}_0$. Additionally, each Gaussian is associated with a learned appearance embedding $\mathbf{e}$ and a transient weight $w_t$.

**Data Structures:**
*   **Gaussian Attributes:** Standard 3DGS attributes ($\mathbf{c}, \alpha, \Sigma, \mathbf{c}_0$).
*   **Appearance Embedding:** A learnable vector $\mathbf{e}_i$ per Gaussian $i$, used to modulate color based on view or time context.
*   **Transient Mask/Weight:** A scalar or binary indicator per Gaussian determining if it contributes to the final color or is suppressed as a transient occluder.

**Steps:**
1.  **Initialization:** Initialize Gaussians from structure-from-motion (SfM) points as in standard 3DGS [S1][S2, unofficial].
2.  **Appearance Conditioning:** Associate each Gaussian with a learned appearance embedding $\mathbf{e}$. The final color $\mathbf{c}$ of a Gaussian is computed by modulating the base color $\mathbf{c}_0$ using $\mathbf{e}$ and potentially view-dependent parameters [S1][S2, unofficial].
3.  **Transient Gaussian Training:** During optimization, Gaussians are trained to distinguish between static scene components and transient occluders. This is achieved by minimizing a photometric loss while encouraging Gaussians corresponding to transient objects to have low opacity or high transient weights, effectively ignoring them in the reconstruction [S1][S2, unofficial].
4.  **Rendering:** Render the scene using standard alpha-blending rasterization. Gaussians identified as transient are suppressed, resulting in a clean reconstruction of the static scene [S1][S2, unofficial].

**Formulas:**
The color contribution of a Gaussian $i$ is modulated by its appearance embedding $\mathbf{e}_i$:
$$ \mathbf{c}_i = \text{Modulate}(\mathbf{c}_{0,i}, \mathbf{e}_i) $$
The final pixel color is computed via standard volume rendering, with transient Gaussians effectively having $\alpha_i \approx 0$ or being masked out [S1][S2, unofficial].

**Parameters:**
*   Learning rate for embeddings: Not given in the sources.
*   Transient suppression threshold: Not given in the sources.

**Edge Cases:**
*   **Precision:** Standard floating-point precision used in 3DGS.
*   **Boundaries:** Transient objects at scene boundaries may be partially reconstructed if not fully occluded.
*   **Degenerate Input:** Heavy occlusion may lead to incomplete static geometry if transient Gaussians are incorrectly classified as static.

## Subtopics

### Neural Transient Field (NeTF)
Uses implicit neural fields to model transients, contrasting with SWAG's explicit Gaussian approach. Choose NeTF for complex, non-rigid transient dynamics where explicit primitives are insufficient. [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)

## Operating characteristics
*   **Time Complexity:** Comparable to 3DGS training, with overhead for appearance embedding optimization.
*   **Memory Complexity:** Increased memory usage due to stored appearance embeddings per Gaussian.
*   **Performance:** State-of-the-art results on diverse photo scenes with transient objects [S1][S2, unofficial].
*   **Strengths:** Robust to transient occluders; high-quality static scene reconstruction.
*   **Limitations:** Requires careful tuning to separate static and transient components; may struggle with highly dynamic scenes where static geometry is ambiguous.
*   **When to use:** Scenes with frequent transient occluders (e.g., urban environments with pedestrians/vehicles).
*   **When not to use:** Static scenes without transients (standard 3DGS is sufficient); highly dynamic scenes with significant non-rigid motion.

## Background
SWAG was introduced as an extension to 3D Gaussian Splatting to address the challenge of transient occluders in novel view synthesis [S1][S2, unofficial]. It builds on the success of 3DGS by incorporating appearance embeddings and unsupervised transient modeling, achieving state-of-the-art results on diverse photo scenes [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
