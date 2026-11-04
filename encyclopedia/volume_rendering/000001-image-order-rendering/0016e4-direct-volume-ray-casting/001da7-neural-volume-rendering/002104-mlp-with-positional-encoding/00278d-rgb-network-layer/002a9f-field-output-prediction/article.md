# Field Output Prediction

Also called: Model Output Unpacking

## Overview
Field Output Prediction is the final stage of the RGB Network Layer in neural volume rendering, responsible for translating raw neural network logits into physically meaningful field values (such as density and temperature) based on a specified reconstruction target. It differs from sibling layers like the Sigma Network Layer by handling the semantic unpacking of multi-channel outputs rather than just computing opacity coefficients, ensuring the model's output aligns with the specific physical quantities required for the rendering pipeline.

## How it works
The method takes raw positional inputs and a reconstruction target configuration, processes them through the underlying MLP, and unpacks the resulting tensor into named field components.

**Inputs and Outputs:**
- **Input:** `x_AABB`, a tensor of shape $(N, D)$ representing $N$ query points in $D$ dimensions (typically 3D spatial coordinates); `self.reconstruction_target`, a string or enum defining the output format (e.g., "plasma").
- **Output:** A structured object or tuple containing unpacked fields (e.g., `log_ne`, `log_temp`).

**Data Structures:**
- `raw`: Intermediate tensor of shape $(N, C)$ where $C$ is the number of output channels from the MLP.
- `outs`: A named tuple or dictionary-like object holding the unpacked fields.

**Steps:**
1. **Forward Pass:** Compute the raw output of the neural network model using the input positions:
   $$ \text{raw} = \text{self.model}(x_{\text{AABB}}) $$
2. **Dimension Handling:** If the output tensor `raw` has fewer dimensions than the input `x_AABB` (i.e., `raw.ndim == x_AABB.ndim - 1`), append a singleton dimension to the end to preserve batch structure:
   $$ \text{raw} = \text{raw}[..., \text{None}] $$
3. **Flattening:** Reshape `raw` to a 2D tensor of shape $(-1, C)$ to facilitate batched unpacking:
   $$ \text{raw\_flat} = \text{raw.reshape}(-1, \text{raw.shape}[-1]) $$
4. **Unpacking:** Pass the flattened raw output and the reconstruction target to the unpacking function:
   $$ \text{result} = \text{unpack\_plasma\_outputs}(\text{raw\_flat}, \text{self.reconstruction\_target}) $$
   The unpacking function maps specific channel indices from `raw_flat` to semantic fields based on the target definition. For plasma targets, this typically involves mapping channels to logarithmic density (`log_ne`) and logarithmic temperature (`log_temp`).

**Parameters:**
- `self.reconstruction_target`: Defines the semantic meaning of the output channels. Typical value: `"plasma"`.
- `self.model`: The underlying MLP instance.

**Edge Cases and Pitfalls:**
- **Dimension Mismatch:** If the model output does not match the expected channel count for the reconstruction target, unpacking will fail or produce incorrect physical values.
- **Precision:** The use of logarithmic fields (`log_ne`, `log_temp`) helps handle wide dynamic ranges in plasma physics but requires careful handling of zero or negative raw outputs before exponentiation if linear values are needed later.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_field_outputs(self, x_AABB):
    """
    Predicts field outputs from the model and unpacks them.
    
    Args:
        x_AABB: Tensor of shape (N, D) containing query positions.
        
    Returns:
        outs: Structured object containing unpacked fields (e.g., log_ne, log_temp).
    """
    # Step 1: Compute raw output from the neural network
    raw = self.model(x_AABB)
    
    # Step 2: Handle dimensionality
    if raw.ndim == x_AABB.ndim - 1:
        raw = raw[..., None]
        
    # Step 3: Flatten for unpacking
    raw_flat = raw.reshape(-1, raw.shape[-1])
    
    # Step 4: Unpack based on reconstruction target
    # The unpacking logic maps channel indices to semantic fields
    # For 'plasma' target, this typically assigns specific channels to log_ne and log_temp
    outs = unpack_plasma_outputs(raw_flat, self.reconstruction_target)
    
    return outs
```

## Subtopics

### Learned Fields Prediction with Clamping
This variant extends the basic prediction by applying hard bounds to the predicted logarithmic fields to ensure physical validity. It calls the base prediction method and then clamps `log_ne` and `log_temp` to configured minimum and maximum values. Clamping is applied if `self.clamp` is True. The bounds are derived from combined constraints (e.g., PSI and atomic constraints for temperature). If the lower bound exceeds the upper bound for any field, a `ValueError` is raised. This ensures that the predicted values remain within physically plausible ranges for plasma simulations.

## Operating characteristics
- **Time Complexity:** Dominated by the forward pass of the neural network, $O(N \cdot C \cdot L)$ where $N$ is the number of points, $C$ is the hidden width, and $L$ is the depth. Unpacking is $O(N)$.
- **Memory Complexity:** $O(N \cdot C)$ for storing the raw output tensor.
- **Strengths:** Decouples the physical interpretation from the network architecture, allowing flexible reconstruction targets. Clamping prevents unphysical values.
- **Limitations:** Relies on the MLP's ability to predict values within the bounded range. Poorly chosen bounds can lead to saturation and loss of gradient information.

## Background
This method is part of the CoroNeRF framework, which integrates neural rendering with plasma physics simulations. It was developed to handle the complex, multi-modal outputs required for reconstructing plasma density and temperature fields from sparse or synthetic data. The approach builds on the general MLP with Positional Encoding paradigm but specializes the output layer for scientific visualization tasks.

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
