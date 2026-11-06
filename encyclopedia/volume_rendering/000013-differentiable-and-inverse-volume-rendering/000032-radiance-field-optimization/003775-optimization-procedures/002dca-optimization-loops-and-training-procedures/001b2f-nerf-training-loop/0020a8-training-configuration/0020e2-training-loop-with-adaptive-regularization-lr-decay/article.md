# Training Loop with Adaptive Regularization & LR Decay

Also called: NeRF Training, Training Loop with Exponential LR Decay

## Overview
This method specifies the iterative optimization procedure for training neural radiance fields, integrating adaptive regularization terms and exponential learning rate decay within the training loop [S1, unofficial][S2, unofficial]. It combines a reconstruction loss with structural priors (orthogonality, L1 density, and total variation) while gradually reducing the optimization step size to stabilize convergence [S2, unofficial].

## How it works
**Inputs:**
- `rays_train`: Ray origins and directions sampled from the dataset [S2, unofficial].
- `rgb_train`: Ground-truth RGB values for the sampled rays [S2, unofficial].
- `tensorf`: Neural field model containing parameters to optimize [S2, unofficial].
- `optimizer`: Adam optimizer instance initialized with `learning_rate` and `weight_decay` [S1, unofficial].
- `args`: Configuration object containing hyperparameters (see Parameters) [S1, unofficial][S2, unofficial].

**Outputs:**
- Updated `tensorf` parameters [S2, unofficial].
- Updated `optimizer` state [S2, unofficial].
- Scalar `total_loss` and PSNR metric for tracking [S2, unofficial].

**Data Structures:**
- Ray tensors are fetched per batch using `trainingSampler.nextids()` [S2, unofficial].
- Rendering outputs include `rgb_map`, `alphas_map`, `depth_map`, `weights`, and `uncertainty` [S2, unofficial].
- Tensor shapes are not given in the sources.

**Parameters:**
- `learning_rate`: Initial learning rate for the Adam optimizer [S1, unofficial].
- `weight_decay`: Weight decay coefficient for the Adam optimizer [S1, unofficial].
- `decay_rate` / `decay_steps`: Exponential decay parameters for the learning rate [S1, unofficial].
- `Ortho_reg_weight`: Initial orthogonality regularization strength (`args.Ortho_weight`) [S2, unofficial].
- `L1_reg_weight`: Initial L1 density regularization strength (`args.L1_weight_inital`), switching to `args.L1_weight_rest` after the first alpha mask update iteration [S2, unofficial].
- `TV_weight_density` / `TV_weight_app`: Initial total variation weights for density and appearance fields (`args.TV_weight_density`, `args.TV_weight_app`) [S2, unofficial].
- `lr_factor`: Multiplicative factor applied to learning rates and TV weights per iteration (typically $< 1$) [S2, unofficial].
- `batch_size`: Chunk size for rendering and ray sampling (`args.batch_size`) [S2, unofficial].
- `nSamples`: Number of samples per ray, dynamically updated during resolution upscaling [S2, unofficial].
- `white_bg`: Boolean flag for background color assumption during compositing [S2, unofficial].
- `ndc_ray`: Boolean flag to switch between standard and NDC ray coordinates [S2, unofficial].
- `crop_steps`: Number of steps to use center-cropped data at the beginning of training [S1, unofficial].

**Steps:**
1. Initialize the Adam optimizer with `learning_rate` and `weight_decay` [S1, unofficial].
2. Sample ray indices using `trainingSampler.nextids()` and fetch `rays_train` and `rgb_train` from the dataset [S2, unofficial].
3. Render rays through the neural field model to obtain `rgb_map`, `alphas_map`, `depth_map`, `weights`, and `uncertainty` [S2, unofficial].
4. Compute the reconstruction loss as the mean squared error: $L_{rec} = \frac{1}{B} \sum_{i=1}^{B} \| \text{rgb\_map}_i - \text{rgb\_train}_i \|^2$ [S2, unofficial].
5. Accumulate regularization losses:
   - If `Ortho_reg_weight > 0`, compute `loss_reg = tensorf.vector_comp_diffs()` and add `Ortho_reg_weight * loss_reg` to `total_loss` [S2, unofficial].
   - If `L1_reg_weight > 0`, compute `loss_reg_L1 = tensorf.density_L1()` and add `L1_reg_weight * loss_reg_L1` to `total_loss` [S2, unofficial].
   - If `TV_weight_density > 0`, decay it by `lr_factor`, compute `loss_tv_density = tensorf.TV_loss_density(tvreg) * TV_weight_density`, and add it to `total_loss` [S2, unofficial].
   - If `TV_weight_app > 0`, decay it by `lr_factor`, compute `loss_tv_app = tensorf.TV_loss_app(tvreg) * TV_weight_app`, and add it to `total_loss` [S2, unofficial].
