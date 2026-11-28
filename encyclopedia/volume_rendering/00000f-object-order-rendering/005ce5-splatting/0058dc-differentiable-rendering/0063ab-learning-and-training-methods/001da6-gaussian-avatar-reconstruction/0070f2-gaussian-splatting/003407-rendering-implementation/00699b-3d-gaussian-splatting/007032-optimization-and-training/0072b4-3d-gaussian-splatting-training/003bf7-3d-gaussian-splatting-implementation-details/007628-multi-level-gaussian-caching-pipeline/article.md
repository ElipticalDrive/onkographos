# Multi-Level Gaussian Caching Pipeline

Also called: gscache

## Overview
The Multi-Level Gaussian Caching Pipeline (gscache) is a hierarchical optimization framework for 3D Gaussian Splatting that maintains multiple independent levels of Gaussian splats, each rasterized and optimized against cached reference radiance samples [S1, unofficial]. It enables progressive refinement or accelerated training by decoupling the optimization of different spatial or frequency details into separate pyramid levels [S1, unofficial]. Unlike standard single-level training, it rasterizes each level against a cached target rather than the ground truth image directly, allowing lower levels to converge on coarse structures while higher levels refine details [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Initial 3D Gaussian point cloud (position, covariance, opacity, SH coefficients).
*   Camera poses and intrinsics.
*   Training images (used to populate initial cache or as reference for loss, though the core loop uses cached radiance).
*   Configuration parameters (see Parameters).

**Outputs:**
*   Optimized Gaussian models for each level.
*   Final rendered images from the highest level or composite.

### Data Structures

**`gscache_config`**
Holds global configuration.
*   `sh_degree` (int): Spherical harmonics degree.
*   `n_levels` (int): Number of pyramid levels. Default: 3. Range: 1 to `MAX_NUM_CACHE_LEVELS` (16).
*   `subsample_levels` (bool): Whether to subsample levels.
*   `scene_extent` (float): Extent of the scene.
*   `size_threshold` (int): Threshold for Gaussian size.
*   `gradient_threshold` (float): Minimum gradient magnitude for densification. Default: 0.0002.
*   `opacity_threshold` (float): Minimum opacity to retain. Default: 0.005.
*   `densification_interval` (int): Iterations between densification. Default: 100.
*   `max_densification_iteration` (int): Max densification iterations. Default: 1000.
*   `opacity_reset_interval` (int): Iterations between opacity resets. Default: 250.
*   `max_opacity_reset_iteration` (int): Max opacity reset iterations. Default: $2^{20}$ (1,048,576).
*   `background` (glm::vec3): Background color.
*   `loss_fn` (enum): Loss function type (`GSCACHE_LOSS_FUNCTION_MSE` or `GSCACHE_LOSS_FUNCTION_HDR`).

**`gscache_context`**
Host-side context managing the pipeline state.
*   `version` (int): Context version.
*   `config` (gscache_config): Configuration.
*   `iterations` (std::vector<size_t>): Iteration count per level.
*   `models` (std::vector<GaussianModel*>): Pointer to GaussianModel for each level.
*   `forward_results` (std::vector<DiffRasterizerForwardResult*>): Forward rasterization results per level.
*   `backward_results` (std::vector<DiffRasterizerBackwardResult*>): Backward rasterization results per level.
*   `camera` (Camera*): Current camera.
*   `rasterizer` (DiffRasterizer*): Rasterizer instance.
*   `sample_factor_c` (float): Scaling factor for sampling strategy.
*   `device_context` (gscache_device_context_t): Pointer to device-side context.
*   `device_radiance_samples` (std::vector<torch::Tensor>): Tensor storing cached radiance samples per level.
*   `device_betas` (std::shared_ptr<torch::Tensor>): Tensor storing beta values for importance sampling.

**`gscache_device_context`**
Device-side (CUDA) context for fast access.
*   `n_levels` (int): Number of levels.
*   `height`, `width` (size_t): Image dimensions.
*   `sample_factor_c` (float): Sampling factor.
*   `cache_levels[MAX_NUM_CACHE_LEVELS]` (gscache_vec3f*): Pointers to cached radiance buffers.
*   `radiance_samples[MAX_NUM_CACHE_LEVELS]` (gscache_vec3f*): Pointers to current radiance samples.
*   `betas` (float*): Pointer to beta values.

### Parameters
*   `n_levels`: Default 3.
*   `gradient_threshold`: Default 0.0002.
*   `opacity_threshold`: Default 0.005.
*   `densification_interval`: Default 100.
*   `max_densification_iteration`: Default 1000.
*   `opacity_reset_interval`: Default 250.
*   `max_opacity_reset_iteration`: Default 1,048,576.
*   `sample_factor_c`: Updated via `GSCACHE_SAMPLE_FACTOR_C` option.

### Steps

1.  **Initialize Context**:
    *   Allocate `gscache_context`.
    *   Set version to 0, copy config.
    *   Loop `i` from 0 to `n_levels - 1`:
        *   Push 0 to `iterations`.
        *   Create new `GaussianModel` with `sh_degree`.
        *   Create new `DiffRasterizerForwardResult`.
        *   Create new `DiffRasterizerBackwardResult`.
        *   Create empty `torch::Tensor` for `device_radiance_samples[i]`.
    *   Create shared `torch::Tensor` for `device_betas`.
    *   Create `DiffRasterizer` with background color.
    *   Create `Camera`.

2.  **Update Device Context**:
    *   If `device_context` exists, `cudaFree` it.
    *   Populate local `device_context` struct:
        *   `radiance_samples[i]` = `context->device_radiance_samples[i].data_ptr<float>()`.
        *   `cache_levels[i]` = `context->forward_results[i]->framebuffer.getRGBPtr()`.
        *   `width`, `height` from framebuffer.
        *   `betas` = `context->device_betas->data_ptr<float>()`.
        *   `n_levels` = `config->n_levels`.
        *   `sample_factor_c` = `config->sample_factor_c`.
    *   `cudaMalloc` `device_context_ptr`.
    *   `cudaMemcpy` `device_context_ptr` from host struct.
    *   Assign to `context->device_context`.

3.  **Rasterize Level** (for each level `l`):
    *   If `l >= n_levels`, return.
    *   Call `context->rasterizer->forward(*context->models[l], *context->camera, *context->forward_results[l])`.
    *   Fill `device_betas` with 1.0.

4.  **Update Level** (for each level `l`):
    *   If `l >= n_levels`, return.
    *   Set `expected` = `device_radiance_samples[l]`.
    *   Compute gradient:
        *   If `loss_fn == GSCACHE_LOSS_FUNCTION_MSE`: `gradient = util::mseLossGradient(framebuffer.getRGB(), expected)`.
        *   If `loss_fn == GSCACHE_LOSS_FUNCTION_HDR`: `gradient = util::hdrLossGradient(framebuffer.getRGB(), expected)`.
    *   Call `context->rasterizer->backward(*context->models[l], *context->camera, *context->forward_results[l], gradient, *context->backward_results[l])`.
    *   Fill `device_radiance_samples[l]` with 0.

5.  **Optimize Level** (for each level `l`):
    *   If `l >= n_levels`, return.
    *   Increment `context->iterations[l]`.
    *   Compute `factor = 1.0f / (1.0f + std::log(frameid + 1))`.
    *   Call `context->models[l]->updateLearningRate(factor)`.
    *   Call `context->models[l]->updateGradients(*context->backward_results[l])`.
    *   Call `context->models[l]->updateDensificationStats(*context->backward_results[l])`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

class GSCacheContext:
    def __init__(self, config, n_levels, initial_gaussians, camera, rasterizer):
        self.config = config
        self.n_levels = n_levels
        self.iterations = [0] * n_levels
        self.models = []
        self.forward_results = []
        self.backward_results = []
        self.device_radiance_samples = []
        self.device_betas = None
        self.camera = camera
        self.rasterizer = rasterizer
        
        # Initialize models and buffers
        for i in range(n_levels):
            # Create a copy or new instance of GaussianModel
            model = GaussianModel(sh_degree=config.sh_degree)
            # Initialize with initial_gaussians or empty
            self.models.append(model)
            
            self.forward_results.append(DiffRasterizerForwardResult())
            self.backward_results.append(DiffRasterizerBackwardResult())
            
            # Create tensor for radiance samples
            # Assuming framebuffer size is known
            w, h = rasterizer.get_framebuffer_size()
            tensor = torch.zeros((h, w, 3), dtype=torch.float32, device='cuda')
            self.device_radiance_samples.append(tensor)
            
        # Create shared betas tensor
        w, h = rasterizer.get_framebuffer_size()
        self.device_betas = torch.ones((h, w), dtype=torch.float32, device='cuda')

    def update_device_context(self):
        # This step involves CUDA memory management and copying structs
        # In Python-like pseudocode, we simulate the data pointers
        self.device_context = {
            'n_levels': self.n_levels,
            'width': self.rasterizer.get_width(),
            'height': self.rasterizer.get_height(),
            'sample_factor_c': self.config.sample_factor_c,
            'cache_levels': [res.framebuffer.getRGBPtr() for res in self.forward_results],
            'radiance_samples': [t.data_ptr() for t in self.device_radiance_samples],
            'betas': self.device_betas.data_ptr()
        }
        # In real implementation: cudaMalloc, cudaMemcpy, etc.

    def rasterize_level(self, level):
        if level >= self.n_levels:
            return
        model = self.models[level]
        camera = self.camera
        forward_res = self.forward_results[level]
        
        # Forward pass
        self.rasterizer.forward(model, camera, forward_res)
        
        # Reset betas
        self.device_betas.fill_(1.0)

    def update_level(self, level):
        if level >= self.n_levels:
            return
        model = self.models[level]
        camera = self.camera
        forward_res = self.forward_results[level]
        backward_res = self.backward_results[level]
        
        framebuffer_rgb = forward_res.framebuffer.getRGB()
        expected = self.device_radiance_samples[level]
        
        # Compute gradient based on loss function
        if self.config.loss_fn == 'MSE':
            gradient = util.mseLossGradient(framebuffer_rgb, expected)
        elif self.config.loss_fn == 'HDR':
            gradient = util.hdrLossGradient(framebuffer_rgb, expected)
        else:
            raise ValueError("Unknown loss function")
            
        # Backward pass
        self.rasterizer.backward(model, camera, forward_res, gradient, backward_res)
        
        # Reset radiance samples
        self.device_radiance_samples[level].zero_()

    def optimize_level(self, level, frameid):
        if level >= self.n_levels:
            return
        model = self.models[level]
        backward_res = self.backward_results[level]
        
        self.iterations[level] += 1
        
        # Learning rate decay
        factor = 1.0 / (1.0 + math.log(frameid + 1))
        model.updateLearningRate(factor)
        
        # Update gradients and densification stats
        model.updateGradients(backward_res)
        model.updateDensificationStats(backward_res)

    def step(self, frameid):
        for level in range(self.n_levels):
            self.rasterize_level(level)
            self.update_level(level)
            self.optimize_level(level, frameid)
```

### Edge Cases and Pitfalls
*   **Subsampling Randomization**: Subsampling assumes the input point cloud is spatially randomized. If not randomized, subsampling may cluster Gaussians or leave gaps [S1, unofficial]. The code includes a FIXME comment noting this limitation [S1, unofficial].
*   **Precision**: Use of `float` for radiance samples and betas may lead to precision issues in HDR scenes. The `HDR` loss function is provided to mitigate this.
*   **Boundary Conditions**: Ensure `level < n_levels` checks are performed before accessing level-specific data structures to avoid out-of-bounds errors.

## Operating characteristics

### Time and Memory Complexity
*   **Time**: $O(N \cdot L)$ per iteration, where $N$ is the number of Gaussians and $L$ is the number of levels. Each level is rasterized and optimized independently.
*   **Memory**: $O(N \cdot L)$ for storing Gaussian parameters per level. Additional memory for `device_radiance_samples` and `device_betas` is $O(W \cdot H \cdot L)$ for image buffers.

### Measured Performance
*   Not given in the sources.

### Scaling
*   Scales linearly with the number of levels and Gaussians.
*   Higher levels may require more Gaussians to capture fine details, increasing memory and compute.

### Strengths and Limitations
*   **Strengths**: Enables progressive refinement, potentially faster convergence for coarse structures, decoupled optimization.
*   **Limitations**: Requires careful tuning of level-specific parameters, potential for artifacts if levels are not well-separated, subsampling artifacts if input is not randomized.

### When to Use
*   Use when training large scenes where coarse-to-fine optimization is beneficial.
*   Use when memory is constrained and lower levels can be optimized with fewer Gaussians.

### When Not to Use
*   Use standard 3DGS if simplicity and single-level optimization are preferred.
*   Avoid if input point cloud is not randomized and subsampling is used.

## Background

### Origin
The Multi-Level Gaussian Caching Pipeline (gscache) is an unofficial implementation based on the 3D Gaussian Splatting framework [S1, unofficial]. It was developed to explore hierarchical optimization strategies for Gaussian Splatting [S1, unofficial].

### History and Development
*   **2024**: Initial release of `gscache` on GitHub by DBauer15 [S1, unofficial].
*   The method builds upon the original 3D Gaussian Splatting paper [S1, unofficial] by introducing a multi-level caching mechanism for radiance samples.

### Where It Is Used
*   Research into hierarchical optimization for 3D Gaussian Splatting.
*   Applications requiring progressive refinement or accelerated training of large scenes.

## Sources

[S1] DBauer15/gscache, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
