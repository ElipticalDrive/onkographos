# Atlas Finetuning with Sparse Adam

Also called: Atlas Finetune Procedure, Atlas Finetuning, Atlas Optimization, Sparse Adam Finetuning, Baked + finetune, Texel finetune

## Overview
Atlas Finetuning with Sparse Adam is a post-baking optimization step that recovers rendering quality lost during the quantization, pruning, and compression of a neural radiance field into a texture atlas [S1, unofficial]. It treats the baked atlas as an initialization and refines only the texel colors for 3,000 iterations, freezing geometry and view-dependent color parameters [S1, unofficial]. This method distinguishes itself from continuing neural teacher training by eliminating hash collisions inherent in the original hash grid and leveraging a larger parameter count (5–29× more parameters) to exceed the teacher's performance [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** A baked, BC7-compressed texture atlas (dequantized to FP16), frozen geometry (surfel positions/scales), frozen view-dependent color coefficients, and training images [S1, unofficial].
- **Output:** An optimized atlas texture where only the texel RGB values have been updated [S1, unofficial].

**Data Structures**
- **Atlas Texture:** A 2D texture storing FP16 values. The layout is determined by the bake and texel allocation stage, which assigns specific tile sizes to surfels based on rate-distortion trade-offs [S1, unofficial].
- **Optimizer State:** Lazy sparse Adam state maintained only for texel rows that receive gradients, storing first and second moment estimates in BF16 precision to reduce memory usage [S1, unofficial].

**Parameters**
- **Iterations:** 3,000 [S1, unofficial].
- **Learning Rate:** $1 \times 10^{-3}$ [S1, unofficial].
- **Optimizer Hyperparameters:** $\beta_1 = 0.9$, $\beta_2 = 0.999$ [S1, unofficial].
- **Loss Weights:** L1 loss weight 1.0, D-SSIM loss weight 0.2 [S1, unofficial].
- **Loss Function:** $L = L1 + 0.2(1 - SSIM)$ [S1, unofficial].