6. Zero gradients, backpropagate `total_loss`, and step the optimizer [S2, unofficial].
7. Update the learning rate for all parameter groups: `param_group['lr'] = param_group['lr'] * lr_factor` [S2, unofficial].
8. Compute PSNR for tracking: $\text{PSNR} = -10 \cdot \log_{10}(L_{rec})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_rays(rays, model, batch_size, n_samples, white_bg, ndc_ray):
    """Standard volumetric integration step [G]. Returns rgb_map, alphas_map, depth_map, weights, uncertainty."""
    pass

def train_step(rays_train, rgb_train, tensorf, optimizer, args):
    # 1. Render rays
    rgb_map, alphas_map, depth_map, weights, uncertainty = render_rays(
        rays_train, tensorf, args.batch_size, args.nSamples, args.white_bg, args.ndc_ray
    )

    # 2. Compute Reconstruction Loss (MSE)
    loss_rec = np.mean((rgb_map - rgb_train) ** 2)
    total_loss = loss_rec

    # 3. Orthogonality Regularization
    if args.Ortho_reg_weight > 0:
        loss_reg = tensorf.vector_comp_diffs()
        total_loss += args.Ortho_reg_weight * loss_reg

    # 4. L1 Density Regularization
    if args.L1_reg_weight > 0:
        loss_reg_L1 = tensorf.density_L1()
        total_loss += args.L1_reg_weight * loss_reg_L1

    # 5. Total Variation Regularization (Density)
    if args.TV_weight_density > 0:
        args.TV_weight_density *= args.lr_factor
        loss_tv_density = tensorf.TV_loss_density(args.tvreg)
        total_loss += args.TV_weight_density * loss_tv_density

    # 6. Total Variation Regularization (Appearance)
    if args.TV_weight_app > 0:
        args.TV_weight_app *= args.lr_factor
        loss_tv_app = tensorf.TV_loss_app(args.tvreg)
        total_loss += args.TV_weight_app * loss_tv_app

    # 7. Optimization Step
    optimizer.zero_grad()
    # Backpropagate total_loss and step optimizer [G]
    optimizer.step()

    # 8. Learning Rate Decay
    for param_group in optimizer.param_groups:
        param_group['lr'] *= args.lr_factor

    # 9. Compute PSNR for tracking
    psnr = -10.0 * np.log10(loss_rec)

    return total_loss, psnr
```

**Edge Cases and Pitfalls:**
- **TV Weight Vanishing**: If `lr_factor < 1`, TV weights decay multiplicatively every iteration, causing regularization strength to vanish rapidly over training [S2, unofficial].
- **L1 Weight Switching**: `L1_reg_weight` switches from `args.L1_weight_inital` to `args.L1_weight_rest` only after the first alpha mask update iteration, potentially causing inconsistent regularization early on [S2, unofficial].
- **Validation Set Size**: The validation set size is capped at $1024 \times 100$ rays. If the dataset is larger, indices are sampled linearly [S1, unofficial].
- **Precision**: Ensure `loss_rec` is not zero to avoid `log(0)` in PSNR calculation. Add a small epsilon if necessary [G].

## Subtopics

### [Training Loop Setup](1522-training-loop-setup.md)
Defines the initial configuration and data loading pipeline.

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Details the rendering process used to generate `rgb_map`.

### [Training Configuration](2332-training-configuration.md)
Specifies hyperparameters like `batch_size` and `nSamples`.

### [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md)
Explains how `nSamples` and resolution are updated.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
An alternative training approach for unbounded scenes.

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Specifics of Mip-NeRF 360's training loop.

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
A multi-resolution training strategy.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Scene representation details.

### [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)
Extension for dynamic scenes.

### [NeuS](4951-neus.md)
Neural Implicit Surfaces, which may use different regularization.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with batch size and number of samples. Resolution upscaling dynamically updates `nSamples` and model complexity [S2, unofficial].
- **Strengths**: Adaptive regularization helps stabilize training of hash-based or high-frequency fields. Exponential LR decay reduces optimization step size over time [S1, unofficial][S2, unofficial].
- **Limitations**: TV weight decay can remove beneficial smoothing too quickly. L1 weight inconsistency may affect early convergence [S2, unofficial].
- **When to Use**: Standard for training neural radiance fields with hash grids or MLPs requiring structural regularization [S2, unofficial].
- **When Not to Use**: For simple MLPs without hash grids, TV regularization may be unnecessary [S2, unofficial].

## Background
This training loop structure is derived from implementations of neural radiance fields, particularly those using hash grids and adaptive regularization [S2, unofficial]. The specific combination of orthogonality, L1 density, and total variation regularization with exponential LR decay is characteristic of recent efficient NeRF implementations [S2, unofficial]. The PSNR metric is calculated from the mean reconstruction loss to track training progress [S1, unofficial].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
