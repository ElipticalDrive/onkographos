# Neural Gaussian Rasterization Pipeline

Also called: DNGaussian rendering, Neural opacity/color rasterization

## Overview
The Neural Gaussian Rasterization Pipeline replaces the standard exponential opacity formulation of 3D Gaussian Splatting with direct neural network outputs for per-Gaussian opacity and color, while retaining the standard 2D Gaussian rasterizer for compositing [S1, unofficial]. It differs from standard Gaussian Splatting by decoupling the appearance and density representation from the explicit Gaussian parameters, allowing for higher-fidelity reconstruction of complex materials and lighting effects [S1, unofficial].

## How it works
The pipeline integrates a neural density network (MLP) into the Gaussian Splatting rendering loop. It computes view-dependent color and density (sigma) for each Gaussian center, maps sigma directly to opacity, and composites the result using the standard alpha-blending rasterizer.

### Inputs and Outputs
*   **Inputs**:
    *   `viewpoint_camera`: Camera parameters including `camera_center`, `FoVx`, `FoVy`, projection matrix, and view matrix.
    *   `pc` (GaussianModel): The set of 3D Gaussians containing `xyz` (3D positions), `scaling_modifier`, and a `neural_renderer` (MLP).
    *   `background`: A tensor of shape $(3,)$ representing the background color.
    *   `pipe`: A configuration object containing `compute_cov3D_python`, `scale_modifier`, and other rasterization settings.
*   **Outputs**:
    *   `rendered_image`: Tensor of shape $(3, H, W)$ containing the final RGB image.
    *   `radii`: Tensor of shape $(N,)$ indicating which Gaussians are visible (radius > 0).
    *   `rendered_depth`: Tensor of shape $(H, W)$ containing the accumulated depth.
    *   `rendered_alpha`: Tensor of shape $(H, W)$ containing the accumulated alpha.

### Data Structures
*   **GaussianModel (`pc`)**:
    *   `xyz`: Tensor of shape $(N, 3)$, 3D positions of $N$ Gaussians.
    *   `neural_renderer`: A neural network (MLP) that takes $(3,)$ position and $(3,)$ direction as input and outputs $(1,)$ sigma and $(3,)$ color.
    *   `get_covariance(scaling_modifier)`: Method to compute the 3D covariance matrix.
    *   `get_opacity_()`: Method to retrieve standard learned opacity if neural mode is disabled.
*   **GaussianRasterizationSettings**:
    *   `image_width`, `image_height`: Integer dimensions of the output image.
    *   `tanfovx`, `tanfovy`: Tangent of half horizontal and vertical field of view.
    *   `background`: Background color tensor.
    *   `scale_modifier`: Float multiplier for Gaussian sizes.
    *   `viewmatrix`, `projmatrix`: Camera transformation matrices.
    *   `sh_degree`: Spherical Harmonics degree (used if not fully neural).
    *   `camera_position`: Camera center vector.
    *   `prefiltered`: Boolean, typically `False` for neural variants.

### Steps
1.  **Compute Direction Vectors**: Calculate the vector from the camera center to each Gaussian center.
    $$ \mathbf{dir}_{pp} = \mathbf{p}_c - \mathbf{c}_{cam} $$
    where $\mathbf{p}_c$ is the tensor of Gaussian positions $(N, 3)$ and $\mathbf{c}_{cam}$ is the camera center $(3,)$.
2.  **Normalize Directions**: Normalize $\mathbf{dir}_{pp}$ along the batch dimension.
    $$ \mathbf{dir}_{pp\_normalized} = \frac{\mathbf{dir}_{pp}}{\|\mathbf{dir}_{pp}\|_2} $$
3.  **Neural Query**: Pass the 3D positions and normalized directions to the `neural_renderer` to obtain density ($\sigma$) and color ($\mathbf{c}$).
    $$ \sigma, \mathbf{c} = \text{neural\_renderer}(\mathbf{p}_c, \mathbf{dir}_{pp\_normalized}) $$
