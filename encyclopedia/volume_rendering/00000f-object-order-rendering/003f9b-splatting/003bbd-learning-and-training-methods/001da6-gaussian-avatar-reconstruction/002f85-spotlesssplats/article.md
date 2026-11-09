# SpotlessSplats
## Overview
SpotlessSplats is a robustness-enhancing method for Gaussian Avatar Reconstruction that mitigates artifacts caused by transient distractors (e.g., moving people, lighting changes) in casual video captures [S1][S2, unofficial]. It improves reconstruction quality by detecting outliers in a richer, pre-trained feature space, allowing the system to ignore these distractors during training [S1][S2, unofficial]. It builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by adding a feature-based outlier detection mechanism.
## How it works
Inputs and outputs:
- Inputs: Multi-view images of a dynamic subject, camera poses, and a parametric mesh (e.g., SMPL/FLAME) for initialization.
- Outputs: Optimized Gaussian parameters (position, covariance, opacity, SH coefficients) constrained to the mesh, robust to transient occlusions.

Data structures:
- Gaussian primitives: Initialized per triangle/vertex of the parametric mesh [S1][S2, unofficial].
- Feature embeddings: Pre-trained feature space vectors associated with each Gaussian or pixel.

Numbered steps:
1. Initialize Gaussians on the parametric mesh surface [S1][S2, unofficial].
2. Deform Gaussians based on pose/shape parameters [S1][S2, unofficial].
3. Render the scene using differentiable splatting [S1][S2, unofficial].
4. Extract features from the rendered image and/or input images using a pre-trained feature extractor [S1][S2, unofficial].
5. Detect outliers in the pre-trained feature space [S1][S2, unofficial].
6. Ignore or down-weight pixels/Gaussians identified as outliers (transient distractors) during the loss computation [S1][S2, unofficial].
7. Optimize Gaussian parameters using gradients from the robustified loss [S1][S2, unofficial].

Formulas:
- Not given in the sources.

Parameters:
- Not given in the sources.

Edge cases and pitfalls:
- Not given in the sources.

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# SpotlessSplats core logic
def train_spotless_splats(images, poses, mesh, feature_extractor, num_iterations):
    # Initialize Gaussians on mesh
    gaussians = initialize_gaussians_on_mesh(mesh)
    
    for iteration in range(num_iterations):
        # Get current view
        image = images[iteration]
        pose = poses[iteration]
        
        # Deform Gaussians
        deformed_gaussians = deform_gaussians(gaussians, pose)
        
        # Render image
        rendered_image = render(deformed_gaussians, pose)
        
        # Extract features
        rendered_features = feature_extractor(rendered_image)
        input_features = feature_extractor(image)
        
        # Detect outliers in feature space
        # Note: Specific outlier detection algorithm not detailed in sources [S1][S2, unofficial]
        outlier_mask = detect_outliers(rendered_features, input_features)
        
        # Compute loss ignoring outliers
        loss = compute_loss(rendered_image, image, outlier_mask)
        
        # Optimize
        optimize(gaussians, loss)
        
    return gaussians
```
## Subtopics

### [Gaussian Splatting](946-gaussian-splatting.md)
The foundational rendering technique.

### [Gaussian Avatars](1477-gaussian-avatars.md)
The base avatar reconstruction method.

### [Relightable Gaussian Codec Avatars](1482-relightable-gaussian-codec-avatars.md)
Adds relighting capabilities.

### [NoPoSplat](1484-noposplat.md)
Pose-free alternative.

### [SMPL-based rigid/non-rigid deformation](8385-smpl-based-rigid-non-rigid-deformation.md)
Deformation models.

## Operating characteristics
Time and memory complexity:
- Not given in the sources.
Measured performance:
- Improves reconstruction quality in casual captures [S1][S2, unofficial].
Scaling:
- Not given in the sources.
Strengths and limitations:
- Strengths: Robustness to transient distractors [S1][S2, unofficial].
- Limitations: Requires a pre-trained feature extractor [S1][S2, unofficial].
When to use it:
- Use when training on casual captures with transient occlusions [S1][S2, unofficial].
When not:
- Not given in the sources.
## Background
Origin:
- Introduced in [S1][S2, unofficial] as an extension to improve robustness in Gaussian Avatar Reconstruction.
History and development:
- Builds on SWAG [S1][S2, unofficial] which uses unsupervised transient Gaussians. SpotlessSplats enhances this by using a richer, pre-trained feature space for outlier detection [S1][S2, unofficial].
Where it is used:
- Gaussian Avatar Reconstruction for casual captures [S1][S2, unofficial].
## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
