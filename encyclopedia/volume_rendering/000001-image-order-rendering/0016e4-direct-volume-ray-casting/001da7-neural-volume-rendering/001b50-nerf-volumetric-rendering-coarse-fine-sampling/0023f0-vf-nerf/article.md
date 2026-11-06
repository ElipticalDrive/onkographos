# VF-NeRF
Also called: Vector Field NeRF

## Overview
VF-NeRF is a neural volume rendering method that represents indoor scenes using learned vector fields rather than standard scalar density fields, trained from multi-view posed images and their associated monocular depth maps [S1, unofficial]. It distinguishes itself from standard NeRF by modeling scene geometry and appearance through a vector field representation integrated along rays, leveraging structural priors from depth supervision [S1, unofficial].

## How it works
**Inputs:** Multi-view posed RGB images and corresponding monocular depth maps [S1, unofficial].
**Outputs:** Not given in the sources [S1, unofficial].

**Data Structures:**
The model state consists of the following components [S1, unofficial]:
1. `vector_field_network`: A neural network (instance of `VectorFieldNetwork`) that outputs the local vector field at a 3D coordinate [S1, unofficial].
2. `rendering_network`: A neural network (instance of `RenderingNetwork`) that maps intermediate features to RGB color and opacity [S1, unofficial].
3. `density`: A density function (instance of `LaplaceDensity`) used to convert vector field magnitudes or auxiliary outputs into volumetric density $\sigma$ [S1, unofficial].
4. `fine_vector_field_network`: Optional secondary vector field network used during fine sampling; defaults to sharing the instance of `vector_field_network` if not explicitly configured [S1, unofficial].
5. `ray_sampler`: Primary sampler (instance of `UniformSampler`) generating coarse samples [S1, unofficial].
6. `fine_sampler`: Optional sampler (instance of `RangeFineSampler`) generating fine samples based on coarse results [S1, unofficial].

**Parameters:**
* `n_samples`: Number of coarse samples per ray [S1, unofficial].
* `n_importance`: Number of additional fine samples per ray [S1, unofficial].
* `near`, `far`: Near and far clipping planes for ray sampling [S1, unofficial].
* `perturb`: Boolean flag for jittering sample positions [S1, unofficial].
* `range`: Range parameter for the fine sampler [S1, unofficial].
* `max_samples`: Maximum total samples allowed [S1, unofficial].
* `beta_bounds`: Bounds for the Laplace density beta parameter, default `[1e-4, 1e9]` [S1, unofficial].
* `mean_bounds`: Bounds for the Laplace density mean parameter, default `[0.6, 1.0]` [S1, unofficial].
* `scale_min`: Minimum scale for density, default `0.1` [S1, unofficial].
* `lr`: Learning rate for the optimizer [S1, unofficial].
* `weight_decay`: Weight decay for the optimizer [S1, unofficial].
* `lr_decay_factor`, `lr_decay_steps`: Parameters for the learning rate scheduler [S1, unofficial].

**Algorithm:**
1. **Initialization:**
   * Initialize `vector_field_network` and `rendering_network` using configuration objects `vf_net_config` and `rendering_net_config` respectively [S1, unofficial].
   * Initialize `density` as `LaplaceDensity` with parameters from `density_config.todict()` [S1, unofficial].
   * If fine sampling is enabled (`config.ray_sampler_config.fine_sampling()`), initialize `fine_vector_field_network`. If not explicitly configured, set `fine_vector_field_network = vector_field_network` [S1, unofficial].
   * Create `ray_sampler` (`UniformSampler`) with `n_samples`, `near`, `far`, and `perturb` (inverted from config) [S1, unofficial].
   * If fine sampling is enabled, create `fine_sampler` (`RangeFineSampler`) with `n_importance`, `near`, `far`, `perturb` (inverted), `range` (`config.ray_sampler_config.fine_range`), and `max_samples` (`config.ray_sampler_config.max_samples`) [S1, unofficial].
   * If `config.cos_sim_weights_anneal` is not 'none', initialize `LinearAnnealing` with the number of weight steps, duration `anneal_end - anneal_start`, and soft mode determined by `cos_sim_weights_anneal == 'soft'` [S1, unofficial].
   * Initialize optimizer with `lr` and `weight_decay` [S1, unofficial].
   * Initialize scheduler with decay factor `lr_decay_factor ** (1. / lr_decay_steps)` [S1, unofficial].
   * If `num_gpus > 1`, wrap networks in `nn.DataParallel` [S1, unofficial].
   * Move all networks and samplers to the specified device [S1, unofficial].
