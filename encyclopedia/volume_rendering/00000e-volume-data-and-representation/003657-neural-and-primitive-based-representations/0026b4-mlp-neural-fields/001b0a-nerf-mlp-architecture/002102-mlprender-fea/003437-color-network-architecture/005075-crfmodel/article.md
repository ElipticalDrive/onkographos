# CrfModel

Also called: Color Response Function Model

## Overview
The CrfModel is a color network architecture component used in neural radiance fields that applies independent, learned non-linear transformations to each RGB channel of the input color vector [S1, unofficial]. It differs from sibling architectures like the Neural Textures Architecture or Intrinsic Color Decomposition Network by processing channels separately rather than jointly, allowing for channel-specific response functions [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A tensor `x` of shape `(batch_size, 3)` containing the raw RGB values.
- Output: A tensor `rgb_out` of shape `(batch_size, 3)` containing the transformed RGB values.

**Data Structures**
The model is implemented as a class with the following parameters:
- `net_depth`: Integer, depth of the first MLP part. Default: 1.
- `net_width`: Integer, width of the first MLP part. Default: 256.
- `net_activation`: Activation function for the first part. Default: ReLU.
- `rgb_activation`: Activation function for the final output. Default: Sigmoid.
- `one_rgb_channels`: Integer, number of output channels per RGB channel. Default: 1.

**Steps**
1. Initialize a dense layer factory `dense_layer` using `functools.partial` with `nn.Dense` and `jax.nn.initializers.glorot_uniform()` initialization [S1, unofficial].
2. Set `inputs_rgb = x` [S1, unofficial].
3. Extract individual channel slices:
   - `r_e = inputs_rgb[:, 0:1]`
   - `g_e = inputs_rgb[:, 1:2]`
   - `b_e = inputs_rgb[:, 2:3]`
4. Process the Red channel:
   - Loop `i` from 0 to `net_depth - 1`:
     - `r_e = dense_layer(net_width)(r_e)`
     - `r_e = net_activation(r_e)`
   - `r_e = dense_layer(one_rgb_channels)(r_e)`
   - `r_e = rgb_activation(r_e)`
5. Process the Green channel:
   - Loop `i` from 0 to `net_depth - 1`:
     - `g_e = dense_layer(net_width)(g_e)`
     - `g_e = net_activation(g_e)`
   - `g_e = dense_layer(one_rgb_channels)(g_e)`
   - `g_e = rgb_activation(g_e)`
6. Process the Blue channel:
   - Loop `i` from 0 to `net_depth - 1`:
     - `b_e = dense_layer(net_width)(b_e)`
     - `b_e = net_activation(b_e)`
   - `b_e = dense_layer(one_rgb_channels)(b_e)`
   - `b_e = rgb_activation(b_e)`
7. Concatenate `r_e`, `g_e`, and `b_e` along the last axis to form `rgb_out` [S1, unofficial].
8. Return `rgb_out` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import functools
import jax
import jax.nn as nn
import jax.numpy as jnp

class CrfModel:
    def __init__(self, net_depth=1, net_width=256, net_activation=nn.relu, 
                 rgb_activation=nn.sigmoid, one_rgb_channels=1):
        self.net_depth = net_depth
        self.net_width = net_width
        self.net_activation = net_activation
        self.rgb_activation = rgb_activation
        self.one_rgb_channels = one_rgb_channels
        # Initialize dense layer factory
        self.dense_layer = functools.partial(
            nn.Dense, 
            kernel_init=jax.nn.initializers.glorot_uniform()
        )

    def __call__(self, x):
        # x shape: (batch_size, 3)
        inputs_rgb = x
        
        # Extract channels
        r_e = inputs_rgb[:, 0:1]
        g_e = inputs_rgb[:, 1:2]
        b_e = inputs_rgb[:, 2:3]
        
        # Process Red
        for _ in range(self.net_depth):
            r_e = self.dense_layer(self.net_width)(r_e)
            r_e = self.net_activation(r_e)
        r_e = self.dense_layer(self.one_rgb_channels)(r_e)
        r_e = self.rgb_activation(r_e)
        
        # Process Green
        for _ in range(self.net_depth):
            g_e = self.dense_layer(self.net_width)(g_e)
            g_e = self.net_activation(g_e)
        g_e = self.dense_layer(self.one_rgb_channels)(g_e)
        g_e = self.rgb_activation(g_e)
        
        # Process Blue
        for _ in range(self.net_depth):
            b_e = self.dense_layer(self.net_width)(b_e)
            b_e = self.net_activation(b_e)
        b_e = self.dense_layer(self.one_rgb_channels)(b_e)
        b_e = self.rgb_activation(b_e)
        
        # Concatenate
        rgb_out = jnp.concatenate([r_e, g_e, b_e], axis=-1)
        return rgb_out
```

**Parameters**
- `net_depth`: 1 [S1, unofficial].
- `net_width`: 256 [S1, unofficial].
- `net_activation`: ReLU [S1, unofficial].
- `rgb_activation`: Sigmoid [S1, unofficial].
- `one_rgb_channels`: 1 [S1, unofficial].

**Edge Cases and Pitfalls**
- The model assumes the input tensor has at least 3 channels. If the input is not RGB, slicing may fail or produce incorrect results [G].
- The use of Sigmoid activation at the output constrains the output values to [0, 1], which is suitable for normalized RGB values but may require scaling if input values are outside this range [G].

## Subtopics

### Neural Textures Architecture
- **What it is**: An architecture that uses neural networks to generate texture maps for 3D objects.
- **Difference**: Processes spatial data jointly rather than per-channel independent transformations.
- **When to choose**: When generating spatially coherent textures for surfaces.
- **Link**: [Neural Textures Architecture](9534-neural-textures-architecture.md)

### Intrinsic Color Decomposition Network
- **What it is**: A network that decomposes color into intrinsic components like albedo and shading.
- **Difference**: Separates physical properties rather than applying independent channel-wise non-linearities.
- **When to choose**: When disentangling lighting and material properties is required.
- **Link**: [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)

### Decomposition Network
- **What it is**: A general network for decomposing input signals into components.
- **Difference**: May use joint processing or different decomposition strategies.
- **When to choose**: When a more general decomposition framework is needed.
- **Link**: [Decomposition Network](12532-decomposition-network.md)

### Squeeze-and-Excitation Module
- **What it is**: A module that recalibrates channel-wise feature responses.
- **Difference**: Uses global pooling and gating to modulate channels based on interdependencies, unlike independent transformations.
- **When to choose**: When channel interdependencies are important to model.
- **Link**: [Squeeze-and-Excitation Module](14050-squeeze-and-excitation-module.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to `net_depth` and `net_width` for each channel. Since channels are processed independently, the total time is roughly 3 times the time for a single channel.
- **Memory Complexity**: Proportional to the number of parameters in the dense layers. For default settings, this is minimal.
- **Strengths**: Simple to implement, computationally efficient, allows for channel-specific adjustments.
- **Limitations**: Does not model inter-channel correlations, which may be important for color consistency.
- **When to use**: When channel-specific response functions are desired and computational efficiency is a priority.
- **When not to use**: When modeling complex color interactions between channels is necessary.

## Background
The CrfModel is part of the CaSE-NeRF implementation, an unofficial extension of mip-NeRF [S1, unofficial]. It was introduced to provide a flexible way to model color response functions in neural radiance fields. The architecture is inspired by the need for independent channel processing in some rendering scenarios.

## Sources
[S1] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
