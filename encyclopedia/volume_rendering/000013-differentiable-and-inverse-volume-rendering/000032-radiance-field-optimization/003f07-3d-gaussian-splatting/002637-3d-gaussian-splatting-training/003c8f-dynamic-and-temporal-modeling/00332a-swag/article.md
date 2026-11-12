# SWAG

Also called: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG extends 3D Gaussian Splatting by conditioning Gaussians on learned appearance embeddings and training unsupervised transient Gaussians to ignore occluders, yielding state-of-the-art results on diverse photo scenes [S1][S2, unofficial]. It distinguishes itself from sibling methods like Wild-GS by explicitly training transient Gaussians to ignore occluders rather than relying solely on depth regularization or visibility maps to mitigate transient effects [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of unposed or loosely posed images containing transient objects (e.g., pedestrians, vehicles).
*   **Outputs:** A set of static Gaussians representing the persistent scene geometry and appearance, and a set of transient Gaussians that capture occluders but are excluded from the final rendering.

**Data Structures:**
*   **Static Gaussians:** Standard 3DGS attributes (position $\mathbf{x}$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$).
*   **Transient Gaussians:** Same attributes as static Gaussians but marked for exclusion during rendering.
*   **Appearance Embeddings:** A learned embedding vector $\mathbf{e}$ associated with each Gaussian, used to condition appearance.

**Steps:**
1.  **Initialization:** Initialize a set of Gaussians from the input images using standard 3DGS initialization techniques.
2.  **Appearance Conditioning:** Condition the appearance of each Gaussian on its learned appearance embedding $\mathbf{e}$. This allows the Gaussian to adapt its color based on the viewing context or specific appearance features.
3.  **Transient Gaussian Training:** Train a subset of Gaussians to represent transient objects. These Gaussians are optimized to fit the pixel colors of transient objects in the input images.
4.  **Occluder Ignorance:** During rendering, transient Gaussians are explicitly ignored. This is achieved by marking them as transient and excluding them from the alpha compositing step. The static Gaussians are rendered to produce the final image, effectively removing transient occluders.
5.  **Optimization:** Optimize the positions, covariances, opacities, and appearance embeddings of both static and transient Gaussians using a loss function that includes photometric loss on the rendered images. The transient Gaussians are allowed to have high opacity to fit the transient objects, but their contribution is masked out in the final image.

**Formulas:**
*   **Rendering:** The final color $C$ is computed by compositing only the static Gaussians:
    $$ C = \sum_{i \in \text{static}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $c_i$ is the color of the $i$-th static Gaussian, $\alpha_i$ is its opacity, and the product term represents the transparency of previous Gaussians. Transient Gaussians are excluded from this sum.

**Parameters:**
*   **Learning Rate:** Standard 3DGS learning rates for positions, covariances, and opacities.
*   **Appearance Embedding Dimension:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point precision is assumed.
*   **Boundaries:** Transient objects that are partially occluded by static geometry may be difficult to separate.
*   **Degenerate Input:** If the input images do not contain transient objects, the transient Gaussians may not converge or may interfere with the static scene.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for SWAG rendering and training
def render_swag(static_gaussians, transient_gaussians, camera):
    """
    Renders the scene using only static Gaussians.
    """
    # Sort Gaussians by depth relative to camera
    sorted_static = sort_by_depth(static_gaussians, camera)
    
    color = (0.0, 0.0, 0.0)
    alpha = 0.0
    
    for gaussian in sorted_static:
        # Project Gaussian to 2D
        screen_pos, screen_cov = project_gaussian(gaussian, camera)
        
        # Compute contribution
        c, a = compute_contribution(gaussian, screen_pos, screen_cov)
        
        # Composite
        color += c * a * (1 - alpha)
        alpha += a * (1 - alpha)
        
        if alpha > 0.99:
            break
            
    return color

def train_swag(static_gaussians, transient_gaussians, images, camera_poses):
    """
    Optimizes static and transient Gaussians.
    """
    for image, camera_pose in zip(images, camera_poses):
        # Render static scene
        rendered_color = render_swag(static_gaussians, transient_gaussians, camera_pose)
        
        # Compute photometric loss
        loss = photometric_loss(rendered_color, image)
        
        # Optimize static Gaussians
        optimize_gaussians(static_gaussians, loss)
        
        # Optimize transient Gaussians to fit transient parts of the image
        # This step is implicit in the training process where transient Gaussians
        # are allowed to absorb the error from transient objects.
        optimize_transient_gaussians(transient_gaussians, image, camera_pose)
```

## Subtopics

### Wild-GS
Aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects. Choose when you need a simpler approach without explicit transient Gaussian training. [Wild-GS](link)

### SpotlessSplats
Enhances robustness by detecting outliers in a richer, pre-trained feature space to ignore transient distractors. Choose when you have access to a pre-trained feature space and want to improve reconstruction quality in casual captures. [SpotlessSplats](link)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Yields state-of-the-art results on diverse photo scenes [S1][S2, unofficial].
*   **Strengths:** Effectively removes transient occluders without requiring explicit segmentation or pose information.
*   **Limitations:** Requires training transient Gaussians, which may increase training time.
*   **When to use:** When dealing with unposed or loosely posed images containing transient objects.
*   **When not to use:** When the scene is static and transient objects are not a concern.

## Background
SWAG was introduced as an extension to 3D Gaussian Splatting to handle transient objects in unposed image collections. It builds on the principles of 3DGS and introduces the concept of unsupervised transient Gaussians to ignore occluders [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
