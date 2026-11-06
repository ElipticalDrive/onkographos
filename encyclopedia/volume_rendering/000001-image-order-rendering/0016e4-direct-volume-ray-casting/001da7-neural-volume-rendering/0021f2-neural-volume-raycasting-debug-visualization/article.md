# Neural Volume Raycasting Debug Visualization

Also called: Debug Visualization Pipeline for Neural Volumes

## Overview
This method provides a diagnostic visualization pipeline for neural volume models by rendering raw sample points as instanced spheres rather than compositing them into a final image [S1, unofficial]. It visualizes the spatial distribution of samples and their predicted radiance, filtered by an opacity threshold, to debug model convergence and sampling density [S1, unofficial]. Unlike standard neural volume rendering which outputs a pixel-perfect image via front-to-back compositing, this method isolates individual samples for inspection [S1, unofficial].

## How it works
**Inputs:**
- A trained neural volume model (e.g., MLP with positional encoding) [S1, unofficial].
- A dataset of camera rays, where each ray contains positions and optionally view directions [S1, unofficial].
- Parameters: `empty_threshold` (default 0.1) and `num_samples` (default 64) [S1, unofficial].

**Outputs:**
- A 3D scene representation containing instanced sphere meshes for each sample point [S1, unofficial].

**Data Structures:**
- **Rays:** Batched tensors of shape `(batch_size, num_samples, 3)` for positions and `(batch_size, num_samples, 3)` for view directions [S1, unofficial].
- **Model Output:** Tensor of shape `(batch_size, num_samples, 4)` containing raw color (3 channels) and opacity (1 channel) [S1, unofficial].
- **Sample Classification:** Boolean mask `empty` of shape `(batch_size, num_samples)` where `True` indicates opacity below `empty_threshold` [S1, unofficial].

**Steps:**
1.  **Ray Subsampling:** Select a subset of rays from the dataset using a precomputed index set to reduce computational load during debugging [S1, unofficial].
2.  **Ray Fetching:** Iterate over each camera in the dataset and fetch the corresponding ray samples [S1, unofficial].
3.  **Data Preparation:** Move ray positions and view directions to the model's compute device (e.g., GPU) and reshape positions to `(-1, 3)` [S1, unofficial].
4.  **Model Inference:** Pass positions (and view directions if the model requires them) to the neural network [S1, unofficial].
    -   If view-dependent: `color_o = model(positions, views)`
    -   If view-independent: `color_o = model(positions)`
5.  **Reshape and Split:** Reshape the output tensor to `(-1, num_samples, 4)` and split the last dimension into color `C` (3 channels) and opacity `A` (1 channel) [S1, unofficial].
6.  **Activation Functions:** Apply activation functions to the raw outputs:
    -   Color: Apply sigmoid to clamp values to $[0, 1]$: $C_{final} = \sigma(C_{raw})$ [S1, unofficial].
    -   Opacity: Apply softplus to ensure non-negative values: $A_{final} = \text{softplus}(A_{raw})$ [S1, unofficial].
7.  **Empty Sample Classification:** Compare opacity against `empty_threshold` (default 0.1) [S1, unofficial]:
    -   If $A_{final} < \text{empty\_threshold}$, mark sample as empty.
    -   Otherwise, mark sample as non-empty.
8.  **Mesh Generation:** Create instanced sphere meshes for visualization [S1, unofficial]:
    -   **Non-empty samples:** Use the predicted color $C_{final}$ as the instance color with a white base.
    -   **Empty samples:** Use a shared black color.
    -   Scale all spheres to a fixed size (e.g., 0.02) [S1, unofficial].
9.  **Scene Export:** Convert the mesh data to a scene format (e.g., HTML/Scenepic) for interactive inspection [S1, unofficial].

**Pitfalls:**
-   **Shape Mismatch:** The model output shape must strictly match `(batch, num_samples, 4)`. If the model returns a different shape, the split and reshape operations will fail or misalign color/opacity channels [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def debug_visualize_neural_volume(model, dataset, args):
    """
    Visualizes neural volume samples as instanced spheres for debugging.
    """
    # 1. Subsample rays if specified
    if args.num_cameras and args.num_cameras < dataset.num_cameras:
        dataset = dataset.sample_cameras(args.num_cameras, args.num_samples, args.stratified)

    # 2. Fetch rays for all cameras
    # Note: In a real implementation, this iterates over cameras and batches rays
    # For this pseudocode, we assume a batched tensor of rays is available
    # ray_samples shape: (batch_size, num_samples, 3) for positions
    # views shape: (batch_size, num_samples, 3) for view directions (if applicable)
    
    # Move to device
    positions = dataset.positions.to(model.device) # Shape: (batch_size, num_samples, 3)
    views = dataset.views.to(model.device) if hasattr(dataset, 'views') else None
    
    # Reshape positions to (-1, 3) for model input
    batch_size, num_samples, _ = positions.shape
    positions_flat = positions.view(-1, 3)
    
    if views is not None:
        views_flat = views.view(-1, 3)
        # 4. Model Inference with view directions
        color_o = model(positions_flat, views_flat)
    else:
        # 4. Model Inference without view directions
        color_o = model(positions_flat)
        
    # 5. Reshape and Split
    # Output shape: (batch_size * num_samples, 4)
    # Reshape to (-1, num_samples, 4)
    color_o = color_o.view(batch_size, num_samples, 4)
    color_raw, opacity_raw = torch.split(color_o, [3, 1], dim=-1)
    
    # 6. Activation Functions
    # Clamp color to [0, 1] using sigmoid
    color = torch.sigmoid(color_raw)
    # Ensure non-negative opacity using softplus
    opacity = F.softplus(opacity_raw)
    
    # 7. Empty Sample Classification
    empty_threshold = args.empty_threshold if hasattr(args, 'empty_threshold') else 0.1
    empty = opacity < empty_threshold
    
    # 8. Mesh Generation (Conceptual)
    # Non-empty samples: color = color[~empty], base = white
    # Empty samples: color = black, base = black
    # Scale = 0.02
    # This step typically involves creating an InstancedMesh object in a 3D library
    # like Three.js or Scenepic
    
    return color, opacity, empty
```

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of samples processed. The bottleneck is the neural network inference for each sample point [S1, unofficial].
-   **Memory Complexity:** Proportional to the number of samples and the model size. Storing the intermediate tensors for all samples in a batch requires significant VRAM for large scenes [S1, unofficial].
-   **Strengths:** Provides immediate visual feedback on sample density and model predictions (radiance/opacity) without the complexity of full volume rendering [S1, unofficial]. Useful for identifying artifacts like banding or incorrect opacity predictions [S1, unofficial].
-   **Limitations:** Not suitable for final image generation. The visualization is discrete and does not represent the continuous volume [S1, unofficial]. Performance degrades with very high `num_samples` per ray [S1, unofficial].

## Background
This method originates from debugging tools used in research on Fourier Feature Networks and Neural Volume Rendering [S1, unofficial]. It was developed to facilitate the inspection of neural radiance fields (NeRFs) and similar implicit representations by visualizing the underlying sample points and their predicted properties [S1, unofficial].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
