# ExpModel

Also called: WbModel

## Overview
ExpModel is a learned colorization module that applies a per-image 3x3 color transform and 3D bias vector to a rendered RGB image to match training-time exposure conditions [S1, unofficial]. It operates as a conditional neural network that maps a scalar condition (temperature for white balance, exposure for log-scale adjustment) to a color correction factor [S1, unofficial]. It builds on [Trained Exposure Application](13959-trained-exposure-application.md) by providing the specific parameterized functions for white balance and exposure adjustments within that framework.

## How it works
The method consists of two distinct variants: a White Balance (WB) model and an Exposure (EXP) model. Both use a Multi-Layer Perceptron (MLP) conditioned on a scalar input to predict color scaling factors.

### Inputs and Outputs
*   **Inputs**:
    *   `inputs_rgb`: A tensor of shape `(N, 3)` representing $N$ RGB pixels, where each pixel is $[r, g, b]$.
    *   `condition`: A scalar tensor of shape `(N,)` representing the conditioning variable (temperature for WB, exposure for EXP).
*   **Outputs**:
    *   `rgb_wb` or `rgb_exp`: A tensor of shape `(N, 3)` representing the corrected RGB pixels.

### Data Structures
The models are implemented as neural networks using dense layers.
*   `dense_layer`: A function created via `functools.partial(nn.Dense, kernel_init=jax.nn.initializers.glorot_uniform())`.
*   `net_depth`: Integer, depth of the MLP.
*   `net_width`: Integer, width of the MLP hidden layers.
*   `net_activation`: Activation function (e.g., ReLU).
*   `rgb_activation`: Activation function for the output layer (e.g., Sigmoid).

### White Balance Model (WbModel)
This model applies a learned scaling factor to RGB inputs based on a temperature condition.

1.  Initialize `dense_layer` with Glorot uniform initialization.
2.  Set `input_temp = condition`.
3.  Loop `i` from 0 to `self.net_depth - 1`:
    *   Apply `dense_layer(self.net_width)` to `input_temp`.
    *   Apply `self.net_activation` to the result.
4.  Predict weight `w` by applying `dense_layer(self.num_rgb_channels)` to `input_temp`.
5.  Apply `self.rgb_activation(w)` and multiply the result by 2.
6.  Compute `rgb_wb = inputs_rgb * w`.
7.  Return `rgb_wb`.

### Exposure Model (ExpModel)
This model applies a learned log-scale adjustment to RGB inputs based on an exposure condition.

1.  Initialize `dense_layer` with Glorot uniform initialization.
2.  Set `input_exp = condition`.
3.  Extract `r`, `g`, `b` from `inputs_rgb` as slices `[:, 0:1]`, `[:, 1:2]`, `[:, 2:3]`.
4.  Loop `i` from 0 to `self.net_depth - 1`:
    *   Apply `dense_layer(self.net_width)` to `input_exp`.
    *   Apply `self.net_activation` to the result.
5.  Predict weight `w` by applying `dense_layer(self.one_rgb_channels)` to `input_exp`.
6.  Apply `self.rgb_activation(w)` and multiply the result by 1.5.
7.  Compute log-adjusted channels:
    *   $r_e = r + \log(w)$
    *   $g_e = g + \log(w)$
    *   $b_e = b + \log(w)$
8.  Concatenate $r_e, g_e, b_e$ along the last axis to get `rgb_exp`.
9.  Return `rgb_exp`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
import functools
from flax.linen import Dense
from flax.linen import relu, sigmoid

def create_dense_layer():
    return functools.partial(Dense, kernel_init=jax.nn.initializers.glorot_uniform())

def wb_model(inputs_rgb, condition, net_depth=2, net_width=32, 
             net_activation=relu, rgb_activation=sigmoid, num_rgb_channels=3):
    """
    White Balance Model.
    inputs_rgb: (N, 3)
    condition: (N,)
    """
    dense_layer = create_dense_layer()
    input_temp = condition
    
    # MLP Hidden Layers
    for _ in range(net_depth):
        input_temp = dense_layer(net_width)(input_temp)
        input_temp = net_activation(input_temp)
        
    # Output Layer
    w = dense_layer(num_rgb_channels)(input_temp)
    w = rgb_activation(w)
    w = w * 2.0
    
    rgb_wb = inputs_rgb * w
    return rgb_wb

