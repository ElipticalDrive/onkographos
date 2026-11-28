# Mip-NeRF 360 MLP Architecture

Also called: Integrated Positional Encoding MLP, PosEnc MLP

## Overview
The Mip-NeRF 360 MLP architecture is a specific configuration of the radiance field neural network that replaces standard positional encoding with **integrated positional encoding** to handle aliasing and frequency uncertainty along camera rays [S1, unofficial]. It differs from the standard NeRF MLP by lifting Gaussian distributions (means and covariances) into the positional encoding space, allowing the network to integrate high-frequency signals over the volume of a ray segment rather than at a single point [S1, unofficial]. This method is essential for 360-degree scenes with large depth variations where standard sampling causes aliasing artifacts [S1, unofficial].

## How it works
The architecture consists of two connected Multi-Layer Perceptrons (MLPs): a density network and a color network. The input to the density network is not a single 3D point, but a Gaussian distribution representing the spatial uncertainty of a ray segment.

### Inputs and Outputs
**Inputs:**
*   **Positional Statistics:** A mean vector $\mu \in \mathbb{R}^3$ and a covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ for each sample point along the ray. These are derived from the ray parameterization and the near/far bounds [S1, unofficial].
*   **View/Reflection Direction:** A direction vector $\mathbf{d} \in \mathbb{R}^3$.
*   **Normals (Optional):** Surface normals $\mathbf{n} \in \mathbb{R}^3$, computed from the density gradient if required for reflections [S1, unofficial].

**Outputs:**
*   **Density:** A scalar volume density $\sigma \in \mathbb{R}$.
*   **Color:** An RGB color vector $\mathbf{c} \in \mathbb{R}^3$.
*   **Normals (Optional):** Normalized surface normals $\mathbf{n} \in \mathbb{R}^3$.
*   **Roughness (Optional):** A scalar roughness value if enabled [S1, unofficial].

### Data Structures
The network state is defined by a configuration object containing the following fields [S1, unofficial]:

| Field | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `net_depth` | int | 8 | Depth of the density MLP. |
| `net_width` | int | 256 | Width of the density MLP hidden layers. |
| `bottleneck_width` | int | 256 | Width of the feature vector passed to the color MLP. |
| `net_depth_viewdirs` | int | 1 | Depth of the color MLP. |
| `net_width_viewdirs` | int | 128 | Width of the color MLP hidden layers. |
| `min_deg_point` | int | 0 | Minimum frequency band for positional encoding. |
| `max_deg_point` | int | 12 | Maximum frequency band for positional encoding. |
| `skip_layer` | int | 4 | Interval for skip connections in the density MLP. |
| `skip_layer_dir` | int | 4 | Interval for skip connections in the color MLP. |
| `deg_view` | int | 4 | Degree of encoding for view/reflection directions. |
| `density_bias` | float | -1.0 | Bias added to raw density before activation. |
| `density_noise` | float | 0.0 | Standard deviation of noise added to raw density. |
| `rgb_bias` | float | 0.0 | Bias added to raw color before activation. |
| `rgb_padding` | float | 0.001 | Padding applied to RGB output range. |
| `use_reflections` | bool | False | Use reflection directions instead of view directions. |
| `use_directional_enc` | bool | False | Use Integrated Directional Encoding (IDE). |
| `basis_shape` | str | 'icosahedron' | Basis shape for IDE ('icosahedron' or 'octahedron'). |
| `basis_subdivisions` | int | 2 | Tessellation count for the basis shape. |
| `enable_pred_normals` | bool | False | Enable explicit normal prediction branch. |
| `disable_density_normals` | bool | False | Disable normal computation from density gradient. |
| `use_diffuse_color` | bool | False | Predict diffuse and specular colors separately. |
| `use_specular_tint` | bool | False | Predict a tint vector for specular color. |
| `use_n_dot_v` | bool | False | Feed $\mathbf{n} \cdot \mathbf{v}$ to the color MLP. |
| `bottleneck_noise` | float | 0.0 | Noise added to the bottleneck feature vector. |
| `warp_fn` | callable | None | Function to linearize means/covs if coordinate warping is used. |

### Numbered Steps

1.  **Setup and Basis Precomputation:**
    If `use_reflections` is True, normals must be available (either via `enable_pred_normals` or by not disabling density normals); otherwise, raise `ValueError` [S1, unofficial].
    Precompute the transpose of the basis matrix $B^T$ using `geopoly.generate_basis(basis_shape, basis_subdivisions)` and store as `self.pos_basis_t` [S1, unofficial].