4.  **Opacity Assignment**: Assign opacity directly from $\sigma$ without the standard $1 - e^{-\sigma}$ transformation.
    $$ \alpha_{neural} = \sigma $$
    *Note: In inference mode, the standard `pc.get_opacity_()` may override this [S1, unofficial].*
5.  **Initialize Screenspace Points**: Prepare the 2D mean positions (`means2D`) for rasterization. These must be initialized with `requires_grad=True` to enable gradient flow back to 2D means during optimization [S1, unofficial].
6.  **Configure Rasterization Settings**:
    *   Compute tangent FOV:
        $$ \text{tanfovx} = \tan\left(\frac{\text{FoVx}}{2}\right), \quad \text{tanfovy} = \tan\left(\frac{\text{FoVy}}{2}\right) $$
    *   Set `scale_modifier` (default 1.0) [S1, unofficial].
    *   Select covariance computation: If `pipe.compute_cov3D_python` is true, compute `cov3D_precomp` via `pc.get_covariance(scaling_modifier)`; otherwise, use precomputed scales and rotations [S1, unofficial].
7.  **Rasterize**: Execute the `GaussianRasterizer` with:
    *   `means3D`: 3D positions.
    *   `means2D`: Projected 2D positions.
    *   `colors_precomp`: Neural color output $\mathbf{c}$.
    *   `opacities`: Neural opacity $\alpha_{neural}$.
    *   `scales`/`rotations` or `cov3D_precomp`: Geometric parameters.
    *   `background`: Background color.
8.  **Filter Visibility**: Check `radii > 0` to exclude frustum-culled or zero-radius Gaussians from further processing (e.g., splitting criteria) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def render_neural(viewpoint_camera, pc, pipe, background):
    """
    Renders a view using the Neural Gaussian Rasterization Pipeline.
    
    Args:
        viewpoint_camera: Object with attributes camera_center, FoVx, FoVy, 
                          viewmatrix, projmatrix.
        pc: GaussianModel object with attributes xyz, neural_renderer, 
            get_covariance, get_opacity_, combine_opacity.
        pipe: PipelineParams object with attributes compute_cov3D_python, 
              scale_modifier.
        background: Tensor of shape (3,) for background color.
        
    Returns:
        render_pkg: Dictionary containing 'render', 'radii', 'depth', 'alpha'.
    """
    # 1. Compute direction vectors from camera center to each Gaussian
    # Repeat camera center to match batch size of Gaussians
    cam_center = viewpoint_camera.camera_center.repeat(pc.get_xyz.shape[0], 1)
    dir_pp = pc.get_xyz - cam_center
    
    # 2. Normalize direction vectors along the batch dimension
    dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)
    
    # 3. Query the neural renderer with 3D positions and normalized directions
    # Move to CUDA if necessary (assumed device handling)
    xyz_cuda = pc.get_xyz.cuda()
    dir_cuda = dir_pp_normalized.cuda()
    
    sigma, color = pc.neural_renderer(xyz_cuda, dir_cuda)
    
    # 4. Assign opacity directly from sigma
    # View as (-1, 1) for compatibility with rasterizer opacity tensor
    opacity = sigma.view(-1, 1)
    
    # 5. Combine opacity (handles inference mode override if needed)
    # [G] combine_opacity is assumed to be a method on pc that applies 
    # standard opacity if inference=True, otherwise uses neural opacity.
    final_opacity = pc.combine_opacity(opacity)
    
    # 6. Compute tangent half-angles
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    # 7. Configure GaussianRasterizationSettings
    # [G] Assuming GaussianRasterizationSettings is a class that stores these params
    # for the rasterizer kernel.
    raster_settings = GaussianRasterizationSettings(
        image_height=viewpoint_camera.image_height, # Assumed attribute
        image_width=viewpoint_camera.image_width,   # Assumed attribute
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=background,
        scale_modifier=pipe.scale_modifier,
        viewmatrix=viewpoint_camera.viewmatrix,
        projmatrix=viewpoint_camera.projmatrix,
        sh_degree=pc.active_sh_degree, # Assumed attribute
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=False
    )
    
    # 8. Select covariance computation mode
    if pipe.compute_cov3D_python:
        cov3D_precomp = pc.get_covariance(pipe.scale_modifier)
        scales = None
        rotations = None
    else:
        cov3D_precomp = None
        scales = pc.get_scaling # Assumed attribute
        rotations = pc.get_rotation # Assumed attribute
        
    # 9. Initialize screenspace points (means2D)
    # [G] This step typically involves projecting means3D to 2D and setting 
    # requires_grad=True. The exact projection logic is handled by the 
    # rasterizer's internal setup or a preceding step in the pipeline.
    # For the purpose of this method description, we assume means2D is 
    # prepared and available.
    means2D = prepare_means2D(pc.get_xyz, raster_settings) # [G] Helper
    
    # 10. Execute rasterizer
    rasterizer = GaussianRasterizer(raster_settings)
    
    rendered_image, radii, rendered_depth, rendered_alpha = rasterizer(
        means3D=pc.get_xyz,
        means2D=means2D,
        colors_precomp=color,
        opacities=final_opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )
    
    # 11. Filter visibility (for downstream tasks like splitting)
    # radii > 0 indicates visible Gaussians
    visible_mask = radii > 0
    
    return {
        'render': rendered_image,
        'radii': radii,
        'depth': rendered_depth,
        'alpha': rendered_alpha,
        'visible_mask': visible_mask
    }