2. **Training Loop:** Not given in the sources [S1, unofficial]. Standard differentiable volume rendering integration and optimization steps apply [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def initialize_vf_nerf(config):
    # Initialize vector field network
    vf_net = VectorFieldNetwork(config.vf_net_config)

    # Initialize rendering network
    render_net = RenderingNetwork(config.rendering_net_config)

    # Initialize density function
    density_fn = LaplaceDensity(**config.density_config.todict())

    # Initialize fine vector field network if fine sampling is enabled
    if config.ray_sampler_config.fine_sampling():
        fine_vf_net = VectorFieldNetwork(config.vf_net_config)
    else:
        fine_vf_net = vf_net

    # Create primary ray sampler
    ray_sampler = UniformSampler(
        n_samples=config.ray_sampler_config.n_samples,
        near=config.ray_sampler_config.near,
        far=config.ray_sampler_config.far,
        perturb=not config.ray_sampler_config.perturb
    )

    # Create fine sampler if enabled
    fine_sampler = None
    if config.ray_sampler_config.fine_sampling():
        fine_sampler = RangeFineSampler(
            n_importance=config.ray_sampler_config.n_importance,
            near=config.ray_sampler_config.near,
            far=config.ray_sampler_config.far,
            perturb=not config.ray_sampler_config.perturb,
            range=config.ray_sampler_config.fine_range,
            max_samples=config.ray_sampler_config.max_samples
        )

    # Initialize annealer if configured
    annealer = None
    if config.cos_sim_weights_anneal != 'none':
        annealer = LinearAnnealing(
            num_steps=config.cos_sim_weights.shape[0],
            duration=config.anneal_end - config.anneal_start,
            soft=(config.cos_sim_weights_anneal == 'soft')
        )

    # Initialize optimizer
    optimizer = torch.optim.Adam(
        parameters=list(vf_net.parameters()) + list(render_net.parameters()) + list(fine_vf_net.parameters()),
        lr=config.scheduler_config.lr,
        weight_decay=config.scheduler_config.weight_decay
    )

    # Initialize learning rate scheduler
    scheduler = torch.optim.lr_scheduler.ExponentialLR(
        optimizer=optimizer,
        gamma=config.scheduler_config.lr_decay_factor ** (1.0 / config.scheduler_config.lr_decay_steps)
    )

    # Wrap in data parallel if multiple GPUs are configured
    if config.cuda_config.num_gpus > 1:
        vf_net = torch.nn.DataParallel(vf_net)
        render_net = torch.nn.DataParallel(render_net)
        density_fn = torch.nn.DataParallel(density_fn)
        fine_vf_net = torch.nn.DataParallel(fine_vf_net)

    # Move to device
    device = config.cuda_config.device
    vf_net = vf_net.to(device)
    render_net = render_net.to(device)
    density_fn = density_fn.to(device)
    fine_vf_net = fine_vf_net.to(device)

    # Training loop, loss computation, and optimization steps are not given in the sources.

    return {
        'vf_net': vf_net, 'render_net': render_net, 'density_fn': density_fn,
        'fine_vf_net': fine_vf_net, 'ray_sampler': ray_sampler, 'fine_sampler': fine_sampler,
        'optimizer': optimizer, 'scheduler': scheduler, 'annealer': annealer
    }
```

**Edge Cases and Pitfalls:**
* **Precision:** The Laplace density parameters `beta` and `mean` have wide bounds (`1e-4` to `1e9` and `0.6` to `1.0` respectively), requiring careful initialization to avoid numerical instability in the exponential term of the opacity calculation [S1, unofficial].
* **Fine Sampling:** If fine sampling is disabled, the `fine_vector_field_network` is not used, and the model relies solely on the coarse network, potentially reducing detail [S1, unofficial].
* **Depth Supervision:** The method relies on monocular depth maps. Inaccurate depth maps can lead to incorrect vector field learning and poor geometry reconstruction [S1, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources [S1, unofficial].
* **Memory Complexity:** Not given in the sources [S1, unofficial].
* **Performance:** Tested on Python 3.9, PyTorch 1.10.0, CUDA 11.3 [S1, unofficial].
* **Strengths:** Leverages monocular depth for better geometry learning in indoor scenes [S1, unofficial].
* **Limitations:** Requires posed images and depth maps [S1, unofficial].
* **When to use:** Indoor scene reconstruction where monocular depth is available and accurate [S1, unofficial].
* **When not to use:** Unposed images or scenes without reliable depth priors [S1, unofficial].

## Background
VF-NeRF was introduced by Albert Gassol et al. as an implementation of a neural volume rendering method using vector fields for indoor scene representation [S1, unofficial]. It builds on the NeRF framework but replaces the scalar density field with a vector field, aiming to improve geometry learning through depth supervision [S1, unofficial].

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