2.  **Directional Encoding Function Definition:**
    If `use_directional_enc` is True, generate the Integrated Directional Encoding (IDE) function using `ref_utils.generate_ide_fn(deg_view)` [S1, unofficial].
    Otherwise, define `dir_enc_fn` to use standard coordinate positional encoding with `min_deg=0`, `max_deg=deg_view`, and `append_identity=True` [S1, unofficial].

3.  **Coordinate Warping (Optional):**
    If `warp_fn` is not None, linearize the means $\mu$ and covariances $\Sigma$ using `coord.track_linearize(warp_fn, mu, Sigma)` to account for non-Euclidean coordinate transformations [S1, unofficial].

4.  **Lift and Diagonalize:**
    Lift the means and covariances into the positional encoding basis space. Compute `lifted_means` and `lifted_vars` (diagonal covariances) using `coord.lift_and_diagonalize(mu, Sigma, pos_basis_t)` [S1, unofficial]. This step projects the spatial uncertainty onto the sinusoidal frequency bands.

5.  **Integrated Positional Encoding (Density Input):**
    Compute the integrated positional encoding $\mathbf{x}$ for the density network using `coord.integrated_pos_enc(lifted_means, lifted_vars, min_deg_point, max_deg_point)` [S1, unofficial]. This replaces the standard $\sin(2^l \pi \mathbf{x})$ with an analytical integral over the Gaussian distribution, dampening high frequencies based on the covariance [S1, unofficial].

6.  **Density Network Forward Pass:**
    Initialize input $x = \mathbf{x}$.
    Iterate $i$ from $0$ to `net_depth` - 1:
    *   Apply a dense layer with width `net_width` and activation `net_activation` (typically ReLU).
    *   If $i > 0$ and $i \% \text{skip\_layer} == 0$, concatenate $x$ with the original input to this block (skip connection) along the last axis [S1, unofficial].
    *   Update $x$ with the layer output.
    Output raw density: `raw_density` = `dense_layer(1)(x)[..., 0]` [S1, unofficial].

7.  **Density Noise (Optional):**
    If `density_key` is not None and `density_noise` > 0, add Gaussian noise: `raw_density += density_noise * random.normal(density_key, raw_density.shape)` [S1, unofficial].

8.  **Normal Computation (Optional):**
    If `disable_density_normals` is False:
    *   Flatten $\mu$ and $\Sigma$ to batch dimensions.
    *   Compute the gradient of the raw density with respect to position using `jax.vmap(jax.value_and_grad(predict_density, has_aux=True))` to get `raw_density` and `raw_grad_density` simultaneously [S1, unofficial].
    *   Compute normals: $\mathbf{n} = -\text{l2\_normalize}(\text{raw\_grad\_density})$. Normals are derived from the pre-activation density for numerical stability [S1, unofficial].

9.  **Density Activation:**
    Compute final density: $\sigma = \text{density\_activation}(\text{raw\_density} + \text{density\_bias})$. The default activation is softplus [S1, unofficial].

10. **Bottleneck Feature Extraction:**
    The output of the density network's final layer (before the density head) serves as the bottleneck feature vector $\mathbf{f}_{bottleneck} \in \mathbb{R}^{\text{bottleneck\_width}}$.

11. **Directional Encoding:**
    Compute directional encoding $\mathbf{d}_{enc}$:
    *   If `use_reflections` is True, compute reflection direction $\mathbf{r} = \text{reflect}(-\mathbf{v}, \mathbf{n})$. Note: view directions $\mathbf{v}$ are flipped because they point from camera to point, while reflection utilities assume vectors pointing toward the camera [S1, unofficial].
    *   Apply `dir_enc_fn` to $\mathbf{v}$ (or $\mathbf{r}$) to get $\mathbf{d}_{enc}$.
    *   Broadcast $\mathbf{d}_{enc}$ to match the bottleneck shape dimensions: $\mathbf{d}_{enc} = \text{broadcast\_to}(\mathbf{d}_{enc}, \text{bottleneck.shape})$ [S1, unofficial].

12. **Color Network Input Construction:**
    Initialize list $x = [\mathbf{f}_{bottleneck}]$.
    *   Append $\mathbf{d}_{enc}$ to $x$.
    *   If `use_n_dot_v` is True, compute dot product $\mathbf{n} \cdot \mathbf{v}$ and append to $x$ [S1, unofficial].
    *   If `glo_vec` (global vector) is not None, broadcast it to match bottleneck shape and append to $x$ [S1, unofficial].
    *   Concatenate all elements in $x$ along the last axis [S1, unofficial].

13. **Color Network Forward Pass:**
    Iterate $i$ from $0$ to `net_depth_viewdirs` - 1:
    *   Apply a dense layer with width `net_width_viewdirs` and activation `net_activation`.
    *   If $i > 0$ and $i \% \text{skip\_layer\_dir} == 0$, concatenate $x$ with the original input to this block (skip connection) along the last axis [S1, unofficial].
    *   Update $x$ with the layer output.

