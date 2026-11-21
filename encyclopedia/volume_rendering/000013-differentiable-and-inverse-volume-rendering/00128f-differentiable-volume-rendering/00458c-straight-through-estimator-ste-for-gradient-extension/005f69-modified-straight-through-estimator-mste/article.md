# Modified Straight-Through Estimator (mSTE)
Also called: mSTE

## Overview
Modified Straight-Through Estimator (mSTE) is a gradient extension technique for differentiable volume rendering that addresses the zero-gradient problem in the backward pass by rescaling the detached codec-induced perturbation using its standard deviation. Unlike the parent method, Straight-Through Estimator (STE) for Gradient Extension, which uses a simple identity surrogate to copy gradients directly to the pre-codec feature plane, mSTE modulates the backward signal based on the observed scale of quantization errors. This approach aims to provide a more informative gradient signal than vanilla STE in high-bitrate regimes, though it is less robust than vanilla STE under strong codec distortion.

## How it works
**Inputs and Outputs**
- **Input**: Pre-codec feature plane $\mathbf{P}$, decoded feature plane $\hat{\mathbf{P}}$ (output of the differentiable codec).
- **Output**: Modified feature plane $\widetilde{\mathbf{P}}$ used for gradient computation in the backward pass.

**Data Structures**
- $\mathbf{P}$: Tensor of shape $(C, H, W)$ representing the pre-quantization features.
- $\hat{\mathbf{P}}$: Tensor of shape $(C, H, W)$ representing the post-quantization features.
- $\epsilon$: Tensor of shape $(C, H, W)$ representing the perturbation.
- $\sigma(\epsilon)$: Scalar or tensor representing the standard deviation of the perturbation.

**Steps**
1. Compute the codec-induced perturbation $\epsilon$ as the difference between the decoded feature plane and the pre-codec feature plane:
   $$ \epsilon = \hat{\mathbf{P}} - \mathbf{P} $$
2. Compute the standard deviation $\sigma(\epsilon)$ of the perturbation tensor $\epsilon$. This is typically calculated over the spatial dimensions and channels, resulting in a scalar value or a per-channel scalar depending on implementation.
3. In the backward pass, compute the modified feature plane $\widetilde{\mathbf{P}}$ using the following formula:
   $$ \widetilde{\mathbf{P}} = \mathbf{P} + \operatorname{sg}(\epsilon) \cdot \frac{\sigma(\epsilon)}{\operatorname{sg}(\sigma(\epsilon))} $$
   where $\operatorname{sg}(\cdot)$ is the stop-gradient operator, which prevents gradients from flowing through its argument.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def mSTE_forward(P, codec_function):
    """
    Forward pass: quantize P using codec_function.
    """
    P_hat = codec_function(P)
    return P_hat

def mSTE_backward(P, P_hat):
    """
    Backward pass: compute modified gradient signal.
    """
    # Step 1: Compute perturbation
    epsilon = P_hat - P
    
    # Step 2: Compute standard deviation of perturbation
    # Calculate std over all dimensions except batch, if applicable.
    # Here we assume P is (C, H, W) or (B, C, H, W).
    # We compute std over spatial and channel dimensions.
    # Using unbiased=False to match typical population std for stability.
    sigma_epsilon = torch.std(epsilon, dim=(1, 2, 3), keepdim=True) if epsilon.dim() == 4 else torch.std(epsilon, dim=(1, 2), keepdim=True)
    
    # Step 3: Compute modified feature plane
    # Apply stop-gradient to epsilon and sigma_epsilon
    sg_epsilon = torch.stop_gradient(epsilon)
    sg_sigma = torch.stop_gradient(sigma_epsilon)
    
    # Avoid division by zero
    sg_sigma = torch.clamp(sg_sigma, min=1e-8)
    
    # Rescale the detached error
    rescaled_error = sg_epsilon * (sigma_epsilon / sg_sigma)
    
    # Add to original P
    P_modified = P + rescaled_error
    
    return P_modified
```

**Parameters**
- Standard deviation calculation: Typically computed over the entire tensor or per-channel. Not given in the sources.
- Epsilon for numerical stability: $10^{-8}$ [G].

**Edge Cases and Pitfalls**
- **Division by Zero**: If $\sigma(\epsilon)$ is zero (e.g., no quantization error), the ratio becomes undefined. A small epsilon value should be added to the denominator [G].
- **Low Bitrate Degradation**: At low bitrates, the perturbation $\epsilon$ reflects non-linear artifacts rather than small local quantization residuals. In this regime, standard-deviation reweighting becomes a poor proxy for true local sensitivity, leading to less stable and less effective optimization compared to vanilla STE [S1].

## Subtopics

### [Standard-Codec-in-the-Loop (SCL)](17889-standard-codec-in-the-loop-scl.md)
A method that integrates codec constraints directly into the loop. Choose when codec artifacts need to be minimized during training.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**:
  - High-rate: PSNR 33.62, SSIM 0.968, LPIPS 0.195, training time 1h 25m [S1].
  - Low-rate: PSNR 30.96, SSIM 0.948, LPIPS 0.368, training time 1h 28m [S1].
- **Scaling**: Not given in the sources.
- **Strengths**: Performs competitively at high bitrate [S1].
- **Limitations**: Degrades noticeably at low bitrate [S1]. Less robust than vanilla STE under strong codec distortion [S1].
- **When to Use**: High-bitrate regimes where codec-induced distortions are small and local quantization residuals are a good proxy for sensitivity [S1].
- **When Not to Use**: Low-bitrate regimes or when strong codec distortion is present, as vanilla STE is more robust [S1].

## Background
Modified Straight-Through Estimator (mSTE) was introduced in the context of Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery. It is a variant of the Straight-Through Estimator (STE) designed to improve gradient estimation by incorporating information about the scale of quantization errors. The method was proposed to address the limitations of vanilla STE, which uses a simple identity surrogate in the backward pass. While vanilla STE is robust, mSTE aims to provide a more informative gradient signal in high-bitrate regimes by rescaling the detached codec error using the standard deviation of the quantization error [S1].

## Sources
[S1] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