def prepare_means2D(means3D, raster_settings):
    """
    [G] Placeholder for the projection logic that converts 3D means to 2D 
    screen coordinates. In the standard Gaussian Splatting pipeline, this 
    is often done inside the rasterizer or a pre-processing step.
    """
    # This function would perform the perspective projection using 
    # raster_settings.viewmatrix and raster_settings.projmatrix.
    # It returns a tensor of shape (N, 2) with requires_grad=True.
    pass
```

### Parameters
*   `scaling_modifier`: Scales the Gaussian covariance/sizes before rasterization. Default value is 1.0 [S1, unofficial].
*   `inference`: Boolean flag. When true, overrides neural opacity with `pc.get_opacity_()` instead of the neural output [S1, unofficial].
*   `value`: Constant opacity multiplier used in depth-only rendering mode. Default value is 0.95 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: The neural renderer outputs $\sigma$ directly as opacity. If $\sigma$ values are very large, standard alpha blending may saturate. The standard $1 - e^{-\sigma}$ transformation is omitted, which simplifies the gradient flow but requires careful initialization of the neural network to prevent exploding opacities [S1, unofficial].
*   **Boundaries**: Frustum culling is handled by the rasterizer's `radii` output. Gaussians with `radii == 0` are excluded from splitting criteria [S1, unofficial].
*   **Degenerate Input**: If `dir_pp` is zero (Gaussian at camera center), normalization fails. [G] A small epsilon should be added to the denominator during normalization to prevent NaN gradients.

## Operating characteristics
*   **Time Complexity**: Dominated by the neural network inference for each Gaussian ($O(N \cdot D_{MLP})$) and the rasterization step ($O(N \cdot P)$ where $P$ is pixels). The neural query adds significant overhead compared to standard Gaussian Splatting, which uses precomputed colors.
*   **Memory Complexity**: Requires storing the neural network weights and the output tensors for $\sigma$ and color for all Gaussians.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Performance scales linearly with the number of Gaussians $N$ and the complexity of the neural network.
*   **Strengths**: Allows for view-dependent appearance and complex material representation without explicit spherical harmonics or texture maps.
*   **Limitations**: Slower rendering due to per-Gaussian neural queries. Requires careful tuning of the neural network architecture and training.
*   **When to use**: When high-fidelity view-dependent effects are required and rendering speed is secondary to quality.
*   **When not to use**: For real-time applications requiring high frame rates with large numbers of Gaussians.

## Background
The Neural Gaussian Rasterization Pipeline was introduced in the DNGaussian paper [S1, unofficial]. It extends the 3D Gaussian Splatting framework by integrating a neural density network to handle opacity and color, aiming to improve reconstruction quality for complex scenes.

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