14. **RGB Output and Tone Mapping:**
    Compute raw RGB: $\mathbf{c}_{raw} = \text{dense\_layer}(\text{num\_rgb\_channels})(x)$ [S1, unofficial].
    Apply activation and bias: $\mathbf{c} = \text{rgb\_activation}(\mathbf{c}_{raw} + \text{rgb\_bias})$.
    Apply padding: $\mathbf{c} = \mathbf{c} \times (1 + 2 \times \text{rgb\_padding}) - \text{rgb\_padding}$. This maps the color range to $[-\text{rgb\_padding}, 1+\text{rgb\_padding}]$ [S1, unofficial].

15. **Specular/Diffuse Split (Optional):**
    If `use_diffuse_color` is True:
    *   Compute diffuse linear: $\mathbf{c}_{diffuse} = \sigma(\mathbf{c}_{raw\_diffuse} - \ln(3.0))$.
    *   If `use_specular_tint` is True, compute specular linear: $\mathbf{c}_{specular} = \text{tint} \times \mathbf{c}$. Else, $\mathbf{c}_{specular} = 0.5 \times \mathbf{c}$ [S1, unofficial].
    *   Combine and tone map: $\mathbf{c}_{final} = \text{clip}(\text{linear\_to\_srgb}(\mathbf{c}_{specular} + \mathbf{c}_{diffuse}), 0.0, 1.0)$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax import random, vmap, value_and_grad
import numpy as np

def mipnerf360_mlp_forward(
    means,           # (batch, 3)
    covs,            # (batch, 3, 3)
    viewdirs,        # (batch, 3)
    normals,         # (batch, 3) or None
    config,          # Configuration object
    params,          # Network weights
    rng              # Random key
):
    """
    Forward pass for Mip-NeRF 360 MLP.
    Returns: density, color, normals (optional)
    """
    
    # 1. Setup and Basis
    if config.use_reflections and normals is None:
        raise ValueError("Reflections require normals")
    
    # Precomputed basis transpose (stored in model state)
    pos_basis_t = config.pos_basis_t 
    
    # 2. Directional Encoding Function
    if config.use_directional_enc:
        dir_enc_fn = generate_ide_fn(config.deg_view, config.basis_shape, config.basis_subdivisions)
    else:
        dir_enc_fn = lambda d: integrated_pos_enc_single(d, 0, config.deg_view) # Simplified for clarity

    # 3. Coordinate Warping
    if config.warp_fn is not None:
        means, covs = track_linearize(config.warp_fn, means, covs)

    # 4. Lift and Diagonalize
    lifted_means, lifted_vars = lift_and_diagonalize(means, covs, pos_basis_t)

    # 5. Integrated Positional Encoding (Density Input)
    x_density = integrated_pos_enc(lifted_means, lifted_vars, 
                                   config.min_deg_point, config.max_deg_point)

    # 6. Density Network
    x = x_density
    for i in range(config.net_depth):
        x = dense_layer(x, params['density_weights'][i], params['density_biases'][i])
        x = activation(x) # e.g., ReLU
        
        if i > 0 and i % config.skip_layer == 0:
            x = jnp.concatenate([x, x_density], axis=-1) # Skip connection logic simplified
            
    raw_density = dense_layer(x, params['density_head_w'], params['density_head_b'])[..., 0]

    # 7. Density Noise
    if config.density_noise > 0:
        noise = random.normal(rng, raw_density.shape)
        raw_density += config.density_noise * noise

    # 8. Normal Computation
    normals_out = None
    if not config.disable_density_normals:
        # Compute gradient of raw_density w.r.t means
        # Note: In practice, this uses vmap over batch and value_and_grad
        def predict_density(m):
            # Re-run steps 4-6 for single point to get gradient
            lm, lv = lift_and_diagonalize(m, covs[0], pos_basis_t)
            x = integrated_pos_enc(lm, lv, config.min_deg_point, config.max_deg_point)
            # ... forward pass ...
            return raw_density_val
        
        # Simplified gradient computation
        raw_grad_density = jnp.gradient(raw_density, means) # Placeholder for JAX vmap/grad
        normals_out = -l2_normalize(raw_grad_density)

    # 9. Density Activation
    density = softplus(raw_density + config.density_bias)

    # 10. Bottleneck Extraction
    # Assume bottleneck is the output of the last hidden layer of density net
    bottleneck = x # Simplified: in real impl, extract before density head

    # 11. Directional Encoding
    if config.use_reflections:
        # Flip viewdirs because they point camera->point
        viewdirs_toward = -viewdirs
        refdirs = reflect(viewdirs_toward, normals_out)
        dir_enc = dir_enc_fn(refdirs)
    else:
        dir_enc = dir_enc_fn(viewdirs)

    # Broadcast dir_enc to match bottleneck shape
    dir_enc = jnp.broadcast_to(dir_enc[..., None, :], bottleneck.shape[:-1] + (dir_enc.shape[-1],))

    # 12. Color Network Input
    x_color = [bottleneck]
    x_color.append(dir_enc)
    
    if config.use_n_dot_v and normals_out is not None:
        n_dot_v = jnp.sum(normals_out * viewdirs[..., None, :], axis=-1, keepdims=True)
        x_color.append(n_dot_v)
        
    if config.glo_vec is not None:
        glo_vec = jnp.broadcast_to(config.glo_vec, bottleneck.shape)
        x_color.append(glo_vec)
        
    x_color = jnp.concatenate(x_color, axis=-1)

    # 13. Color Network
    for i in range(config.net_depth_viewdirs):
        x_color = dense_layer(x_color, params['color_weights'][i], params['color_biases'][i])
        x_color = activation(x_color)
        
        if i > 0 and i % config.skip_layer_dir == 0:
            x_color = jnp.concatenate([x_color, x_color], axis=-1) # Skip connection

    # 14. RGB Output
    raw_rgb = dense_layer(x_color, params['color_head_w'], params['color_head_b'])
    rgb = activation(raw_rgb + config.rgb_bias)
    rgb = rgb * (1 + 2 * config.rgb_padding) - config.rgb_padding

    # 15. Specular/Diffuse Split (Optional)
    if config.use_diffuse_color:
        # Simplified logic
        diffuse_linear = sigmoid(raw_rgb - jnp.log(3.0))
        if config.use_specular_tint:
            specular_linear = config.tint * rgb
        else:
            specular_linear = 0.5 * rgb
        rgb = jnp.clip(linear_to_srgb(specular_linear + diffuse_linear), 0.0, 1.0)

    return density, rgb, normals_out
