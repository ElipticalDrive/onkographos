# Gradient-Aware ResNet Reconstruction

Also called: Gradient-Conditioned Reconstruction

## Overview
Gradient-Aware ResNet Reconstruction extends standard ResNet-based CT reconstruction by conditioning the neural network on the explicit Radon matrix $A$ (the forward projection operator) [S1, unofficial]. This allows the network to incorporate geometric projection constraints directly during the reconstruction process, rather than treating the input as a static noisy image [S1, unofficial]. It differs from pure post-processing methods by using the physics of the acquisition geometry to guide the denoising and artifact suppression [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tuple $(f, g)$, where $f$ is the current image estimate (initially the noisy filtered backprojection) and $g$ is the Radon matrix $A$ [S1, unofficial].
*   **Output:** A tuple containing the concatenated image-gradient tensor and the Radon matrix $g$ [S1, unofficial].

**Data Structures:**
*   The Radon matrix $A$ is passed explicitly as $g$. The source code indicates a boolean flag `sparse_svd` within an `explicit_radon` context, defaulting to `True` [S1, unofficial].
*   Gradient utilities (`applyDx`, `applyDy`, `applyDxTrans`, `applyDyTrans`) implement finite differences using `np.diff` with boundary padding to handle image edges [S1, unofficial].

**Algorithm Steps:**
1.  **Gradient Computation:** The `gradient` method computes the gradient of the data fidelity term. It projects $f$ using $A$, computes $A^T(Af)$, permutes the result, and triggers a backward pass on the mean to extract $\nabla_f$ [S1, unofficial]. The output is reshaped to match the input image dimensions [S1, unofficial].
2.  **Concatenation:** The image $f$ and computed gradient are concatenated along the channel dimension: $x_{\text{cat}} = \text{cat}(f, \nabla_f, \text{dim}=1)$ [S1, unofficial].
3.  **Output:** The module returns the concatenated tensor and the Radon matrix $g$ for subsequent processing steps [S1, unofficial].
4.  **Training Loop:** The model is trained via supervised learning. Inputs and targets are passed through the model, a loss (e.g., MSE) is computed, and gradients are backpropagated to update weights [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class GradientAwareResNet(nn.Module):
    def __init__(self):
        super().__init__()
        # ResNet layers processing concatenated image and gradient
        self.resnet = nn.Sequential(
            nn.Conv2d(2, 64, 3, padding=1),
            nn.ReLU(),
            nn.Conv2d(64, 1, 3, padding=1)
        )

    def gradient(self, f, g):
        # f: image estimate [B, 1, H, W]
        # g: Radon matrix A [B, N_proj, N_det] or similar
        # Compute forward projection Af
        Af_g = torch.matmul(g, f.view(f.shape[0], -1)).view(f.shape)
        # Compute A^T(Af) and adjust dimensions
        ATAf_g = torch.matmul(g.transpose(-2, -1).squeeze(0), Af_g).permute(0, 2, 1)
        # Extract gradient via autograd backward pass on mean
        ATAf_g.mean().backward()
        out = f.grad.permute(0, 2, 1).reshape(f.shape)
        return out

    def forward(self, x):
        f, g = x
        grad = self.gradient(f, g)
        x_cat = torch.cat((f, grad), dim=1)
        return (x_cat, g)

# Training loop structure from [S1, unofficial]
def train_step(model, inputs, targets, optimizer, criterion):
    optimizer.zero_grad()
    outputs = model(inputs)
    loss = criterion(outputs, targets.float())
    loss.backward()
    optimizer.step()
```

**Parameters:**
*   `sparse_svd`: Boolean flag in `explicit_radon` to control sparse SVD computation. Default: `True` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The gradient computation relies on matrix operations with the Radon matrix, which can be memory-intensive for large datasets [S1, unofficial].
*   **Boundaries:** The project includes finite difference utilities (`applyDx`, `applyDy`, etc.) that use `np.diff` with boundary padding, implying zero-padding or similar boundary handling must be consistent with the projection geometry [S1, unofficial].

## Subtopics

### Learned Model-Based Reconstruction
Incorporates the Radon transform and its gradient directly into a learned iterative framework [S1, unofficial]. At each step, the gradient of the data fidelity term is computed and passed through a residual block alongside the image to refine the reconstruction [S1, unofficial]. It merges traditional optimization with neural network flexibility, allowing the model to adaptively learn prior constraints from inverse problems theory, unlike pure post-processing which only corrects final images [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the size of the Radon matrix $A$ and the ResNet depth. The gradient computation involves matrix-vector multiplications ($A$ and $A^T$), which scales with the number of projection angles and detector samples [S1, unofficial].
*   **Memory Complexity:** Requires storing the explicit Radon matrix $A$ and intermediate gradient tensors. The concatenation of $f$ and `grad` doubles the channel dimension input to the ResNet [S1, unofficial].
*   **Strengths:** Incorporates physical constraints (geometry) directly into the learning process, potentially leading to more accurate reconstructions than methods ignoring the projection model [S1, unofficial].
*   **Limitations:** Requires explicit access to the Radon matrix, which may not be available or feasible to store for very large datasets [S1, unofficial].

## Background
This method originates from the `inverse-problems-CT-scans` repository by imanfs [S1, unofficial]. It builds on the concept of using ResNets for CT reconstruction [8264-resnet-based-ct-reconstruction.md] by conditioning the network on the forward projection operator [S1, unofficial]. The source code includes utilities for wavelet thresholding and iterative thresholding, indicating a broader context of inverse problem solving in CT [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
