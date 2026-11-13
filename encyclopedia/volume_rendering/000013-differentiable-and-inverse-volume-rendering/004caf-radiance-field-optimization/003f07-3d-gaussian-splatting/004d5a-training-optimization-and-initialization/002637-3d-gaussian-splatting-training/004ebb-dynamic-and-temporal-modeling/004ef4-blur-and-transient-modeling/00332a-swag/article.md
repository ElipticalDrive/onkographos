# SWAG
Also called: SWAG: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG extends 3D Gaussian Splatting to handle unstructured, casual photo collections containing transient occluders (e.g., pedestrians, vehicles) without requiring explicit segmentation masks or temporal synchronization [S1][S2, unofficial]. It distinguishes itself from supervised transient modeling methods by training "unsupervised transient Gaussians" that learn to ignore occluders, conditioned on learned appearance embeddings, thereby achieving state-of-the-art reconstruction quality on diverse photo scenes [S1][S2, unofficial].

## How it works
SWAG builds on the static 3D Gaussian Splatting pipeline [Parent](9265-dynamic-and-temporal-modeling.md) by introducing a dual-Gaussian representation and appearance-conditioned training.

**Inputs and Outputs:**
*   **Input:** A set of unstructured images $\{I_i\}$ captured from varying viewpoints, potentially containing transient objects. No temporal order or masks are required.
*   **Output:** A set of 3D Gaussians representing the static scene geometry and appearance, robust to transient distractors.

**Data Structures:**
The method maintains two sets of Gaussians for each spatial location or cluster:
1.  **Static Gaussians:** Represent the permanent scene structure. Attributes include position $\mathbf{c}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, and spherical harmonic coefficients $\mathbf{c}_{sh}$ for color.
2.  **Transient Gaussians:** A parallel set of Gaussians initialized to capture transient artifacts. These are trained to have low opacity or are explicitly suppressed during the final rendering of the static scene.

**Method Steps:**
1.  **Initialization:** Initialize a set of 3D Gaussians from the input images using standard SfM (Structure from Motion) points or random initialization, similar to standard 3DGS.
2.  **Appearance Embedding:** Learn a set of appearance embeddings $\mathbf{e}_i$ for each input image $I_i$. These embeddings capture view-dependent and lighting variations.
3.  **Conditioning:** Condition the Gaussian attributes (specifically opacity and color) on the learned appearance embeddings. The color $\mathbf{c}$ of a Gaussian is modulated by the embedding $\mathbf{e}_i$ of the camera view:
    $$ \mathbf{c}_{rendered} = \text{Decode}(\mathbf{c}_{sh}, \mathbf{e}_i) $$
    [G] The decoding function is typically a small MLP or linear projection.
4.  **Transient Gaussian Training:** Train the "unsupervised transient Gaussians" alongside the static ones. The loss function encourages these Gaussians to explain pixel residuals that cannot be explained by the static scene and appearance embeddings.
5.  **Occluder Ignorance:** During optimization, the transient Gaussians absorb the photometric error caused by occluders. The static Gaussians are regularized to remain consistent across views, effectively ignoring the transient components.
6.  **Rendering:** To render the static scene, only the static Gaussians are rasterized. The transient Gaussians are discarded or rendered with zero opacity.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_swag(images, cameras):
    # Initialize static and transient Gaussians
    static_gaussians = initialize_gaussians(images)
    transient_gaussians = initialize_gaussians(images)
    
    # Learn appearance embeddings for each image
    appearance_embeddings = learn_embeddings(images)
    
    for iteration in range(num_iterations):
        # Sample a random image and camera
        img_idx = random_sample(images)
        image = images[img_idx]
        camera = cameras[img_idx]
        embedding = appearance_embeddings[img_idx]
        
        # Render static scene
        static_render = render_gaussians(static_gaussians, camera)
        
        # Render transient scene (optional, for loss calculation)
        transient_render = render_gaussians(transient_gaussians, camera)
        
        # Compute photometric loss between rendered static scene and image
        # The transient Gaussians are trained to minimize the residual
        static_loss = l1_loss(static_render, image)
        
        # Transient loss encourages transient Gaussians to explain residuals
        residual = image - static_render
        transient_loss = l1_loss(transient_render, residual)
        
        # Total loss
        loss = static_loss + lambda_transient * transient_loss
        
        # Backpropagate and update Gaussians and embeddings
        update_parameters(static_gaussians, transient_gaussians, appearance_embeddings, loss)
        
    # Final static scene consists only of static_gaussians
    return static_gaussians
```

**Parameters:**
*   $\lambda_{transient}$: Weight for the transient loss. Not given in the sources.
*   Learning rate for Gaussians and embeddings: Standard 3DGS values. Not given in the sources.

**Edge Cases:**
*   **Highly Dynamic Scenes:** If the scene is predominantly dynamic, the static Gaussians may fail to converge. Not given in the sources.
*   **Transient Object Persistence:** If a transient object appears in all views, it may be incorrectly classified as static. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to standard 3DGS, with a slight increase due to the additional transient Gaussians and appearance embedding optimization. Not given in the sources.
*   **Memory Complexity:** Memory usage increases by a factor of roughly 2 due to the dual Gaussian representation (static and transient). Not given in the sources.
*   **Strengths:** Robust to unstructured data; no need for masks or temporal synchronization; high-quality static reconstruction.
*   **Limitations:** Cannot render transient objects explicitly; performance depends on the ability to separate static and transient components via photometric residuals.

## Background
SWAG was introduced as a method to handle unstructured photo collections in 3D Gaussian Splatting. It extends the 3DGS framework by conditioning Gaussians on learned appearance embeddings and training unsupervised transient Gaussians to ignore occluders [S1][S2, unofficial]. It yields state-of-the-art results on diverse photo scenes [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
