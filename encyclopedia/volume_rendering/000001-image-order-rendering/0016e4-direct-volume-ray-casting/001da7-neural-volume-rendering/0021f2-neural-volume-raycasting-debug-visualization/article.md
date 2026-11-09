# Neural Volume Raycasting Debug Visualization

Also called: Debug Visualization Pipeline for Neural Volumes

## Overview
This method provides a diagnostic visualization pipeline for neural volume models by rendering individual sample points as instanced spheres rather than compositing them into a final image [S1, unofficial]. It solves the problem of verifying that a neural network is correctly querying the spatial domain and producing physically plausible density/radiance values by making the discrete sampling structure visible [S1, unofficial]. It differs from standard [Neural volume rendering](1576-neural-volume-rendering.md) by bypassing the front-to-back compositing stage and instead visualizing the raw output of the neural query at each sample location, filtered by an opacity threshold [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A dataset of camera rays (position and direction), a trained neural network model (MLP), and visualization parameters (`empty_threshold`, `num_samples`) [S1, unofficial].
- **Outputs:** A 3D scene representation containing instanced sphere meshes, where each sphere corresponds to a ray sample point, colored by the model's predicted radiance [S1, unofficial].

**Data Structures:**
- **Ray Samples:** A tensor of shape `(batch_size, num_samples, 3)` containing 3D positions $\mathbf{p}_i$ [S1, unofficial].
- **View Directions:** A tensor of shape `(batch_size, num_samples, 3)` containing viewing directions $\mathbf{d}_i$ (if view-dependent) [S1, unofficial].
- **Model Output:** A tensor of shape `(batch_size, num_samples, 4)` containing raw predictions, split into color $\mathbf{c}_i \in \mathbb{R}^3$ and opacity $\sigma_i \in \mathbb{R}$ [S1, unofficial].
- **Instanced Mesh:** A collection of sphere primitives. Each instance has a base color (white or black) and a per-instance attribute color derived from $\mathbf{c}_i$ [S1, unofficial].

**Parameters:**
- `empty_threshold`: Opacity threshold to filter out negligible samples. Default: 0.1 [S1, unofficial].
- `num_samples`: Number of samples per ray used for reshaping model output. Default: 64 [S1, unofficial].

**Steps:**
1. **Ray Subsampling and Fetching:** Select a subset of rays from the dataset using a precomputed index set to manage computational load [S1, unofficial]. Iterate over each camera in the dataset and fetch the corresponding ray samples (positions and directions) [S1, unofficial].
2. **Neural Query:** Move the ray sample positions to the model's compute device and reshape them to `(-1, 3)` [S1, unofficial]. If the model is view-dependent, reshape view directions to `(-1, 3)` and pass both positions and views to the model; otherwise, pass only positions [S1, unofficial]. Compute the raw output:
   $$
   \mathbf{o} = \text{Model}(\mathbf{p}, \mathbf{d}) \quad \text{or} \quad \mathbf{o} = \text{Model}(\mathbf{p})
   $$
   where $\mathbf{o}$ is the raw output tensor [S1, unofficial].
3. **Reshape and Split:** Reshape the model output to `(-1, num_samples, 4)` and split the last dimension into color (3 channels) and opacity (1 channel) [S1, unofficial]:
   $$
   \mathbf{c}_{raw}, \sigma_{raw} = \text{split}(\mathbf{o}, [3, 1], \text{dim}=-1)
   $$
   [S1, unofficial].
4. **Activation Functions:** Apply activation functions to bound the outputs:
   - Color: Apply sigmoid to clamp to $[0, 1]$:
     $$
     \mathbf{c} = \sigma(\mathbf{c}_{raw})
     $$
     [S1, unofficial].
   - Opacity: Apply softplus to ensure non-negative values:
     $$
     \sigma = \text{softplus}(\sigma_{raw})
     $$
     [S1, unofficial].
5. **Classification:** Convert tensors to CPU numpy arrays [S1, unofficial]. Classify samples as empty or non-empty using the `empty_threshold`:
   $$
   \text{empty}_i = (\sigma_i < \text{empty\_threshold})
   $$
   [S1, unofficial].
6. **Mesh Generation:** Create instanced sphere meshes with a fixed scale of 0.02 [S1, unofficial].
   - For non-empty samples ($\text{empty}_i = \text{False}$): Use a white base color with the per-instance color $\mathbf{c}_i$ [S1, unofficial].
   - For empty samples ($\text{empty}_i = \text{True}$): Use a shared black color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def debug_visualize_rays(dataset, model, empty_threshold=0.1, num_samples=64):
    """
    Visualizes neural volume ray samples as instanced spheres.
    
    Args:
        dataset: RayDataset object with rays_for_camera(i) and subsample_index.
        model: Neural network model taking positions (and optionally views).
        empty_threshold: Float, opacity threshold for filtering.
        num_samples: Int, number of samples per ray.
        
    Returns:
        scene: Scenepic scene object with instanced spheres.
    """
    # 1. Ray Subsampling and Fetching
    # Assume dataset has a method to sample cameras and rays
    # In the source, this is handled by args.num_cameras and dataset.sample_cameras
    # Here we iterate over cameras and fetch rays
    scene = None # Placeholder for scene object creation
    
    for i in range(dataset.num_cameras):
        # Fetch rays for camera i
        ray_samples = dataset.rays_for_camera(i)
        
        # Extract positions and views
        positions = ray_samples['positions'] # Shape: (num_rays, num_samples, 3)
        views = ray_samples.get('views', None)
        
        # 2. Neural Query
        # Move to device (assume model is on cuda)
        positions = positions.to(model.device)
        positions_flat = positions.reshape(-1, 3)
        
        if views is not None:
            views = views.to(model.device)
            views_flat = views.reshape(-1, 3)
            # Pass both positions and views
            color_o = model(positions_flat, views_flat)
        else:
            # Pass only positions
            color_o = model(positions_flat)
            
        # 3. Reshape and Split
        # Reshape to (-1, num_samples, 4)
        # Note: color_o shape is (num_rays * num_samples, 4)
        color_o = color_o.reshape(-1, num_samples, 4)
        
        # Split into color (3) and opacity (1)
        color_raw, opacity_raw = torch.split(color_o, [3, 1], dim=-1)
        
        # 4. Activation Functions
        color = torch.sigmoid(color_raw)
        opacity = F.softplus(opacity_raw)
        
        # 5. Classification
        # Convert to numpy for thresholding
        opacity_np = opacity.cpu().numpy()
        empty = opacity_np < empty_threshold
        
        # Reshape positions and color for mesh generation
        positions_np = positions.cpu().numpy().reshape(-1, 3)
        color_np = color.cpu().numpy().reshape(-1, 3)
        
        # 6. Mesh Generation
        # Create instanced spheres
        # Non-empty samples: white base, per-instance color
        # Empty samples: black base, shared color
        # Scale spheres to 0.02
        
        # Implementation detail: Scenepic or similar library call
        # This is pseudo-code for the logic, not the specific API call
        for idx in range(len(positions_np)):
            pos = positions_np[idx]
            col = color_np[idx]
            is_empty = empty[idx]
            
            if is_empty:
                base_color = [0.0, 0.0, 0.0]
                instance_color = [0.0, 0.0, 0.0]
            else:
                base_color = [1.0, 1.0, 1.0]
                instance_color = col.tolist()
                
            # Add instanced sphere to scene
            # scene.add_instanced_sphere(position=pos, scale=0.02, base_color=base_color, instance_color=instance_color)
            pass
            
    return scene
```

**Edge Cases and Pitfalls:**
- **Model Output Shape:** The model output must match the expected `(batch, num_samples, 4)` layout. If the model returns a different shape, the split and reshape operations will fail or misalign color/opacity channels [S1, unofficial].
- **Precision:** Ensure that the `softplus` activation is used for opacity to prevent negative values, which are physically invalid for density/opacity [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of rays and samples per ray, dominated by the neural network inference step [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of rays and samples, storing positions, views, and model outputs in memory before conversion to numpy [S1, unofficial].
- **Strengths:** Provides immediate visual feedback on the spatial distribution of samples and the network's output range [S1, unofficial].
- **Limitations:** Not suitable for final image generation due to the lack of compositing and the discrete nature of the sphere visualization [S1, unofficial].
- **When to use:** During model development and debugging to verify that the neural network is correctly sampling the volume and producing valid radiance/density values [S1, unofficial].
- **When not to use:** For final rendering or performance-critical applications where compositing is required [S1, unofficial].

## Background
This method originates from the "Fourier Feature Networks and Neural Volume Rendering" repository, which provides a debugging pipeline for visualizing neural volume rendering outputs [S1, unofficial]. It is part of the broader effort to make neural volume rendering models interpretable by exposing the internal sampling and prediction steps [S1, unofficial].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
