# Curvature Loss

Also called: Curvature Regularization

## Overview
Curvature Loss is a geometric regularization term used in differentiable volume rendering and neural implicit surface optimization to penalize high-frequency oscillations in the predicted scalar field (density or signed distance function). It enforces spatial smoothness by computing a discrete approximation of the Laplacian or curvature operator on the model output [S1, unofficial]. It sets itself apart from [Normal Ray Alignment Loss](4202-normal-ray-alignment-loss.md) and [Predicted Normal Loss](7713-predicted-normal-loss.md) by operating directly on the scalar field values rather than on derived normal vectors or ray alignments, and from [Material Smoothness Loss](7299-material-smoothness-loss.md) by targeting geometric curvature rather than material property continuity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The model output tensor $V$ (density $\sigma$ or SDF values) at query points, typically a tensor of shape $(N, 1)$ or $(N, C)$ where $N$ is the number of samples.
*   **Output:** A scalar loss value $L_{curv}$.

**Data Structures:**
*   `self.C`: A curvature operator module or function. In the reference implementation, this is often a discrete Laplacian kernel or a finite difference operator applied to the grid or point cloud [S1, unofficial].
*   `lambda_curvature`: A scalar hyperparameter controlling the weight of the loss term [S1, unofficial].

**Steps:**
1.  **Compute Curvature:** Apply the curvature operator $C$ to the model output $V$.
    $$ \hat{K} = C(V) $$
    Where $\hat{K}$ represents the estimated curvature or Laplacian of the field [S1, unofficial].
2.  **Compute Loss:** Calculate the mean squared error (or L2 norm) of the curvature values and scale by the weight.
    $$ L_{curv} = \lambda_{curv} \cdot \frac{1}{N} \sum_{i=1}^N \hat{K}_i^2 $$
    [S1, unofficial]

**Parameters:**
*   `lambda_curvature` ($\lambda_{curv}$): The weight for the curvature loss term. Typical values are small (e.g., $10^{-4}$ to $10^{-2}$) to avoid over-smoothing [S1, unofficial]. Retrieved from configuration `self.config.system.loss.lambda_curvature` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_loss(model_output, curvature_operator, lambda_curvature):
    """
    Computes the curvature regularization loss.
    
    Args:
        model_output: Tensor of shape (N, 1) or (N, C) containing density or SDF values.
        curvature_operator: Function or module applying the curvature/Laplacian operator.
        lambda_curvature: Scalar weight for the loss term.
        
    Returns:
        Scalar loss value.
    """
    # Step 1: Apply curvature operator to the model output
    # C is the curvature operator, e.g., discrete Laplacian
    curvature_values = curvature_operator(model_output)
    
    # Step 2: Compute the mean squared curvature
    # Using L2 loss on the curvature values
    loss = torch.mean(curvature_values ** 2)
    
    # Step 3: Scale by the hyperparameter
    weighted_loss = lambda_curvature * loss
    
    return weighted_loss
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Discrete curvature operators may produce inaccurate results at the boundaries of the domain or near sparse regions. Padding or boundary-aware operators may be required [G].
*   **Over-smoothing:** Excessive weight can remove fine geometric details. This is mitigated by dynamic weighting strategies like [Curvature Regularization Decay](9943-curvature-regularization-decay.md) [S1, unofficial].
*   **Scale Sensitivity:** The magnitude of the curvature depends on the scale of the input field. Normalization of the input field or adaptive weighting may be necessary [G].

## Subtopics

### [Normal Smoothness Loss](4190-normal-smoothness-loss.md)
A regularization loss that encourages consecutive surface normals along a viewing ray to be similar, enforcing spatial smoothness in the estimated geometry. It sets itself apart from [Surface Curvature Regularization](6510-surface-curvature-regularization.md) by operating on discrete sample points along a ray rather than computing differential geometric invariants like mean or Gaussian curvature from a continuous field.

### [Surface Curvature Regularization](6510-surface-curvature-regularization.md)
Surface Curvature Regularization is a technique for regularizing neural implicit surfaces (specifically Signed Distance Functions, SDFs) by computing and penalizing Gaussian and Mean curvatures directly during training, rather than regularizing projections or depth maps. It sets itself apart from sibling methods by operating on the final implicit function's differential geometry within the loss function, enabling direct surface regularization. It builds on [Curvature Loss](2911-curvature-loss.md).

### [Curvature Regularization Decay](9943-curvature-regularization-decay.md)
Curvature Regularization Decay is a dynamic weighting strategy for curvature-based geometric losses in differentiable volume rendering, specifically designed to preserve high-frequency details during optimization. It addresses the trade-off between surface smoothness and detail retention by scaling down the curvature regularization strength ($w_{curv}$) as the optimization step size ($\epsilon$) decreases and finer hash grid resolutions are activated. Unlike static curvature losses, which may overly smooth fine structures, this method adapts the regularization intensity to the current spatial resolution.

#### SDF Edge Regularization Loss
**Definition:** A loss function that penalizes SDF values at edges where the sign changes, encouraging sharp features or correct topology at boundaries [S2, unofficial].

**Steps:**
1.  Gather SDF values at edge endpoints: `sdf_f1x6x2 = sdf[all_edges.reshape(-1)].reshape(-1, 2)` [S2, unofficial].
2.  Create mask for edges where signs differ: `mask = sign(sdf_f1x6x2[:, 0]) != sign(sdf_f1x6x2[:, 1])` [S2, unofficial].
3.  Filter SDF values to only those edges with sign changes: `sdf_f1x6x2 = sdf_f1x6x2[mask]` [S2, unofficial].
4.  Compute binary cross-entropy loss between first endpoint and boolean second endpoint, and vice versa [S2, unofficial].
5.  Return the computed loss [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of samples, as it involves a single pass over the output tensor to compute the curvature and its square [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the curvature values before reduction [G].
*   **Strengths:** Simple to implement; effective at removing noise and high-frequency artifacts in the scalar field; differentiable [S1, unofficial].
*   **Limitations:** Can over-smooth fine details if not weighted carefully; discrete operators may introduce grid-dependent artifacts [S1, unofficial].
*   **When to Use:** When the optimization process exhibits noisy or jagged geometry; when using neural implicit representations that are prone to high-frequency oscillations [S1, unofficial].
*   **When Not to Use:** When preserving high-frequency details is critical and dynamic weighting is not employed; when the scalar field is already well-regularized [S1, unofficial].

## Background
Curvature Loss originates from the broader field of geometric regularization in computer vision and graphics, where Laplacian smoothing is a standard technique for mesh and field denoising. In the context of differentiable volume rendering and neural implicit surfaces, it was adopted to stabilize the optimization of neural networks predicting density or SDF fields, particularly in methods like Instant-NSR-PL [S1, unofficial]. The specific implementation cited here is from the `instant-nsr-pl` repository [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S2] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