```

### Parameters with Typical Values
*   `net_depth`: 8
*   `net_width`: 256
*   `bottleneck_width`: 256
*   `net_depth_viewdirs`: 1
*   `net_width_viewdirs`: 128
*   `min_deg_point`: 0
*   `max_deg_point`: 12
*   `skip_layer`: 4
*   `deg_view`: 4
*   `density_bias`: -1.0
*   `rgb_padding`: 0.001

### Edge Cases and Pitfalls
*   **Reflections Require Normals:** If `use_reflections` is True, normals must be computed. If `enable_pred_normals` is False and `disable_density_normals` is True, a `ValueError` is raised [S1, unofficial].
*   **View Direction Flip:** When computing reflection directions, view directions must be flipped (`-viewdirs`) before passing to the reflection function because standard view directions point from the camera to the point, while reflection utilities typically assume vectors pointing toward the camera [S1, unofficial].
*   **Normal Stability:** Normals are computed from the gradient of the **raw** (pre-activation) density. Using the post-activation density (e.g., after softplus) can lead to numerical instability due to steep or flat gradients [S1, unofficial].
*   **Precision:** Color correction and metric computation should use 64-bit floating point precision to ensure high accuracy, especially when applying color correction functions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The integrated positional encoding step adds computational overhead compared to standard positional encoding due to the calculation of the exponential term involving the covariance trace. However, it reduces the need for excessive sampling density to avoid aliasing.
*   **Memory Complexity:** The architecture stores the same number of parameters as the standard NeRF MLP (approx. 8-9 million for default settings). The covariance matrices ($\Sigma$) are stored per sample point during rendering but are not part of the model weights.
*   **Strengths:** Significantly reduces aliasing artifacts in scenes with large depth variations and wide fields of view (360-degree scenes). It provides a theoretically grounded way to handle the frequency content of the signal along a ray.
*   **Limitations:** Slightly higher computational cost per sample due to the integration step. Requires careful tuning of the `min_deg_point` and `max_deg_point` parameters.
*   **When to Use:** Use for 360-degree scenes or scenes with significant depth variation where standard NeRF produces aliasing. Not necessary for small, bounded scenes with uniform depth.

## Background
The Mip-NeRF 360 MLP architecture was introduced by Ben Mildenhall et al. in the paper "Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields" (2022) [S1, unofficial]. It builds upon the original Mip-NeRF (2021) by extending the integrated positional encoding technique to unbounded, 360-degree scenes. The method addresses the limitation of standard NeRF, which assumes a narrow perspective projection and suffers from aliasing when applied to wide-angle or spherical imagery. The implementation is part of the `google-research/multinerf` repository [S1, unofficial].

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
