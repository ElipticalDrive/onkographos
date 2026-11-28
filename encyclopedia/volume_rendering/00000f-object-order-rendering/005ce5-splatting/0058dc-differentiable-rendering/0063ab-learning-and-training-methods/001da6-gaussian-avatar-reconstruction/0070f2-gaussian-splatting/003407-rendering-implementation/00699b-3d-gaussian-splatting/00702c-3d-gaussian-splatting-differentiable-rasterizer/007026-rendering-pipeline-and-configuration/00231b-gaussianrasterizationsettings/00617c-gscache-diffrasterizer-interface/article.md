# GSCache DiffRasterizer Interface

Also called: DiffRasterizer

## Overview
The GSCache DiffRasterizer Interface is a C++ class interface for differentiable rasterization, managing camera state, frame buffers, and forward/backward passes for Gaussian splatting [S1, unofficial]. It distinguishes itself by providing a unified interface that handles not only rendering but also the integration of optimization steps (densification, pruning) and data access for the Gaussian model parameters [S1, unofficial].

## How it works
### Inputs and Outputs
The interface accepts camera parameters (intrinsic and extrinsic via `m_view_matrix`, `m_projection_matrix`, `m_fovy`, `m_fovx`, `m_aspect_ratio`, `m_position`, `m_center`, `m_up`), Gaussian model parameters (`m_xyz`, `m_features`, `m_scaling`, `m_rotation`, `m_opacity`), and loss gradients for the backward pass [S1, unofficial]. It outputs a rendered image (RGB via `m_rgb`), intermediate buffers (`radii`, `geometry_buffer`, `binning_buffer`, `image_buffer`), and gradients with respect to Gaussian parameters (`d_xyz`, `d_conic`, `d_opacity`, `d_color`, `d_mean2D`, `d_cov3D`, `d_sh`, `d_scale`, `d_rotation`) [S1, unofficial].

### Data Structures
The interface relies on specific structs to manage state and results [S1, unofficial].

**Camera Struct**
Fields: `m_fovy` (float), `m_fovx` (float), `m_aspect_ratio` (float), `m_position` (vector3), `m_center` (vector3), `m_up` (vector3), `m_view_matrix` (Tensor), `m_projection_matrix` (Tensor), `m_position_tensor` (Tensor) [S1, unofficial].

**FrameBuffer Struct**
Fields: `m_dirty_cpu` (bool), `m_dirty_gpu` (bool), `m_size` (ivec2), `m_rgb` (Tensor), `m_rgb_rw_cpu` (Tensor), `m_rgb_rw_gpu` (Tensor) [S1, unofficial].

**DiffRasterizerForwardResult**
Fields: `n_rendered` (int), `framebuffer` (FrameBuffer), `radii` (Tensor), `geometry_buffer` (Tensor), `binning_buffer` (Tensor), `image_buffer` (Tensor) [S1, unofficial].

**DiffRasterizerBackwardResult**
Fields: `radii` (Tensor), `d_xyz` (Tensor), `d_conic` (Tensor), `d_opacity` (Tensor), `d_color` (Tensor), `d_mean2D` (Tensor), `d_cov3D` (Tensor), `d_sh` (Tensor), `d_scale` (Tensor), `d_rotation` (Tensor) [S1, unofficial].

