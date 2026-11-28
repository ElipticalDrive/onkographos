# SWAG

Also called: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG extends 3D Gaussian Splatting by conditioning Gaussians on learned appearance embeddings and training unsupervised transient Gaussians to ignore occluders, yielding state-of-the-art results on diverse photo scenes [S1][S2, unofficial]. It distinguishes itself from siblings like Wild-GS by explicitly modeling transient elements as separate Gaussian entities rather than relying solely on depth regularization or visibility maps to mitigate their effects [S1][S2, unofficial].

## How it works
SWAG builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) by introducing two key mechanisms: appearance-conditioned Gaussians and unsupervised transient Gaussian training.

**Inputs and Outputs:**
- **Inputs:** A set of images with known camera poses, initial 3D Gaussian primitives (position, covariance, opacity, spherical harmonics coefficients).
- **Outputs:** Optimized Gaussian parameters including transient Gaussians that capture moving or transient objects, and appearance embeddings for static geometry.

**Data Structures:**
- **Gaussian Primitives:** Each Gaussian $i$ has:
  - Position $\mathbf{p}_i \in \mathbb{R}^3$
  - Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$)
  - Opacity $\alpha_i \in [0, 1]$
  - Spherical Harmonics coefficients $\mathbf{c}_i$ for view-dependent color
  - **Appearance Embedding** $\mathbf{e}_i \in \mathbb{R}^d$: A learned vector conditioning the appearance of static Gaussians.
  - **Transient Flag** $t_i \in \{0, 1\}$: Indicates if the Gaussian is transient (1) or static (0).

**Numbered Steps:**
1. **Initialization:** Initialize standard 3D Gaussians from point cloud or SfM data. Initialize appearance embeddings $\mathbf{e}_i$ for each Gaussian.
2. **Appearance Conditioning:** Modify the color computation of static Gaussians ($t_i=0$) to depend on their appearance embedding $\mathbf{e}_i$ and view direction. The color $\mathbf{C}_i$ is computed as a function $f(\mathbf{c}_i, \mathbf{e}_i, \mathbf{v})$, where $\mathbf{v}$ is the view direction.
3. **Transient Gaussian Training:** Introduce a subset of Gaussians designated as transient ($t_i=1$). These Gaussians are optimized to capture pixels that do not correspond to static geometry (e.g., moving people, reflections). They are trained without appearance embeddings or with a separate transient appearance model.
4. **Occlusion Handling:** During rendering, transient Gaussians are allowed to occlude static Gaussians and other transient Gaussians. The training loss encourages transient Gaussians to explain pixel colors that cannot be explained by static geometry, effectively ignoring occluders in the static scene reconstruction.
5. **Optimization:** Jointly optimize Gaussian parameters, appearance embeddings, and transient flags using a photometric loss. The loss function typically includes:
   - Photometric loss between rendered and ground truth images.
   - Regularization on transient Gaussians to prevent them from overfitting to noise.
   - Regularization on appearance embeddings to ensure smoothness.

**Formulas:**
- **Color for Static Gaussian:** $\mathbf{C}_i = \text{Decode}(\mathbf{c}_i, \mathbf{e}_i, \mathbf{v})$
- **Color for Transient Gaussian:** $\mathbf{C}_i = \text{Decode}(\mathbf{c}_i, \mathbf{v})$ (or similar, without embedding)
- **Total Loss:** $\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_t \mathcal{L}_{\text{transient}} + \lambda_e \mathcal{L}_{\text{embed}}$

**Parameters:**
- $\lambda_t$: Weight for transient regularization. Not given in the sources.
- $\lambda_e$: Weight for embedding regularization. Not given in the sources.
- $d$: Dimension of appearance embedding. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Ensure numerical stability when optimizing transient flags. [G] Use a soft threshold or sigmoid for transient flag optimization to allow gradient flow.
- **Boundaries:** Transient Gaussians may leak into static regions if not properly regularized. [G] Add a spatial regularization term to encourage transient Gaussians to be localized.
- **Degenerate Input:** If no transient objects are present, transient Gaussians should converge to low opacity or be pruned. [G] Implement a pruning step based on opacity and contribution to the loss.

## Subtopics

### Wild-GS
Aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects. Choose when you want to constrain geometry more strictly using depth. [Wild-GS](6064-wildgaussians.md)

### SpotlessSplats
Detects outliers in a richer, pre-trained feature space to ignore transient distractors. Choose when you have access to a pre-trained feature extractor and want robustness in casual captures. [SpotlessSplats](6064-spotless-splats.md) (Note: Link placeholder, adjust if actual link exists)

## Operating characteristics
- **Time Complexity:** Similar to 3DGS, but with additional overhead for optimizing appearance embeddings and transient Gaussians. Not given in the sources.
- **Memory Complexity:** Increased memory usage due to appearance embeddings and additional transient Gaussians. Not given in the sources.
- **Measured Performance:** Yields state-of-the-art results on diverse photo scenes. Not given in the sources.
- **Strengths:** Handles transient objects effectively without manual segmentation. Improves reconstruction quality in casual captures.
- **Limitations:** Requires careful tuning of regularization parameters. May be slower to converge than standard 3DGS.
- **When to Use:** When dealing with scenes containing moving objects or transient elements.
- **When Not to Use:** When scenes are static and computational resources are limited.

## Background
SWAG was introduced as an extension to 3D Gaussian Splatting to address the challenge of transient objects in novel view synthesis. It builds on the success of 3DGS by introducing unsupervised transient Gaussian training. The method is part of a broader trend in differentiable rendering to handle complex real-world scenes. [S1][S2, unofficial]

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
