# LPIPS Computation
Also called: LPIPS, Learned Perceptual Image Patch Similarity, LPIPS Loss
## Overview
LPIPS (Learned Perceptual Image Patch Similarity) measures perceptual similarity between original and reconstructed images by comparing deep features extracted from neural networks, outputting a distance metric where lower values indicate higher similarity [S1][S2, unofficial]. It is preferred over pixel-space metrics like Mean Squared Error (MSE) because it accounts for neighboring information and structural context, providing a more robust optimization objective for differentiable rendering [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two image tensors, `render` (the generated image) and `gt` (the ground truth image), both in the range $[0, 1]$ [S2, unofficial].
- Output: A scalar float representing the perceptual distance [S2, unofficial].

**Data Structures:**
- The method relies on a pretrained VGG network configured as a feature extractor, set to evaluation mode to disable dropout and batch normalization updates [S2, unofficial].

**Steps:**
1. **Normalization:** Scale the pixel values of both `render` and `gt` from the range $[0, 1]$ to $[-1, 1]$ using the linear transformation:
   $$ x_{norm} = 2.0 \cdot x - 1.0 $$
   This aligns the input distribution with the expectations of the pretrained VGG model [S2, unofficial].
2. **Batching:** If the input tensors do not have a batch dimension, add one using `unsqueeze(0)` [S2, unofficial].
3. **Feature Extraction:** Pass the normalized tensors through the pretrained VGG model [S2, unofficial].
4. **Aggregation:** Return the mean of the output tensor as a single float value [S2, unofficial].

**Parameters:**
- `net`: The backbone architecture for perceptual features. The default and most common value is `"vgg"` [S2, unofficial].

**Pitfalls:**
- **First Run Download:** The first execution requires downloading pretrained weights to the `TORCH_HOME` cache directory, which requires an internet connection or pre-cached weights [S2, unofficial].
- **Input Range:** Failing to normalize inputs to $[-1, 1]$ before passing them to the VGG model will result in incorrect feature activations and invalid distance scores [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def compute_lpips(render: torch.Tensor, gt: torch.Tensor) -> float:
    # 1. Load pretrained VGG model and set to evaluation mode [S2, unofficial]
    # Standard VGG-16 feature extractor initialization [G]
    # Not given in the sources.
    model = nn.Sequential(...) # Standard VGG layers [G]
    model.eval()

    # 2. Scale render and GT tensors from [0,1] to [-1,1] [S2, unofficial]
    render_norm = 2.0 * render - 1.0
    gt_norm = 2.0 * gt - 1.0

    # 3. Add batch dimension with unsqueeze(0) [S2, unofficial]
    if render_norm.dim() == 3:
        render_norm = render_norm.unsqueeze(0)
    if gt_norm.dim() == 3:
        gt_norm = gt_norm.unsqueeze(0)

    # 4. Pass through model [S2, unofficial]
    with torch.no_grad():
        distance = model(render_norm, gt_norm)

    # 5. Return mean of output tensor as float [S2, unofficial]
    return distance.mean().item()
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Accounts for neighboring information and structural context, providing a more robust optimization objective than pixel-space metrics [S3, unofficial].
- **Limitations:** Requires a pretrained model and internet access for initial weight download [S2, unofficial].
- **When to use:** When perceptual quality is more important than exact pixel fidelity [S3, unofficial].
- **When not to use:** When computational efficiency is paramount and pixel-level accuracy is sufficient, or when network access is unavailable for weight caching [S2, unofficial].

## Background
LPIPS was introduced by Zhang et al. in 2018 [S3, unofficial]. It was designed to address the limitations of pixel-space metrics like MSE, which do not account for neighboring information and often lead to suboptimal optimization results in perceptual tasks [S3, unofficial]. It is used in differentiable rendering pipelines [S3, unofficial].

## Sources
[S1] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S2] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S3] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
