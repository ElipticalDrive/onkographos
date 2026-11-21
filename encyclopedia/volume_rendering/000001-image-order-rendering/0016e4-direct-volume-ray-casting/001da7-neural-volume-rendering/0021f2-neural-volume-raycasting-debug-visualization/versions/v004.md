# Neural Volume Raycasting Debug Visualization

Also called: Debug Visualization Pipeline for Neural Volumes

## Overview
This method provides a diagnostic visualization pipeline for neural volume models by rendering individual sample points as instanced spheres, colored by the network's predicted radiance and filtered by an opacity threshold [S1, unofficial]. It differs from standard rendering methods by bypassing front-to-back compositing to inspect the raw distribution and magnitude of neural network outputs in 3D space, serving as a tool to verify that the implicit representation is capturing scene geometry and appearance correctly [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A dataset of rays associated with camera poses, a trained neural network model $f_\theta$, and visualization parameters (empty threshold, number of samples per ray).
*   **Output**: A 3D scene representation (e.g., HTML/Scenepic format) containing instanced sphere meshes representing non-empty samples, colored by predicted radiance.

**Data Structures**
*   **Ray Samples**: A tensor of shape `(num_rays, num_samples, 3)` containing 3D spatial coordinates $\mathbf{x}$.
*   **View Directions**: A tensor of shape `(num_rays, num_samples, 3)` containing viewing directions $\mathbf{d}$ (if the model is view-dependent).
*   **Model Output**: A tensor of shape `(num_rays, num_samples, 4)` containing raw predictions. The last dimension is split into color $\mathbf{c} \in \mathbb{R}^3$ and opacity $\sigma \in \mathbb{R}^1$.
*   **Instance Data**: A list or array of sphere meshes where each instance has a position, a base color (white or black), and a per-instance color attribute.

**Parameters**
*   `empty_threshold`: Opacity threshold to classify samples as empty. Default: $0.1$ [S1, unofficial].
*   `num_samples`: Number of samples per ray used for reshaping model output. Default: $64$ [S1, unofficial].
*   `sphere_scale`: Scale factor for the instanced spheres. Default: $0.02$ [S1, unofficial].

**Steps**
1.  **Ray Subsampling**: Select a subset of rays from the dataset using a precomputed index set to manage memory and visualization density [S1, unofficial].
    ```python
    # Example: dataset.subsample_index = set(index.tolist())
    ```
2.  **Ray Fetching**: Iterate over each camera in the dataset and fetch the corresponding ray samples (positions and optionally view directions) [S1, unofficial].
3.  **Device Transfer and Reshaping**: Move the ray sample positions to the model's compute device (e.g., GPU) and reshape them to `(-1, 3)` to flatten the batch and ray dimensions [S1, unofficial]. If the model uses view directions, reshape them similarly to `(-1, 3)` [S1, unofficial].
4.  **Neural Query**: Pass the positions (and view directions, if applicable) to the neural network model to obtain raw outputs [S1, unofficial].
    *   If view-dependent: `raw_output = model(positions, views)`
    *   If view-independent: `raw_output = model(positions)`
5.  **Reshape and Split**: Reshape the model output to `(-1, num_samples, 4)` and split the last dimension into color (3 channels) and opacity (1 channel) [S1, unofficial].
    ```python
    color_raw, opacity_raw = torch.split(raw_output, [3, 1], dim=-1)
    ```
6.  **Activation Functions**: Apply activation functions to constrain the outputs to valid ranges [S1, unofficial]:
    *   Color: Apply sigmoid to clamp values to $[0, 1]$.
        $$ \mathbf{c} = \text{sigmoid}(\mathbf{c}_{\text{raw}}) $$
    *   Opacity: Apply softplus to ensure non-negative values.
        $$ \sigma = \text{softplus}(\sigma_{\text{raw}}) = \ln(1 + e^{\sigma_{\text{raw}}}) $$
7.  **CPU Conversion**: Convert tensors to CPU numpy arrays and reshape positions to `(-1, 3)` and colors to `(-1, 3)` for visualization [S1, unofficial].
8.  **Empty Classification**: Classify samples as empty or non-empty based on the opacity threshold [S1, unofficial]:
    ```python
    empty = opacity < empty_threshold
    ```
9.  **Mesh Generation**: Create instanced sphere meshes [S1, unofficial]:
    *   For non-empty samples: Use a white base color with the per-instance predicted color $\mathbf{c}$.
    *   For empty samples: Use a shared black color.
    *   Scale all spheres by `sphere_scale` (default $0.02$) [S1, unofficial].
10. **Scene Export**: Save the resulting scene (e.g., as HTML/Scenepic) for interactive inspection [S1, unofficial].

**Pitfalls**
*   **Shape Mismatch**: The model output shape must match the expected `(batch, num_samples, 4)` layout. If the model returns a different shape, the split and reshape operations will fail or misalign color/opacity channels [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays and samples per ray, dominated by the neural network forward pass and tensor operations.
*   **Memory Complexity**: Linear with respect to the number of rays and samples, storing positions, colors, and opacity values for all samples.
*   **Strengths**: Provides immediate visual feedback on the neural network's ability to represent density and color; helps identify artifacts like "ghosting" or incorrect density estimates.
*   **Limitations**: Not suitable for final image synthesis due to lack of compositing and potential overdraw; performance degrades with very high sample counts per ray.
*   **When to Use**: During model development and debugging to verify that the neural volume is learning the correct scene structure.
*   **When Not to Use**: For final rendering or performance-critical applications.

## Background
This method originates from the implementation of Fourier Feature Networks for neural volume rendering, specifically within the context of debugging and visualizing the learned implicit representations [S1, unofficial]. It is part of a broader effort to make neural rendering pipelines interpretable by exposing the raw outputs of the neural network in 3D space.

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
