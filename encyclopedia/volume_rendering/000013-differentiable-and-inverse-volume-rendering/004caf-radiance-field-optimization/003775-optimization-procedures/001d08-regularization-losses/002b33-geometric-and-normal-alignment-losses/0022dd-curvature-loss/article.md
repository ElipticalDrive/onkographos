# Curvature Loss
## Overview
Curvature Loss is a geometric regularization term that applies a curvature operator to the model output and scales the result by a configurable weight [S1, unofficial]. It operates on second-order derivatives of the implicit function, distinguishing it from sibling methods that use first-order derivatives or pixel-space alignment [G].

## How it works
**Inputs and Outputs:**
- Input: Model output (scalar field values) at query points [S1, unofficial].
- Output: A scalar curvature loss value [S1, unofficial].

**Data Structures:**
- Not given in the sources [G].

**Method:**
1. Evaluate the model output at the query points [S1, unofficial].
2. Apply the curvature operator $C$ to the model output [S1, unofficial].
3. Multiply the result by the curvature loss weight $\lambda_{\text{curvature}}$ [S1, unofficial].

**Formula:**
$$L_{\text{curv}} = \lambda_{\text{curvature}} \cdot C(\text{model output})$$
where:
- $\lambda_{\text{curvature}}$ is the curvature loss weight [S1, unofficial].
- $C(\cdot)$ is the curvature operator applied to the model output [S1, unofficial].

**Parameters:**
- $\lambda_{\text{curvature}}$: The weight for the curvature loss term, retrieved from `self.config.system.loss.lambda_curvature` [S1, unofficial]. Typical values are not given in the sources [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_loss(model_output, lambda_curvature):
    """
    Computes the curvature regularization loss.
    
    Args:
        model_output: Tensor of model predictions (e.g., SDF values).
        lambda_curvature: Float weight for the curvature loss.
        
    Returns:
        Scalar loss value.
    """
    # [G] Compute curvature operator C on model_output.
    # Standard implementations use automatic differentiation of the network
    # to compute second-order derivatives (e.g., Laplacian).
    curvature_values = model_output  # Represents C(model_output)
    
    # Multiply by the curvature loss weight
    weighted_curvature = lambda_curvature * curvature_values
    
    # [G] Reduce to a scalar by computing the arithmetic mean
    return weighted_curvature.mean()
```

**Edge Cases and Pitfalls:**
- **Precision:** Computing second-order derivatives can be numerically unstable if the network output is not smooth [G]. Ensure the network architecture supports stable gradient computation (e.g., using smooth activation functions) [G].
- **Boundaries:** Curvature is undefined or ill-behaved at domain boundaries [G]. Exclude points near the boundary of the query volume or use appropriate padding [G].
- **Degenerate Input:** If the scalar field is constant, curvature is zero and provides no gradient [G].

## Subtopics

### [SDF Edge Regularization Loss](3706-sdf-edge-regularization-loss.md)
Focuses on sign changes at mesh edges to enforce zero-crossing consistency for discrete representations like FlexiCubes. Sets itself apart by targeting edge connectivity rather than continuous curvature. Choose when sharp features or correct topology at boundaries are required.

### [Normal Smoothness Loss](4190-normal-smoothness-loss.md)
Penalizes discontinuities in surface normals along viewing rays. Sets itself apart by operating on first-order derivatives (dot product of consecutive normals) rather than higher-order operators. Choose when enforcing spatial coherence of surface orientation is prioritized over curvature.

### [Surface Curvature Regularization](6510-surface-curvature-regularization.md)
Computes and penalizes Gaussian and Mean curvatures directly on the implicit function. Sets itself apart by operating on the final implicit function's differential geometry within the loss. Choose when direct surface regularization is needed.

### [Curvature Regularization Decay](9943-curvature-regularization-decay.md)
Dynamically scales down curvature regularization strength as optimization progresses. Sets itself apart by adapting intensity to spatial resolution rather than using a static weight. Choose when preserving high-frequency details during fine-grained optimization is critical.

## Operating characteristics

## Background
The method is implemented in the Instant-NSR-PL framework [S1, unofficial]. Specific origin details and historical development are not given in the sources [G].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
