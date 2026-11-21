# Joint Finetuning for Vector-Quantized Radiance Fields

Also called: VQ-DVGO, VQ-Plenoxels, VQ-TensoRF finetuning

## Overview
Joint finetuning is a post-compression optimization procedure applied to vector-quantized radiance fields to recover rendering quality lost during voxel pruning and codebook initialization [S1]. It adapts gradient-based optimization to the constraints of vector-quantized structures, building on [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md).

## How it works
The method recovers rendering quality in compressed radiance field representations through a sequence of pruning, initialization, and joint optimization [S1].

### Inputs and Outputs
- **Inputs**: A pre-trained radiance field model (DVGO, Plenoxels, or TensoRF), training images, camera poses, and a pruning quantile for cumulative score rates [S1].
- **Outputs**: A vector-quantized radiance field model with recovered rendering quality and up to 100× lower storage cost compared to uncompressed models [S1].

### Data Structures
The data structures depend on the specific variant being finetuned [S1]:
1. **DVGO Model**: Density grid, feature grid, and a shallow MLP for color estimation [S1].
2. **Plenoxels Model**: Density grid and a feature grid filled with spherical harmonic coefficients [S1].
3. **TensoRF Model**: Virtual grid for importance scoring, with points projected onto density and feature tri-planes by aggregating along each axis [S1].

### Steps
1. **Pruning**: Prune less important voxels according to the quantile of the cumulative score rate [S1].
2. **Codebook Initialization**: Use only the pruned voxels for codebook initialization [S1].
3. **Joint Finetuning**: Begin joint finetuning with method-specific learning rates and exponential decay schedules [S1].

### Parameters and Schedules
The specific parameters for each variant are as follows [S1]:

**DVGO (VQ-DVGO)**
- Density grid learning rate: $8 \times 10^{-2}$ [S1].
- Feature grid and shallow MLP learning rate: $1.6 \times 10^{-3}$ [S1].
- Decay schedule: Exponential decay, scaling learning rates by $0.3$ every $10,000$ iterations [S1].
- Duration: $10,000$ iterations [S1].
- Batch size: $8,192$ rays per iteration [S1].

**Plenoxels (VQ-Plenoxels)**
- Duration: $25,600$ iterations, equivalent to two epochs in the original Plenoxels training [S1].
- Scheduler reset: Reset to the sixth epoch of the original setting [S1].

**TensoRF (VQ-TensoRF)**
- Learning rate: $5 \times 10^{-3}$ for all density tri-planes, feature tri-planes, and the shallow MLP [S1].
- Decay schedule: Exponential decay, scaling learning rate by $0.3$ every $30,000$ iterations [S1].
- Duration: $10,000$ iterations [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joint_finetune(model, training_data, config):
    # 1. Pruning
    scores = compute_voxel_importance(model)
    threshold = sorted(scores)[int(config['pruning_quantile'] * len(scores))]
    model.prune_voxels_below(threshold)

    # 2. Codebook Initialization
    model.initialize_codebook_from_pruned_voxels()

    # 3. Joint Finetuning
    num_iterations = config['num_iterations']
    lr_density = config['lr_density']
    lr_feature = config['lr_feature']
    lr_mlp = config['lr_mlp']
    decay_factor = config['decay_factor']
    decay_interval = config['decay_interval']
    rays_per_iter = config['rays_per_iter']

    current_lrs = {'density': lr_density, 'feature': lr_feature, 'mlp': lr_mlp}

    for iteration in range(num_iterations):
        # Sample rays
        rays = sample_rays_from_dataset(training_data, rays_per_iter)
        
        # Forward pass
        rendered = model.forward(rays)
        ground_truth = rays.gt_images
        
        # Compute loss
        loss = sum((r - g) ** 2 for r, g in zip(rendered, ground_truth)) / len(rendered)
        
        # Backward pass
        gradients = compute_gradients(loss, model.parameters())
        
        # Update parameters
        for name, param in model.parameters().items():
            param -= current_lrs[name] * gradients[name]
            
        # Learning rate decay
        if config['variant'] == 'plenoxels':
            # Scheduler reset to sixth epoch is handled by external configuration
            pass
        else:
            if (iteration + 1) % decay_interval == 0:
                for name in current_lrs:
                    current_lrs[name] *= decay_factor

    return model
```

### Edge Cases and Pitfalls
- **Pruning Threshold**: The quantile of the cumulative score rate must be chosen carefully to balance compression ratio and quality recovery. [Not given in the sources.]
- **Scheduler Reset**: For Plenoxels, the learning rate scheduler is reset to the sixth epoch of the original setting. [S1]
- **Precision and Boundaries**: Standard differentiable rendering precision limits apply during gradient computation through the quantized grids. [G]

## Subtopics

### [Atlas Finetuning with Sparse Adam](2711-atlas-finetuning-with-sparse-adam.md)
A post-baking optimization step that refines only texel colors for 3,000 iterations while freezing geometry and view-dependent color parameters. It distinguishes itself from continuing neural teacher training by eliminating hash collisions inherent in the original hash grid and leveraging a larger parameter count (5–29× more) to exceed the teacher's performance. Choose this when recovering rendering quality after compression into a texture atlas is required.

## Operating characteristics
- **Storage Cost**: Achieves up to 100× lower storage cost compared to uncompressed models while preserving rendering quality [S1].
- **Performance**: Effectively preserves rendering quality, including fine details, compared to the uncompressed model [S1].
- **Computational Cost**: Finetuning duration varies by variant: 10,000 iterations for DVGO and TensoRF, and 25,600 iterations for Plenoxels [S1].
- **Scalability**: Not given in the sources.
- **Strengths**: Significant compression with minimal quality loss; recovers quality lost during quantization.
- **Limitations**: Performance depends on the underlying grid structure (e.g., hash grids for DVGO) and the chosen pruning quantile. [G]

## Background
Joint finetuning for vector-quantized radiance fields was introduced in "Compressing Volumetric Radiance Fields to 1 MB" [S1]. The method addresses the challenge of compressing neural radiance fields while maintaining high rendering quality by applying targeted gradient-based optimization after quantization and pruning.

## Sources
[S1] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
