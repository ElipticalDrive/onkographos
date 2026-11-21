# Neural Volume Raycasting Debug Visualization

Also called: Debug Visualization Pipeline for Neural Volumes

## Overview
This method provides a diagnostic visualization pipeline for neural volume models by rendering individual sample points as instanced spheres, colored by the network's predicted radiance and filtered by an opacity threshold [S1, unofficial]. It differs from standard volume rendering by bypassing front-to-back compositing to inspect the raw distribution and magnitude of neural network outputs in 3D space [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A dataset of rays associated with camera poses, a trained neural network model $f_\theta$, and visualization parameters (empty threshold, number of samples per ray) [S1, unofficial].
*   **Output**: A 3D scene representation containing instanced sphere meshes representing non-empty samples, colored by predicted radiance [S1, unofficial].

**Data Structures**
*   **Ray Samples**: A tensor of positions $\mathbf{x}$ and optionally view directions $\mathbf{d}$ [S1, unofficial].
*   **Model Output**: A tensor of shape `(batch, num_samples, 4)` containing raw predictions, split into color $\mathbf{c} \in \mathbb{R}^3$ and opacity $\sigma \in \mathbb{R}^1$ [S1, unofficial].
*   **Instance Data**: Sphere meshes where each instance has a position, a base color (white or black), and a per-instance color attribute [S1, unofficial].

**Parameters**
*   `empty_threshold`: Opacity threshold to classify samples as empty. Default: $0.1$ [S1, unofficial].
*   `num_samples`: Number of samples per ray used for reshaping model output. Default: $64$ [S1, unofficial].
*   `sphere_scale`: Scale factor for the instanced spheres. Default: $0.02$ [S1, unofficial].

**Steps**
1.  **Ray Subsampling**: Select a subset of rays from the dataset using a precomputed index set to manage memory and visualization density [S1, unofficial].
2.  **Ray Fetching**: Iterate over each camera in the dataset and fetch the corresponding ray samples (positions and optionally view directions) [S1, unofficial].
3.  **Device Transfer and Reshaping**: Move the ray sample positions to the model's compute device and reshape them to `(-1, 3)` to flatten the batch and ray dimensions [S1, unofficial]. If the model uses view directions, reshape them similarly to `(-1, 3)` [S1, unofficial].
4.  **Neural Query**: Pass the positions (and view directions, if applicable) to the neural network model to obtain raw outputs [S1, unofficial].
5.  **Reshape and Split**: Reshape the model output to `(-1, num_samples, 4)` and split the last dimension into color (3 channels) and opacity (1 channel) [S1, unofficial].
6.  **Activation Functions**: Apply activation functions to constrain the outputs to valid ranges [S1, unofficial]:
    *   Color: Apply sigmoid to clamp values to $[0, 1]$.
        $$ \mathbf{c} = \text{sigmoid}(\mathbf{c}_{\text{raw}}) $$
    *   Opacity: Apply softplus to ensure non-negative values.
        $$ \sigma = \text{softplus}(\sigma_{\text{raw}}) = \ln(1 + e^{\sigma_{\text{raw}}}) $$
7.  **CPU Conversion**: Convert tensors to CPU numpy arrays and reshape positions to `(-1, 3)` and colors to `(-1, 3)` for visualization [S1, unofficial].
8.  **Empty Classification**: Classify samples as empty or non-empty based on the opacity threshold [S1, unofficial].
9.  **Mesh Generation**: Create instanced sphere meshes [S1, unofficial]:
    *   For non-empty samples: Use a white base color with the per-instance predicted color $\mathbf{c}$.
    *   For empty samples: Use a shared black color.
    *   Scale all spheres by `sphere_scale` (default $0.02$) [S1, unofficial].
10. **Scene Export**: Save the resulting scene for interactive inspection [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def softplus(x):
    return math.log(1.0 + math.exp(x))

def debug_visualize(rays_dataset, model, empty_threshold=0.1, num_samples=64, sphere_scale=0.02):
    # 1. Subsample rays using a precomputed index set
    indices = rays_dataset.subsample_index
    for i in indices:
        # 2. Fetch ray samples for the camera
        ray_samples = rays_dataset.rays_for_camera(i)
        positions = ray_samples.positions
        views = ray_samples.views if hasattr(ray_samples, 'views') else None

        # 3. Move to compute device and reshape positions to (-1, 3)
        positions = positions.to(device).reshape(-1, 3)
        if views is not None:
            views = views.to(device).reshape(-1, 3)

        # 4. Neural query
        if views is not None:
            raw_output = model(positions, views)
        else:
            raw_output = model(positions)

        # 5. Reshape to (-1, num_samples, 4) and split into color (3) and opacity (1)
        raw_output = raw_output.reshape(-1, num_samples, 4)
        color_raw, opacity_raw = raw_output.split([3, 1], dim=-1)

        # 6. Apply activation functions (element-wise)
        color = [sigmoid(c) for c in color_raw.flatten()]
        opacity = [softplus(o) for o in opacity_raw.flatten()]

        # 7. Convert to CPU numpy arrays and reshape to (-1, 3)
        color = numpy.array(color).reshape(-1, 3)
        positions = positions.cpu().numpy().reshape(-1, 3)

        # 8. Classify samples as empty or non-empty
        empty_mask = numpy.array(opacity) < empty_threshold

        # 9. Create instanced sphere meshes
        spheres = []
        for idx in range(len(positions)):
            pos = positions[idx]
            col = color[idx]
            if empty_mask[idx]:
                base_color = (0.0, 0.0, 0.0)  # Black
            else:
                base_color = (1.0, 1.0, 1.0)  # White
            spheres.append(InstancedSphere(position=pos, base_color=base_color, instance_color=col, scale=sphere_scale))

        # 10. Export scene
        scene = Scene(spheres)
        scene.save_html(output_path)
```

**Pitfalls**
*   **Shape Mismatch**: The model output shape must match the expected `(batch, num_samples, 4)` layout; if the model returns a different shape, the split and reshape operations will fail or misalign color/opacity channels [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Described in [S1, unofficial] as a debugging visualization pipeline for neural volume models. Not given in the sources.

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
