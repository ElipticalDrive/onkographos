# Field Output Prediction

Also called: Model Output Unpacking

## Overview
Field Output Prediction is the final stage of the neural volume rendering pipeline that transforms raw neural network logits into physically meaningful or semantically useful values for a specific reconstruction target. It solves the problem of mapping unconstrained model outputs to the required data format, such as plasma parameters or standard rendering attributes. It differs from its sibling [Sigma Network Layer](6256-sigma-network-layer.md), which focuses specifically on density/opacity, and [CNN Post-Processing Network](7179-cnn-post-processing-network.md), which applies spatial filters, by handling the direct semantic unpacking of the MLP's final layer.

## How it works
The method takes raw predictions from the preceding network layer and converts them based on the configured `reconstruction_target`.

**Inputs and Outputs:**
- Input: Raw tensor from the neural network model, shape $(N, C)$ or $(N,)$, where $N$ is the number of samples and $C$ is the channel dimension.
- Output: Unpacked tensor of semantically meaningful values (e.g., density, temperature, RGB).

**Data Structures:**
- `reconstruction_target`: A string or enum specifying the output type (e.g., "plasma", "rgb").
- `bounds_dict`: (Optional) Dictionary containing lower and upper bounds for physical quantities, used in clamping variants.

**Steps:**
1. **Model Forward Pass:** Compute raw outputs by passing input positions $x_{AABB}$ through the neural network model $M$.
   $$ raw = M(x_{AABB}) $$
2. **Dimension Handling:** If the output dimensionality of $raw$ is one less than the input dimensionality of $x_{AABB}$, append a singleton dimension to preserve spatial structure.
   $$ \text{if } raw.ndim == x_{AABB}.ndim - 1: \quad raw = raw[..., \text{None}] $$
3. **Flattening:** Reshape $raw$ to a 2D tensor for batch processing.
   $$ raw_{flat} = raw.reshape(-1, raw.shape[-1]) $$
4. **Unpacking:** Apply the unpacking function specific to the reconstruction target.
   $$ output = \text{unpack\_plasma\_outputs}(raw_{flat}, \text{reconstruction\_target}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def field_output_prediction(model, x_AABB, reconstruction_target):
    """
    Passes input positions through the neural network model and unpacks the raw outputs.
    
    Args:
        model: The neural network module.
        x_AABB: Input positions tensor.
        reconstruction_target: String specifying the output format (e.g., 'plasma').
        
    Returns:
        Unpacked output tensor.
    """
    # Step 1: Compute raw outputs
    raw = model(x_AABB)
    
    # Step 2: Handle dimensionality mismatch
    if raw.ndim == x_AABB.ndim - 1:
        raw = raw[..., None]
        
    # Step 3: Reshape to 2D
    raw_flat = raw.reshape(-1, raw.shape[-1])
    
    # Step 4: Unpack based on target
    # Implementation of unpack_plasma_outputs depends on the specific target.
    # For plasma, this involves applying bounds to log density and log temperature.
    output = unpack_plasma_outputs(raw_flat, reconstruction_target)
    
    return output
```

**Parameters and Edge Cases:**
- `reconstruction_target`: Determines the unpacking logic. Common values include "plasma" for astrophysical simulations or implicit RGB for standard NeRF.
- **Precision:** Raw outputs can be numerically unstable. Unconstrained outputs may lead to physically impossible values (e.g., negative density).
- **Boundaries:** If bounds are not applied, the model may produce values outside the valid physical range. The clamping variant addresses this by enforcing hard bounds.

## Subtopics

### [Learned Fields Prediction with Clamping](6361-learned-fields-prediction-with-clamping.md)
This variant predicts field outputs and applies clamping to log density ($\log n_e$) and log temperature ($\log T$) based on configured bounds. It serves as the final output layer in neural volume rendering pipelines, ensuring that physically impossible or numerically unstable values are constrained before further processing. It differs from raw field output prediction by enforcing hard bounds on specific physical quantities, addressing the instability of unconstrained neural network outputs in tomographic reconstruction. The bounds are derived from combining PSI and atomic constraints, raising an error if the lower bound exceeds the upper bound.

### Raw to Outputs Conversion
This variant transforms raw network predictions into semantically meaningful values like RGB, depth, and weights for volume rendering [S2, unofficial]. It extracts RGB values from the first three channels, converts raw sigma to alpha and weights using a `raw2alpha` function, and computes RGB, depth, disparity, and accumulated alpha maps by summing weighted contributions [S2, unofficial]. If a white background is requested, it adds the difference between 1 and accumulated alpha to the RGB map.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples $N$, as it involves a single forward pass and element-wise operations.
- **Memory Complexity:** Linear with respect to $N$, storing the raw and unpacked tensors.
- **Strengths:** Provides a flexible interface for different reconstruction targets; ensures physical validity in specialized domains like plasma simulation.
- **Limitations:** Performance depends on the underlying model's output dimension; clamping may introduce bias if bounds are too tight.
- **When to use:** When the application requires specific physical parameters (e.g., density, temperature) or standard rendering attributes (RGB, depth) from a neural field.

## Background
The method originates from implementations in neural volume rendering repositories, specifically those dealing with plasma reconstruction and point-based NeRF variants. It addresses the need to map latent space representations to observable quantities.

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