**Algorithm Steps**
1. **Dequantization:** Convert the baked BC7-compressed atlas into a trainable FP16 texture [S1, unofficial].
2. **Freeze Non-Texel Parameters:** Explicitly freeze the geometry (surfel positions and scales) and view-dependent color coefficients [S1, unofficial].
3. **Differentiable Rasterization Setup:** Configure the renderer to use the exact inference atlas fetch logic: Forward pass performs rect transform, coordinate clamping, and bilinear interpolation to fetch texel colors [S1, unofficial]. Backward pass scatters the residual gradient from the bilinear read into the four contributing texels [S1, unofficial].
4. **Sparse Optimization Loop:** For each iteration from 1 to 3,000: Render the scene using the current atlas and frozen parameters to generate a predicted image [S1, unofficial]. Compute the loss $L = L1 + 0.2(1 - SSIM)$ against the ground truth training image [S1, unofficial]. Compute gradients with respect to the atlas texels [S1, unofficial]. Identify the set of texel rows touched by the current view (gradient sparsity is typically 5–7% of rows) [S1, unofficial]. Update only the touched texels using the Sparse Adam optimizer with BF16 moment storage [S1, unofficial]. Continue the learning rate schedule from the teacher training phase; do not restart [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atlas_finetune(atlas_texture, frozen_geometry, frozen_view_color, training_dataset, iterations=3000, lr=1e-3, beta1=0.9, beta2=0.999, eps=1e-8):
    optimizer_state = {}
    for step in range(iterations):
        gt_image, camera = next(training_dataset)
        predicted_image = bilinear_atlas_fetch(atlas_texture, frozen_geometry, frozen_view_color, camera)
        loss = mean_absolute_error(predicted_image, gt_image) + 0.2 * (1.0 - structural_similarity(predicted_image, gt_image))
        gradients = compute_gradients(loss, atlas_texture)
        touched_rows = [r for r in range(atlas_texture.shape[0]) if any(gradients[r] != 0)]
        for row_idx in touched_rows:
            if row_idx not in optimizer_state:
                optimizer_state[row_idx] = {'mean': zeros_like(atlas_texture[row_idx], dtype='bf16'), 'var': zeros_like(atlas_texture[row_idx], dtype='bf16')}
            state = optimizer_state[row_idx]
            grad_row = gradients[row_idx]
            state['mean'] = bf16(beta1 * state['mean'] + (1 - beta1) * grad_row)
            state['var'] = bf16(beta2 * state['var'] + (1 - beta2) * (grad_row ** 2))
            mean_hat = state['mean'] / (1 - beta1 ** (step + 1))
            var_hat = state['var'] / (1 - beta2 ** (step + 1))
            atlas_texture[row_idx] = atlas_texture[row_idx] - lr * mean_hat / (sqrt(var_hat) + eps)
    return atlas_texture

def bilinear_atlas_fetch(texture, geometry, view_color, camera):
    uv_coords = map_to_uv(geometry, camera)
    uv_coords = clamp(uv_coords, 0.0, 1.0)
    return bilinear_interpolate(texture, uv_coords)

def compute_gradients(loss, texture):
    return scatter_bilinear_gradients(loss, texture)

def mean_absolute_error(pred, gt):
    return sum(abs(pred - gt)) / len(pred)

def structural_similarity(pred, gt):
    return 1.0 - sum(abs(pred - gt)) / (len(pred) * 3)

def zeros_like(tensor, dtype):
    return tensor * 0

def bf16(tensor):
    return tensor.to('bf16')

def sqrt(tensor):
    return tensor ** 0.5
```

**Edge Cases and Pitfalls**
- **Unfreezing Geometry:** Unfreezing geometry during finetuning degrades quality, costing 0.18 dB PSNR and 0.009 LPIPS, even at 0.1× the learning rate [S1, unofficial]. This is because moving surfels invalidates the binding between texels and scene positions established at bake time [S1, unofficial].
- **Unfreezing View-Dependent Color:** Jointly training view-dependent color with the atlas costs 2.7 dB PSNR [S1, unofficial]. Unfreezing view-dependent color alone is neutral (0.01 dB tie), but joint training is ill-posed because the view-dependent term can cancel against the residual [S1, unofficial].
- **Precision:** Optimizer moments are stored in BF16 to halve memory usage [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The finetuning stage costs a few minutes per scene [S1, unofficial]. The sparse update ensures that only 5–7% of texel rows are processed per iteration, reducing computational load compared to dense updates [S1, unofficial].
- **Memory Complexity:** Memory usage is reduced by half compared to dense Adam due to BF16 moment storage and lazy allocation for touched rows only [S1, unofficial].
- **Performance:** The finetuned atlas surpasses the neural teacher model by +0.24 to +0.47 dB PSNR and improves LPIPS by 0.025–0.026 on all benchmarks [S1, unofficial]. It also improves SSIM on two of three datasets [S1, unofficial].
- **Scaling:** The method scales with the number of surfels and the sparsity of the gradient. The sparse update avoids the "slow random walk" that degrades the atlas when untouched texels are updated by dense optimizers [S1, unofficial].
- **Strengths:** Eliminates hash collisions present in the original hash grid; leverages higher parameter count (13× average) for better representation; low memory overhead; fast convergence [S1, unofficial].
- **Limitations:** Requires a prior bake step; only refines color, not geometry or view-dependence; sensitive to learning rate if geometry is accidentally unfrozen [S1, unofficial].

## Background
Atlas Finetuning with Sparse Adam was introduced in the paper "Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting" [S1, unofficial]. It addresses the quality loss inherent in baking neural radiance fields into texture atlases due to quantization, Nyquist clamping, and pruning [S1, unofficial]. The method was developed to allow real-time rendering of high-quality radiance fields by offloading the heavy neural network evaluation to a texture lookup, while recovering the lost fidelity through targeted optimization [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