def exp_model(inputs_rgb, condition, net_depth=2, net_width=32, 
              net_activation=relu, rgb_activation=sigmoid, one_rgb_channels=1):
    """
    Exposure Model.
    inputs_rgb: (N, 3)
    condition: (N,)
    """
    dense_layer = create_dense_layer()
    input_exp = condition
    
    # Extract channels
    r = inputs_rgb[:, 0:1]
    g = inputs_rgb[:, 1:2]
    b = inputs_rgb[:, 2:3]
    
    # MLP Hidden Layers
    for _ in range(net_depth):
        input_exp = dense_layer(net_width)(input_exp)
        input_exp = net_activation(input_exp)
        
    # Output Layer
    w = dense_layer(one_rgb_channels)(input_exp)
    w = rgb_activation(w)
    w = w * 1.5
    
    # Log-scale adjustment
    r_e = r + jnp.log(w)
    g_e = g + jnp.log(w)
    b_e = b + jnp.log(w)
    
    rgb_exp = jnp.concatenate([r_e, g_e, b_e], axis=-1)
    return rgb_exp
```

### Parameters
*   **WB Model**:
    *   `net_depth`: int = 2. Depth of the first part of MLP [S1, unofficial].
    *   `net_width`: int = 32. Width of the first part of MLP [S1, unofficial].
    *   `net_activation`: Callable = `nn.relu` [S1, unofficial].
    *   `rgb_activation`: Callable = `nn.sigmoid` [S1, unofficial].
    *   `num_rgb_channels`: int = 3. Number of RGB channels [S1, unofficial].
*   **EXP Model**:
    *   `net_depth`: int = 2. Depth of the first part of MLP [S1, unofficial].
    *   `net_width`: int = 32. Width of the first part of MLP [S1, unofficial].
    *   `net_activation`: Callable = `nn.relu` [S1, unofficial].
    *   `rgb_activation`: Callable = `nn.sigmoid` [S1, unofficial].
    *   `one_rgb_channels`: int = 1. Number of RGB channels (used for scalar weight) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Logarithm Domain**: The Exposure model computes $\log(w)$. Since `w` is the output of a sigmoid activation scaled by 1.5, $w \in (0, 1.5)$. The logarithm is defined for $w > 0$. Sigmoid outputs are strictly positive, so $\log(w)$ is always defined. However, if $w$ approaches 0, $\log(w)$ approaches $-\infty$, potentially causing numerical instability or NaNs if not handled. [G] In practice, sigmoid outputs are bounded away from exact zero, but clipping `w` to a small epsilon (e.g., $10^{-7}$) before taking the log is recommended for stability.
*   **Condition Scaling**: The condition inputs (temperature, exposure) must be scaled appropriately for the network. The sources do not specify normalization steps for the condition input, implying it is used as-is or pre-normalized by the caller.

## Subtopics

### ISP
An Image Signal Processing pipeline that chains White Balance, Exposure, and CRF models [S1, unofficial]. It instantiates `wb = WbModel()`, `exp = ExpModel()`, and `crf = CrfModel()`, then applies White Balance first: `input_wb = wb(input, temperature)` [S1, unofficial].
*   **Parameters**:
    *   `net_depth`: int = 8. Depth of the first part of MLP [S1, unofficial].
    *   `net_width`: int = 256. Width of the first part of MLP [S1, unofficial].
    *   `net_depth_condition`: int = 1. Depth of the second part of MLP [S1, unofficial].
    *   `net_width_condition`: int = 128. Width of the second part of MLP [S1, unofficial].
    *   `net_activation`: Callable = `nn.relu` [S1, unofficial].
    *   `skip_layer`: int = 4. Add a skip connection to the output of every N layers [S1, unofficial].
    *   `num_rgb_channels`: int = 3. Number of RGB channels [S1, unofficial].
    *   `num_density_channels`: int = 1. Number of density channels [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of pixels $N$ and the depth of the MLP $D$. $O(N \cdot D \cdot W)$, where $W$ is the width.
*   **Memory Complexity**: $O(W \cdot D)$ for model weights.
*   **Strengths**: Simple, differentiable, and easy to integrate into neural rendering pipelines.
*   **Limitations**: Relies on learned parameters, so performance depends on the training data distribution. The log-scale adjustment in the Exposure model may be sensitive to small values of $w$.
*   **When to use**: When a learned, per-image color correction is needed to match training-time exposure conditions.
*   **When not to use**: When a simple, fixed white balance or exposure correction is sufficient, or when computational resources are extremely limited.

## Background
ExpModel and WbModel are part of the CaSE-NeRF project, which extends mip-NeRF with learned colorization models [S1, unofficial]. The code is available on GitHub [S1, unofficial].

## Sources
[S1] CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
