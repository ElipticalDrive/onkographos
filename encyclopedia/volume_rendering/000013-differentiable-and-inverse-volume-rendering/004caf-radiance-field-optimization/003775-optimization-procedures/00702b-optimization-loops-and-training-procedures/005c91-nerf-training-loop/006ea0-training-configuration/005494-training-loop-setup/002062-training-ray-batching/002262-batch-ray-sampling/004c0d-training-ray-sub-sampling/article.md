# Training Ray Sub-sampling

Also called: SampleRays Transform, Sub-sampling Rays

## Overview
Training Ray Sub-sampling is a data transform that reduces the number of rays processed per training iteration by selecting a fixed subset from a larger batch of multiview rays [S1, unofficial]. It solves the problem of high memory consumption and computational cost during radiance-field optimization by limiting the batch size to a manageable `num_samples` count [S1, unofficial]. Unlike [Batch Ray Sampling](2767-batch-ray-sampling.md), which determines *which* rays enter the pipeline, this method determines *how many* of those rays are actually forwarded to the network for gradient computation [S1, unofficial].

## How it works
**Inputs:**
- `inputs`: A dictionary or object containing ray data.
  - `inputs['rays']`: Tensor of shape `[B, R, ...]` where `B` is the batch size (number of images/videos) and `R` is the number of rays per image.
  - `inputs.ray_values()`: An iterable of key-value pairs representing auxiliary ray data (e.g., colors, normals, semantic labels).

**Outputs:**
- `out`: A dictionary containing the sub-sampled data.
  - `out['rays']`: Tensor of shape `[num_samples, ...]`.
  - `out[channel_name]`: Tensors corresponding to the sub-sampled auxiliary data.

**Parameters:**
- `num_samples` (int): The fixed number of rays to select from the input batch [S1, unofficial].

**Algorithm:**
1.  **Generate Indices:** Create a tensor of random integers to select `num_samples` unique indices from the available rays.
    $$ \text{ray\_idx} = \text{torch.randint}(0, \text{inputs}['\text{rays}'].shape[0], [\text{num\_samples}], \text{device}=\text{device}) $$
    *Note: The source code uses `inputs['rays'].shape[0]` as the upper bound, implying `num_samples` is typically less than or equal to the total number of rays in the batch, or the implementation assumes a specific batch structure where this dimension represents the ray count [S1, unofficial].*

2.  **Sub-sample Rays:** Extract the selected rays from the main ray tensor.
    $$ \text{out}['\text{rays}'] = \text{inputs}['\text{rays}'][\text{ray\_idx}].\text{contiguous}() $$

3.  **Sub-sample Auxiliary Data:** Iterate over all auxiliary ray values and apply the same index selection.
    For each `(channel_name, ray_value)` in `inputs.ray_values()`:
    $$ \text{out}[\text{channel\_name}] = \text{ray\_value}[\text{ray\_idx}].\text{contiguous}() $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def training_ray_sub_sampling(inputs, num_samples, device):
    """
    Sub-samples a fixed number of rays from the input batch.
    
    Args:
        inputs: Dictionary-like object with 'rays' tensor and ray_values() method.
        num_samples: Integer, number of rays to select.
        device: torch.device, target device for the output tensors.
        
    Returns:
        out: Dictionary containing sub-sampled rays and auxiliary data.
    """
    out = {}
    
    # Step 1: Generate random ray indices
    # Assuming inputs['rays'] is a tensor where the first dimension is the count of rays
    # to be sampled from.
    ray_count = inputs['rays'].shape[0]
    ray_idx = torch.randint(0, ray_count, [num_samples], device=device)
    
    # Step 2: Sub-sample main rays
    out['rays'] = inputs['rays'][ray_idx].contiguous()
    
    # Step 3: Sub-sample auxiliary ray values
    # Iterate over key-value pairs in ray_values
    for channel_name, ray_value in inputs.ray_values().items():
        out[channel_name] = ray_value[ray_idx].contiguous()
        
    return out
```

**Edge Cases and Pitfalls:**
- **Index Bounds:** If `num_samples` exceeds the number of available rays, `torch.randint` will generate valid indices, but the subsequent indexing `inputs['rays'][ray_idx]` may fail or produce unexpected results if the tensor shape does not support it. The source assumes `num_samples` is within valid bounds [S1, unofficial].
- **Contiguity:** The `.contiguous()` call is explicitly used to ensure memory layout compatibility for subsequent GPU operations [S1, unofficial].
- **Determinism:** The use of `torch.randint` implies non-deterministic sampling per iteration, which is standard for stochastic gradient descent but requires careful seeding for reproducibility [S1, unofficial].

## Subtopics

### Training Ray-Subsampling Scheme
A variant used for dynamic 6-DoF video training (HyperReel) that combines sub-sampling with temporal resolution adaptation [S2]. Instead of a fixed random selection, it iterates over $N$ videos and $M$ frames, applying conditional downsampling to rays and colors based on frame index divisibility [S2]. If frame $m$ is not divisible by 8, rays and colors are downsampled by a factor of 4 using nearest-neighbor interpolation [S2]. If $m$ is not divisible by 4, an additional factor of 2 downsampling is applied [S2]. This scheme builds a training set $\text{rays}_{GT}, C_{GT}$ by aggregating these potentially downsampled frames [S2]. It is chosen when training on video datasets with varying temporal importance or to reduce memory for high-resolution video sequences [S2].

## Operating characteristics
- **Time Complexity:** $O(K)$ where $K$ is `num_samples`, as it involves a single random generation and indexing operation [S1, unofficial].
- **Memory Complexity:** $O(K \cdot D)$ where $D$ is the dimensionality of the ray data, as it creates new contiguous tensors for the subset [S1, unofficial].
- **Performance:** Reduces GPU memory usage proportionally to the ratio of `num_samples` to the total batch size [S1, unofficial].
- **Strengths:** Simple to implement; effectively controls memory footprint during training [S1, unofficial].
- **Limitations:** Random sub-sampling may miss rare or important rays if the distribution is not uniform [S1, unofficial]. The HyperReel variant introduces spatial resolution changes that may affect training stability if not handled carefully [S2].

## Background
The basic `SampleRays` transform originates from the NVIDIA Kaolin-Wisp framework, a library for 3D deep learning [S1, unofficial]. The temporal sub-sampling scheme was introduced in the HyperReel paper for high-fidelity 6-DoF video generation, addressing the need to manage memory when training on large video datasets with multiple views [S2].

## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
[S2] HyperReel: High-Fidelity 6-DoF Video with Ray-Conditioned Sampling, https://doi.org/10.1109/cvpr52729.2023.01594, fetched 2026-10-26
