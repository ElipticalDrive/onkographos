# Straight-Through Estimator (STE) for Confidence Scores

Also called: Straight-Through Estimator for Gradient Extension

## Overview
The Straight-Through Estimator (STE) for confidence scores is a gradient approximation technique used in voxel selection schemes to enable backpropagation through discrete or hard-thresholded selection decisions [S1, unofficial]. It solves the problem of non-differentiable voxel masking by allowing gradients to flow directly from the loss to the confidence scores, bypassing the hard boundary of the selection function [S2]. Unlike the Gumbel-Sigmoid approach used in VS-Splat, which provides a differentiable relaxation, STE applies a hard selection during the forward pass while passing gradients unchanged during the backward pass [S1, unofficial].

## How it works
The method is applied within a voxel selection framework where each voxel $v$ is assigned a learned confidence score $c_v$. The goal is to select a subset of voxels for rendering based on these scores.

**Inputs and Outputs:**
- Input: A set of confidence scores $\{c_v\}$ for all candidate voxels.
- Output: A binary mask $m_v \in \{0, 1\}$ indicating selected voxels, and gradients $\frac{\partial \mathcal{L}}{\partial c_v}$ for optimization.

**Data Structures:**
- Confidence scores: A tensor of floating-point values, typically in $[0, 1]$.
- Selection mask: A binary tensor of the same shape as the confidence scores.

**Steps:**
1. **Forward Pass (Hard Selection):**
   Compute the binary selection mask $m_v$ using a threshold $\tau$:
   $$
   m_v = \begin{cases} 
   1 & \text{if } c_v > \tau \\
   0 & \text{otherwise}
   \end{cases}
   $$
   The rendered output is computed using only voxels where $m_v = 1$. The threshold $\tau$ is typically a fixed hyperparameter or learned, but the selection itself is non-differentiable [S1, unofficial].

2. **Backward Pass (Straight-Through):**
   Compute the gradient of the loss $\mathcal{L}$ with respect to the confidence scores $c_v$ by ignoring the discontinuity of the thresholding operation:
   $$
   \frac{\partial \mathcal{L}}{\partial c_v} = \frac{\partial \mathcal{L}}{\partial \text{output}} \cdot \frac{\partial \text{output}}{\partial c_v} \approx \frac{\partial \mathcal{L}}{\partial \text{output}} \cdot \frac{\partial \text{output}}{\partial m_v}
   $$
   In practice, this means the gradient flows through the hard threshold as if it were an identity function for the selected voxels and zero for unselected ones, or simply passes the upstream gradient directly to $c_v$ for all voxels depending on the specific implementation variant [S1, unofficial][S2].

**Parameters:**
- Threshold $\tau$: A scalar value determining the selection cutoff. In the VS-Splat context, this is often compared against opacity-based thresholding (OBT) which uses a predefined threshold like 0.005 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Limited Selection:** Simply applying STE to confidence scores can result in selecting an extremely limited number of voxels, leading to poor rendering quality [S1, unofficial].
- **Optimization Stalling:** Without extending gradients beyond the primitive boundary, optimization may stall. STE helps by allowing gradient flow even where the primitive boundary is hard or truncated [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(confidence_scores, threshold):
    # Hard selection: 1 if score > threshold, else 0
    mask = (confidence_scores > threshold).float()
    # Render using masked voxels
    rendered_output = render(masked_voxels)
    return rendered_output, mask

def backward(loss, confidence_scores, mask):
    # Straight-through: pass gradient directly to confidence scores
    # ignoring the non-differentiable threshold operation
    grad_confidence = torch.autograd.grad(loss, confidence_scores, retain_graph=True)[0]
    # Note: In some implementations, gradients might be zeroed for unselected voxels
    # grad_confidence = grad_confidence * mask
    return grad_confidence
```

## Subtopics

### Opacity-Based Thresholding (OBT)
Uses a fixed opacity threshold (e.g., 0.005) to select voxels. It is simple but requires manual tuning and can be unreliable for object-centric selection [S1, unofficial].

### Gumbel-Sigmoid Relaxation
Uses a stochastic relaxation of the selection process to allow differentiable optimization. It generally yields better voxel selection and rendering quality than STE [S1, unofficial].

### Deterministic Sigmoid
Replaces Gumbel-Sigmoid with a standard sigmoid function. This fails to properly select object-centric voxels and leads to significant rendering quality drops [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The forward pass is $O(N)$ where $N$ is the number of voxels. The backward pass is also $O(N)$ as it involves a simple element-wise operation.
- **Memory Complexity:** $O(N)$ to store the confidence scores and the selection mask.
- **Performance:** On the CO3D dataset, using STE for voxel selection resulted in a PSNR of 14.71, SSIM of 0.814, and LPIPS of 0.317, with only 121 voxels selected [S1, unofficial]. This is significantly worse than the proposed VS-Splat method (PSNR 21.31, 19,587 voxels) and comparable to OBT (PSNR 21.21, 22,867 voxels) but with far fewer voxels, indicating poor selection quality [S1, unofficial].
- **Strengths:** Simple to implement; enables gradient flow through hard selection boundaries.
- **Limitations:** Can lead to very sparse voxel selection and poor rendering quality if not carefully tuned. It does not inherently encourage object-centric selection as effectively as Gumbel-Sigmoid [S1, unofficial].

## Background
The Straight-Through Estimator is a common technique in differentiable programming to handle non-differentiable operations [S2]. In the context of 3D Gaussian Splatting and voxel selection, it was explored as a variant of the VS-Splat framework to enable gradient flow to confidence scores [S1, unofficial]. It is also used in other radiance field methods, such as Fourier Splatting, to extend gradients beyond primitive boundaries and prevent optimization stalling [S2].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
[S2] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
