# Conditional Normalizing Flow (Sylvester Flow)

Also called: CNF, Sylvester Flows

## Overview
Conditional Normalizing Flows (CNFs) using Sylvester transformations map samples from a latent prior distribution to radiance and density values, conditioned on 5D location-direction pairs [S1]. This method serves as the invertible transformation layer within the Voxel-Conditioned Continuous Normalizing Flow (VC-CNF) framework [6313-voxel-conditioned-continuous-normalizing-flow-vc-cnf.md], enabling the generation of probabilistic radiance fields with reliable uncertainty quantification. It distinguishes itself from standard MLP-based radiance fields by providing a structured, invertible mapping that allows for exact likelihood computation and depth uncertainty estimation, rather than deterministic point estimates [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A sample $z_{k-1}$ from the latent prior distribution and a 5D conditional feature vector $c$ derived from the spatial location and viewing direction.
*   **Output:** The transformed sample $z_k$, which contributes to the final radiance and density values.

**Data Structures**
The flow parameters are not static tensors but are outputs of a conditional function.
*   **Conditional Feature:** A vector of dimension 64, derived from the 5D location-direction pair [S1].
*   **Flow Parameters:** Matrices $A$ and $B$, and bias vector $b$, which are functions of the conditional feature [S1].

**Algorithm**
The transformation is applied sequentially through $K$ flow steps. In the CF-NeRF implementation, $K=4$ for both radiance and density CNFs [S1]. During training, a single sample from the latent prior distribution is shared across different spatial-location and viewing-direction inputs within the same batch to prevent overfitting on sparse training views [S1].

1.  Initialize the latent sample $z_0$ from the prior distribution.
2.  For each flow step $k$ from 1 to $K$:
    a. Compute the conditional feature $c$ from the 5D location-direction pair.
    b. Generate flow parameters $A, B, b$ from $c$ using the conditional network.
    c. Apply the Sylvester transformation:
       $$z_k = z_{k-1} + A h(B z_{k-1} + b)$$
       where $h$ is the hyperbolic tangent activation function ($\tanh$) [S1].
    d. Set $z_{k-1} \leftarrow z_k$ for the next iteration.
3.  After $K$ steps, the final $z_K$ is used to decode radiance and density.

**Parameters**
*   **Number of flow steps ($K$):** 4 [S1].
*   **Conditional feature dimension:** 64 [S1].
*   **MLP Hidden Units:** 512 for all layers in the backbone network [S1].
*   **Entropy Loss Weight:** 0.01 [S1].
*   **Depth Loss Weight:** $1 \times 10^{-2}$ (0.01) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sylvester_flow_step(z_prev, conditional_feature):
    """
    Performs one step of the Sylvester Normalizing Flow.
    
    Args:
        z_prev: Previous latent vector.
        conditional_feature: 64-dimensional vector derived from 5D location-direction.
        
    Returns:
        z_next: Transformed latent vector.
    """
    # Generate flow parameters from conditional feature
    # Note: The specific architecture of the conditional network is not detailed
    # beyond using the same MLP backbone as NeRF with 512 hidden units [S1].
    # [G] Assuming a standard MLP maps conditional_feature to A, B, b.
    A, B, b = generate_parameters(conditional_feature)
    
    # Apply Sylvester transformation
    # h is tanh activation
    transformed = tanh(dot_product(B, z_prev) + b)
    z_next = z_prev + dot_product(A, transformed)
    
    return z_next

def conditional_normalizing_flow(z_0, conditional_features, num_flows=4):
    """
    Applies K sequential Sylvester flows.
    
    Args:
        z_0: Initial sample from latent prior.
        conditional_features: List of K conditional feature vectors.
        num_flows: Number of flow steps (K).
        
    Returns:
        z_K: Final transformed latent vector.
    """
    z = z_0
    for k in range(num_flows):
        z = sylvester_flow_step(z, conditional_features[k])
    return z
```

**Edge Cases and Pitfalls**
*   **Overfitting:** With sparse training views, the model may overfit. The method mitigates this by sharing latent samples across different inputs in a batch and adding a depth loss [S1].
*   **Parameter Generation:** The conditional network must reliably map the 5D input to valid $A, B, b$ parameters. Degenerate inputs could lead to unstable transformations if the conditional network is not robust.

## Operating characteristics
**Time and Memory Complexity**
The complexity is linear with respect to the number of flow steps $K$ and the dimensionality of the latent vector. Each step involves matrix-vector multiplications and a non-linear activation. The memory overhead includes storing the parameters $A, B, b$ for each flow step, which are generated on-the-fly from the conditional feature.

**Measured Performance**
On the LF dataset, CF-NeRF with 4 Sylvester flows achieves the following metrics [S1]:
*   **PSNR:** 24.78
*   **SSIM:** 0.86
*   **LPIPS:** 0.168
*   **AUSE RMSE:** 0.051
*   **AUSE MAE:** 0.026
*   **NLL:** -0.710
*   **Depth RMSE:** 0.118
*   **Depth MAE:** 0.074
*   **Depth $\delta_3$:** 0.81

**Ablation Studies**
*   **Without Entropy Term:** Removing the entropy term (weight 0.01) degrades performance (PSNR: 23.82, SSIM: 0.83, NLL: -0.578) [S1].
*   **Single Flow:** Using only one flow step degrades performance further (PSNR: 23.40, SSIM: 0.81, NLL: -0.448) [S1].

**Strengths and Limitations**
*   **Strengths:** Provides accurate 3D modeling with reliable uncertainty quantification. The invertible nature allows for exact likelihood computation [S1].
*   **Limitations:** Requires careful tuning of loss weights (entropy, depth) to avoid overfitting on sparse data. The conditional network adds computational overhead compared to static MLPs.

## Background
Conditional Normalizing Flows, specifically Sylvester Flows, were introduced as a method for density estimation and generative modeling. In the context of neural radiance fields, they were adapted by Shen et al. in "Conditional-Flow NeRF: Accurate 3D Modelling with Reliable Uncertainty Quantification" to provide uncertainty-aware rendering [S1]. This approach builds on the original NeRF architecture [36] and integrates with the VC-CNF framework [6313-voxel-conditioned-continuous-normalizing-flow-vc-cnf.md] to condition the latent distribution on voxel features.

## Sources
[S1] Conditional-Flow NeRF: Accurate 3D Modelling with Reliable Uncertainty Quantification, https://doi.org/10.1007/978-3-031-20062-5_31, fetched 2026-10-30
