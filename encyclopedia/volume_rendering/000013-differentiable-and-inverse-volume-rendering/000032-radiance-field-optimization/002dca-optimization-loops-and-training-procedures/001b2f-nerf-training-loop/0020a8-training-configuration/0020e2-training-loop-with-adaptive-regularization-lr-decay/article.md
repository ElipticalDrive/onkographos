# Training Loop with Adaptive Regularization & LR Decay

Also called: NeRF Training, Training Loop with Exponential LR Decay

## Overview
This method defines the iterative optimization procedure for training neural radiance fields, specifically focusing on the integration of adaptive regularization terms and exponential learning rate decay within the training loop. It solves the problem of stable convergence in high-dimensional implicit surface representations by combining reconstruction loss with structural priors (orthogonality, sparsity, and total variation) while gradually reducing the optimization step size. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by specifying the exact loss composition and parameter update schedule.

## How it works
**Inputs:**
- `rays_train`: Tensor of ray origins and directions.
- `rgb_train`: Tensor of ground-truth RGB values for the rays.
- `tensorf`: The neural field model (e.g., hash grid or MLP) containing parameters to optimize.
- `optimizer`: An Adam or AdamW optimizer instance initialized with `learning_rate` and `weight_decay` [S1, unofficial].
- `args`: Configuration object containing hyperparameters (see Parameters).

**Outputs:**
- Updated `tensorf` parameters.
- Updated `optimizer` state.
- Scalar `total_loss` and PSNR metric for tracking.

**Data Structures:**
- `rays_train`: Shape $(B, 3)$ for origins and $(B, 3)$ for directions, where $B$ is `batch_size` [S2, unofficial].
- `rgb_train`: Shape $(B, 3)$ [S2, unofficial].
- `rgb_map`, `alphas_map`, `depth_map`, `weights`, `uncertainty`: Tensors returned by the renderer, shape dependent on $B$ and sampling strategy [S2, unofficial].

**Parameters:**
- `learning_rate`: Initial learning rate for the Adam optimizer [S1, unofficial].
- `weight_decay`: Weight decay coefficient for the Adam optimizer [S1, unofficial].
- `decay_rate`: Base for exponential decay (typically $e$ or a specific base if implemented as $base^{step}$, but sources imply multiplicative factor per step or exponential function of step) [S1, unofficial]. *Note: Source [S2, unofficial] uses a multiplicative `lr_factor` per iteration, while [S1, unofficial] references an `exponential_lr_decay` function. The implementation below follows the explicit multiplicative update described in [S2, unofficial] which is more granular.*
- `decay_steps`: Number of steps over which decay is compounded [S1, unofficial].
- `Ortho_reg_weight`: Initial orthogonality regularization strength (`args.Ortho_weight`) [S2, unofficial].
- `L1_reg_weight`: Initial L1 density regularization strength (`args.L1_weight_inital`), switching to `args.L1_weight_rest` after the first alpha mask update [S2, unofficial].
- `TV_weight_density`: Initial total variation weight for density field (`args.TV_weight_density`) [S2, unofficial].
- `TV_weight_app`: Initial total variation weight for appearance field (`args.TV_weight_app`) [S2, unofficial].
- `lr_factor`: Multiplicative factor applied to learning rates and TV weights per iteration (typically $< 1$) [S2, unofficial].
- `batch_size`: Chunk size for rendering and ray sampling (`args.batch_size`) [S2, unofficial].
- `nSamples`: Number of samples per ray, dynamically updated during resolution upscaling [S2, unofficial].
- `white_bg`: Boolean flag for background color assumption [S2, unofficial].
- `ndc_ray`: Boolean flag for NDC ray coordinates [S2, unofficial].
- `crop_steps`: Number of steps to use center-cropped data [S1, unofficial].

**Steps:**
1. **Initialization**: Initialize the Adam optimizer with `learning_rate` and `weight_decay` [S1, unofficial].
2. **Ray Sampling**: Sample ray indices using `trainingSampler.nextids()` and fetch corresponding `rays_train` and `rgb_train` from the dataset [S2, unofficial].
3. **Rendering**: Render the rays through the neural field model to obtain `rgb_map`, `alphas_map`, `depth_map`, `weights`, and `uncertainty` [S2, unofficial].
   $$ \text{render\_output} = \text{renderer}(\text{rays\_train}, \text{tensorf}, \text{chunk}=\text{batch\_size}, N\_samples=\text{nSamples}, \text{white\_bg}=\text{white\_bg}, \text{ndc\_ray}=\text{ndc\_ray}) $$
4. **Reconstruction Loss**: Compute the mean squared error (MSE) between the rendered RGB and ground truth [S2, unofficial].
   $$ L_{rec} = \frac{1}{B} \sum_{i=1}^{B} \| \text{rgb\_map}_i - \text{rgb\_train}_i \|^2 $$
