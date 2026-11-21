# Gaussian Rasterization Variable Preparation

Also called: render variable setup, splat parameter extraction

## Overview
This method prepares the input data structures required by the differentiable Gaussian rasterizer kernel. It filters raw Gaussian parameters by a specified timestep, normalizes and converts latent representations (logits, logs) into physical attributes (opacities, scales), and packages them into a dictionary (`rendervar`) compatible with the CUDA backend. It distinguishes itself by handling temporal filtering and specific tensor shape adjustments (such as tiling single-dimension scales) before the rasterization call, ensuring the kernel receives correctly formatted and normalized data [S1, unofficial].

## How it works
**Inputs:**
- `params`: A dictionary containing raw Gaussian parameters. Key fields include `timestep` (integer/float), `log_scales` (tensor), `logit_opacities` (tensor), `unnorm_rotations` (tensor), `rgb_colors` (tensor), and `means3D` (tensor).
- `curr_timestep`: The current time step index used for filtering [S1, unofficial].
- `w2c`: World-to-camera transformation matrix (required only for depth rendering variant) [S1, unofficial].

**Outputs:**
- `rendervar`: A dictionary of PyTorch tensors formatted for standard color rendering.
- `depth_rendervar`: A dictionary of PyTorch tensors formatted for depth/silhouette rendering.

**Data Structures:**
The output is a Python dictionary (`rendervar`) with the following keys:
- `means3D`: 3D mean positions of Gaussians.
- `colors_precomp`: Pre-computed colors (RGB).
- `rotations`: Normalized rotation quaternions.
- `opacities`: Scalar opacity values.
- `scales`: Log-scaled dimensions.
- `means2D`: 2D projected means, initialized to zeros.

**Steps:**
1. **Temporal Filtering**: Identify active Gaussians by filtering `params['timestep']` where the value is less than or equal to `curr_timestep`. Use boolean indexing to extract the subset of parameters, resulting in `selected_params` [S1, unofficial].
2. **Scale Tiling**: Check the last dimension of `selected_params['log_scales']`. If the size is 1, tile this dimension to 3 using `torch.tile` to match the expected 3D scale format [S1, unofficial].
3. **Opacity Conversion**: Compute final opacities by applying the sigmoid function to `selected_params['logit_opacities']`:
   $$ \text{opacities} = \sigma(\text{logit\_opacities}) $$
4. **Scale Conversion**: Compute final scales by applying the exponential function to the (possibly tiled) `log_scales`:
   $$ \text{scales} = \exp(\text{log\_scales}) $$
5. **Rotation Normalization**: Normalize the `unnorm_rotations` using L2 normalization to ensure valid unit quaternions:
   $$ \text{rotations} = \frac{\text{unnorm\_rotations}}{\|\text{unnorm\_rotations}\|_2} $$
6. **Means2D Initialization**: Create a zero tensor on the CUDA device with the same shape as `means3D` to serve as the placeholder for 2D projected means [S1, unofficial].
7. **Construct `rendervar`**: Assemble the dictionary with keys `means3D`, `colors_precomp` (taken from `rgb_colors`), `rotations`, `opacities`, `scales`, and `means2D` [S1, unofficial].
8. **Construct `depth_rendervar`**: Assemble a similar dictionary, but replace `colors_precomp` with the output of a helper function `get_depth_and_silhouette(selected_params['means3D'], w2c)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def prepare_gaussian_variables(params, curr_timestep, w2c=None):
    # 1. Filter by timestep
    mask = params['timestep'] <= curr_timestep
    selected_params = {k: v[mask] for k, v in params.items()}
    
    # 2. Handle scale dimension
    log_scales = selected_params['log_scales']
    if log_scales.shape[-1] == 1:
        log_scales = torch.tile(log_scales, (1, 3))
        
    # 3. Compute opacities
    opacities = torch.sigmoid(selected_params['logit_opacities'])
    
    # 4. Compute scales
    scales = torch.exp(log_scales)
    
    # 5. Normalize rotations
    rotations = F.normalize(selected_params['unnorm_rotations'], p=2, dim=-1)
    
    # 6. Initialize means2D
    means3D = selected_params['means3D']
    means2D = torch.zeros_like(means3D, device=means3D.device)
    
    # 7. Construct rendervar for color
    rendervar = {
        'means3D': means3D,
        'colors_precomp': selected_params['rgb_colors'],
        'rotations': rotations,
        'opacities': opacities,
        'scales': scales,
        'means2D': means2D
    }
    
    # 8. Construct rendervar for depth
    depth_rendervar = rendervar.copy()
    if w2c is not None:
        # Helper function implementation detail: projects means3D using w2c
        # to get depth and silhouette values
        depth_rendervar['colors_precomp'] = get_depth_and_silhouette(means3D, w2c)
    else:
        # Fallback if w2c not provided, though typically required for depth variant
        depth_rendervar['colors_precomp'] = torch.zeros_like(means3D)
        
    return rendervar, depth_rendervar

def get_depth_and_silhouette(means3D, w2c):
    # [G] Implementation detail: Projects 3D means to 2D using w2c.
    # Returns depth values and silhouette mask as a tensor.
    # This is a placeholder for the specific projection logic defined in the source context.
    pass
```

**Parameters:**
- `curr_timestep`: Integer or float indicating the current time step. Typical values depend on the sequence length [S1, unofficial].

**Edge Cases:**
- If `log_scales` has a last dimension of 1, it must be tiled to 3; failure to do so causes shape mismatch in the rasterizer [S1, unofficial].
- `means2D` must be initialized on the same device (CUDA) as `means3D` [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of active Gaussians $N$, due to filtering, element-wise operations (sigmoid, exp, normalize), and dictionary construction.
- **Memory Complexity**: Linear with respect to $N$, as it creates new tensors for the output dictionaries.
- **Strengths**: Handles temporal sparsity via filtering; ensures numerical stability via normalization and sigmoid/exp conversions.
- **Limitations**: Requires `w2c` for depth rendering variant; relies on specific tensor shapes from the upstream parameter storage.

## Background
This method is part of the SplaTAM pipeline, which integrates Gaussian Splatting with SLAM (Simultaneous Localization and Mapping) for dynamic scene reconstruction. It was introduced in the context of learning and training methods for Gaussian Avatar Reconstruction, specifically within the 3D Gaussian Splatting differentiable rasterizer implementation [S1, unofficial].

## Sources
[S1] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