### Numbered Steps
1. **Initialization**: The rasterizer initializes with Gaussian model parameters (`m_xyz`, `m_features`, `m_scaling`, `m_rotation`, `m_opacity`) and camera settings [S1, unofficial]. It allocates internal buffers for rendering and gradient accumulation (`m_xyz_gradient_accum`, `m_denom`) [S1, unofficial].
2. **Visibility Culling**: Call `markVisible` or `setVisible` to determine which Gaussians are within the camera frustum and have non-zero radii, updating the `m_visible` tensor [S1, unofficial].
3. **Forward Pass**: Call `forward` with the current camera state [S1, unofficial]. This performs projection of 3D Gaussians to 2D, sorting by depth, and rasterization onto the frame buffer, returning a `DiffRasterizerForwardResult` [S1, unofficial].
4. **Backward Pass**: Call `backward` with a loss gradient tensor [S1, unofficial]. This computes gradients for all Gaussian parameters based on the forward pass results and the input loss gradient, returning a `DiffRasterizerBackwardResult` [S1, unofficial].
5. **Optimization Step**: The interface provides `optimize()` to trigger parameter updates using an internal optimizer (`m_optimizer`) [S1, unofficial].
6. **Densification and Pruning**: Periodically, `densifyAndPrune` is called with thresholds (`gradient_threshold`, `opacity_threshold`, `extent`, `size_threshold`) to split or clone Gaussians with high gradients or remove those with low opacity or large extent [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Tensor:
    """Standard n-dimensional array for parameters and gradients [G]."""
    def __init__(self, shape, dtype=float):
        self.shape = shape
        self.dtype = dtype
        self.data = [0.0] * shape[0]
    @staticmethod
    def zeros(shape, dtype=float):
        return Tensor(shape, dtype)

class FrameBuffer:
    """Stores rendering state and buffers [S1, unofficial]."""
    def __init__(self):
        self.m_dirty_cpu = False
        self.m_dirty_gpu = False
        self.m_size = (0, 0)
        self.m_rgb = Tensor((0, 0, 3))
        self.m_rgb_rw_cpu = Tensor((0, 0, 3))
        self.m_rgb_rw_gpu = Tensor((0, 0, 3))

class Optimizer:
    """Standard gradient-based parameter updater [G]."""
    def step(self):
        pass

class GSCacheDiffRasterizer:
    def __init__(self, camera, gaussian_model, options):
        self.camera = camera
        self.m_xyz = gaussian_model.m_xyz
        self.m_features = gaussian_model.m_features
        self.m_scaling = gaussian_model.m_scaling
        self.m_rotation = gaussian_model.m_rotation
        self.m_opacity = gaussian_model.m_opacity
        self.m_visible = Tensor.zeros((gaussian_model.num_gaussians,), dtype=bool)
        self.m_max_radii_2d = Tensor.zeros((gaussian_model.num_gaussians,))
        self.m_xyz_gradient_accum = Tensor.zeros((gaussian_model.num_gaussians,))
        self.m_denom = Tensor.zeros((gaussian_model.num_gaussians,))
        self.m_max_sh_degree = options.get('max_sh_degree', 3)
        self.m_active_sh_degree = options.get('active_sh_degree', 0)
        self.m_options = options
        self.m_optimizer = Optimizer()
        self.framebuffer = FrameBuffer()

    def mark_visible(self):
        # Update self.m_visible based on camera frustum and radii [S1, unofficial]
        pass

    def forward(self):
        # Project Gaussians, sort by depth, rasterize to framebuffer [S1, unofficial]
        # Returns DiffRasterizerForwardResult [G]
        pass

    def backward(self, loss_gradient):
        # Compute gradients w.r.t. Gaussian parameters [S1, unofficial]
        # Updates self.m_xyz_gradient_accum and self.m_denom [G]
        # Returns DiffRasterizerBackwardResult [G]
        pass

    def optimize(self):
        self.m_optimizer.step()

    def densify_and_prune(self, gradient_threshold, opacity_threshold, extent, size_threshold):
        # Calls densifyAndClone, densifyAndSplit, prunePoints internally [S1, unofficial]
        pass

    def get_xyz(self): return self.m_xyz
    def get_features(self): return self.m_features
    def get_scaling(self): return self.m_scaling
    def get_rotation(self): return self.m_rotation
    def get_opacity(self): return self.m_opacity
    def get_visible(self): return self.m_visible
    def get_max_radii_2d(self): return self.m_max_radii_2d
    def get_xyz_gradient_accum(self): return self.m_xyz_gradient_accum
    def get_denom(self): return self.m_denom
    def get_max_sh_degree(self): return self.m_max_sh_degree
    def get_active_sh_degree(self): return self.m_active_sh_degree
    def save_ply(self, filename): pass
    def load_ply(self, filename): pass
```

### Parameters
`gradient_threshold` (float), `opacity_threshold` (float), `extent` (float), and `size_threshold` (int) control densification and pruning thresholds [S1, unofficial]. Typical values are not given in the sources.

### Edge Cases and Pitfalls
Precision limits in GPU computations, frustum boundary handling for partially visible Gaussians, and degenerate zero-opacity or zero-radius inputs are not given in the sources.

## Operating characteristics
Time complexity, memory complexity, measured performance, scaling behavior, strengths, limitations, and usage guidelines are not given in the sources.

## Background
The GSCache DiffRasterizer Interface is part of the GSCache project, an unofficial implementation of 3D Gaussian Splatting [S1, unofficial]. It was developed to provide a flexible and efficient interface for differentiable rendering of Gaussian splats [S1, unofficial].

## Sources
[S1] DBauer15/gscache, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