5. **Regularization Losses**:
   - **Orthogonality**: If `Ortho_reg_weight > 0`, compute `loss_reg` using `tensorf.vector_comp_diffs()` and add to total loss [S2, unofficial].
   - **L1 Density**: If `L1_reg_weight > 0`, compute `loss_reg_L1` using `tensorf.density_L1()` and add to total loss [S2, unofficial].
   - **Total Variation (Density)**: If `TV_weight_density > 0`, decay the weight by `lr_factor`, compute `loss_tv_density` using `tensorf.TV_loss_density(tvreg)`, and add `TV_weight_density * loss_tv_density` to total loss [S2, unofficial].
   - **Total Variation (Appearance)**: If `TV_weight_app > 0`, decay the weight by `lr_factor`, compute `loss_tv_app` using `tensorf.TV_loss_app(tvreg)`, and add `TV_weight_app * loss_tv_app` to total loss [S2, unofficial].
6. **Total Loss**:
   $$ L_{total} = L_{rec} + L_{ortho} + L_{L1} + L_{TV\_density} + L_{TV\_app} $$
7. **Optimization Step**: Zero gradients, backpropagate `total_loss`, and step the optimizer [S2, unofficial].
8. **Learning Rate Decay**: Update the learning rate for all parameter groups by multiplying by `lr_factor` [S2, unofficial].
   $$ \text{lr}_{new} = \text{lr}_{old} \times \text{lr\_factor} $$
9. **TV Weight Decay**: Multiply `TV_weight_density` and `TV_weight_app` by `lr_factor` [S2, unofficial].
10. **Tracking**: Compute PSNR for monitoring [S1, unofficial].
    $$ \text{PSNR} = -10 \cdot \log_{10}(L_{rec}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_step(rays_train, rgb_train, tensorf, optimizer, args):
    # 1. Render rays
    # Returns: rgb_map, alphas_map, depth_map, weights, uncertainty
    rgb_map, alphas_map, depth_map, weights, uncertainty = renderer(
        rays_train, 
        tensorf, 
        chunk=args.batch_size, 
        N_samples=args.nSamples, 
        white_bg=args.white_bg, 
        ndc_ray=args.ndc_ray, 
        device=args.device, 
        is_train=True
    )
    
    # 2. Compute Reconstruction Loss (MSE)
    loss_rec = torch.mean((rgb_map - rgb_train) ** 2)
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
        # Decay TV weight
        args.TV_weight_density *= args.lr_factor
        loss_tv_density = tensorf.TV_loss_density(args.tvreg)
        total_loss += args.TV_weight_density * loss_tv_density
        
    # 6. Total Variation Regularization (Appearance)
    if args.TV_weight_app > 0:
        # Decay TV weight
        args.TV_weight_app *= args.lr_factor
        loss_tv_app = tensorf.TV_loss_app(args.tvreg)
        total_loss += args.TV_weight_app * loss_tv_app
        
    # 7. Optimization Step
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
    
    # 8. Learning Rate Decay
    for param_group in optimizer.param_groups:
        param_group['lr'] *= args.lr_factor
        
    # 9. Compute PSNR for tracking
    psnr = -10.0 * np.log10(loss_rec.item()) / np.log10(np.e) # Note: np.log10 is preferred for PSNR formula
    # Correct PSNR formula from source [S1, unofficial]: -10. * np.log10(loss)
    psnr = -10.0 * np.log10(loss_rec.item())
    
    return total_loss.item(), psnr
```

*Note on PSNR Formula:* Source [S1, unofficial] specifies `psnr = -10. * np.log10(loss)`. Source [S2, unofficial] code snippet uses `np.log` (natural log) but divides by `np.log(10.0)` implicitly or explicitly in some variants. The standard PSNR formula uses base-10 log. The pseudocode above uses `np.log10` to match the standard definition and the explicit formula in [S1, unofficial].

**Edge Cases and Pitfalls:**
- **TV Weight Vanishing**: If `lr_factor < 1`, TV weights decay multiplicatively every iteration. This causes regularization strength to vanish rapidly over training [S2, unofficial].
- **L1 Weight Switching**: `L1_reg_weight` switches from `args.L1_weight_inital` to `args.L1_weight_rest` only after the first alpha mask update iteration. This can cause inconsistent regularization early in training [S2, unofficial].
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
- **Time Complexity**: Dominated by the rendering step, which is $O(B \cdot N_{samples})$ per batch, where $B$ is batch size and $N_{samples}$ is samples per ray. Backpropagation is proportional to the model size.
- **Memory Complexity**: $O(B \cdot N_{samples})$ for storing intermediate rendering tensors (weights, alphas, etc.) [S2, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with batch size and number of samples. Resolution upscaling (dynamic growth) increases $N_{samples}$ and model complexity [S2, unofficial].
- **Strengths**: Adaptive regularization helps stabilize training of hash-based or high-frequency fields. Exponential LR decay prevents oscillation in later stages.
- **Limitations**: TV weight decay can remove beneficial smoothing too quickly. L1 weight inconsistency may affect early convergence.
- **When to Use**: Standard for training neural radiance fields with hash grids or MLPs requiring structural regularization.
- **When Not to Use**: For simple MLPs without hash grids, TV regularization may be unnecessary. For real-time inference, training loop details are irrelevant.

## Background
This training loop structure is derived from implementations of neural radiance fields, particularly those using hash grids and adaptive regularization. The specific combination of orthogonality, L1 density, and total variation regularization with exponential LR decay is characteristic of recent efficient NeRF implementations [S2, unofficial]. The PSNR metric is standard for evaluating reconstruction quality [S1, unofficial].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
