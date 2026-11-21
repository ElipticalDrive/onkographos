# Mip-NeRF 360 MLP Architecture

Also called: Integrated Positional Encoding MLP, PosEnc MLP

## Overview
The Mip-NeRF 360 MLP architecture is a specific configuration of the radiance field neural network that utilizes Integrated Positional Encoding (IPE) to handle aliasing and frequency limitations inherent in standard positional encoding [S1, unofficial]. Unlike the standard NeRF MLP which maps raw or standard positional encoded coordinates directly to density and color, this architecture lifts the mean and covariance of the integrated Gaussian query distribution through a basis transformation before encoding, allowing the network to approximate the integral of the positional encoding function over the ray segment [S1, unofficial]. It is distinguished by its support for advanced appearance modeling, including specular tinting, roughness prediction, and reflection directions, while maintaining the core two-branch (density and color) structure [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Position:** $\mathbf{x} \in \mathbb{R}^3$, the 3D coordinates of the sample point.
*   **Covariance:** $\Sigma \in \mathbb{R}^{3 \times 3}$, the covariance matrix of the Gaussian distribution along the ray segment at $\mathbf{x}$, representing the integrated volume [S1, unofficial].
*   **View Direction:** $\mathbf{v} \in \mathbb{R}^3$, the normalized direction from the camera to the point [S1, unofficial].
*   **Reflection Direction:** $\mathbf{r} \in \mathbb{R}^3$, optional, used if `use_reflections` is True [S1, unofficial].
*   **Normals:** $\mathbf{n} \in \mathbb{R}^3$, optional, required if `use_reflections` is True [S1, unofficial].
*   **GLO Vector:** $\mathbf{g} \in \mathbb{R}^k$, optional, global appearance vector [S1, unofficial].

**Outputs:**
*   **Density:** $\sigma \in \mathbb{R}$, the volume density.
*   **Color:** $\mathbf{c} \in \mathbb{R}^3$, the RGB color.
*   **Normals:** $\mathbf{n}_{pred} \in \mathbb{R}^3$, optional, predicted surface normals.
*   **Roughness:** $\rho \in \mathbb{R}$, optional, predicted roughness.

### Data Structures and Parameters
The architecture is defined by a configuration object containing the following parameters [S1, unofficial]:

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `net_depth` | int | 8 | Depth of the density MLP. |
| `net_width` | int | 256 | Width of the density MLP. |
| `bottleneck_width` | int | 256 | Width of the bottleneck vector connecting density and color branches. |
| `net_depth_viewdirs` | int | 1 | Depth of the color MLP. |
| `net_width_viewdirs` | int | 128 | Width of the color MLP. |
| `min_deg_point` | int | 0 | Minimum degree of positional encoding for points. |
| `max_deg_point` | int | 12 | Maximum degree of positional encoding for points. |
| `skip_layer` | int | 4 | Skip connection interval for density MLP. |
| `skip_layer_dir` | int | 4 | Skip connection interval for color MLP. |
| `deg_view` | int | 4 | Degree of encoding for view/reflection directions. |
| `density_bias` | float | -1.0 | Bias added to raw density before activation. |
| `density_noise` | float | 0.0 | Standard deviation of noise added to raw density. |
| `rgb_bias` | float | 0.0 | Bias added to raw color before activation. |
| `rgb_padding` | float | 0.001 | Padding for RGB output range. |
| `use_reflections` | bool | False | Enable reflection direction encoding. |
| `use_directional_enc` | bool | False | Enable Integrated Directional Encoding (IDE). |
| `enable_pred_roughness` | bool | False | Enable roughness prediction. |
| `roughness_bias` | float | -1.0 | Bias for raw roughness. |
| `use_diffuse_color` | bool | False | Split color into diffuse/specular. |
| `use_specular_tint` | bool | False | Predict specular tint vector. |
| `use_n_dot_v` | bool | False | Include $\mathbf{n} \cdot \mathbf{v}$ in color input. |
| `bottleneck_noise` | float | 0.0 | Noise added to bottleneck vector. |
| `basis_shape` | str | 'icosahedron' | Basis shape for IPE ('octahedron' or 'icosahedron'). |
| `basis_subdivisions` | int | 2 | Tessellation count for basis shape. |
| `warp_fn` | callable | None | Warping function for linearization. |
| `density_activation` | callable | Softplus | Activation for density. |
| `rgb_activation` | callable | Sigmoid | Activation for RGB. |
| `net_activation` | callable | ReLU | Activation for hidden layers. |

### Algorithm Steps

1.  **Setup and Basis Generation:**
    Generate the transpose of the basis matrix $B^T$ using `geopoly.generate_basis(basis_shape, basis_subdivisions)`. Store as `self.pos_basis_t` [S1, unofficial].
    If `use_directional_enc` is True, generate the Integrated Directional Encoding (IDE) function using `ref_utils.generate_ide_fn(deg_view)`. Otherwise, define `dir_enc_fn` using standard positional encoding with `min_deg=0`, `max_deg=deg_view`, and `append_identity=True` [S1, unofficial].

2.  **Linearization (Optional):**
    If `warp_fn` is not None, linearize the means and covariances:
    $$ (\mu_{lin}, \Sigma_{lin}) = \text{coord.track\_linearize}(\text{warp\_fn}, \mu, \Sigma) $$
    [S1, unofficial]

3.  **Lift and Diagonalize:**
    Lift the means $\mu$ and diagonalize the covariances $\Sigma$ using the precomputed basis transpose $B^T$:
    $$ \mu_{lifted}, \Sigma_{lifted} = \text{coord.lift\_and\_diagonalize}(\mu, \Sigma, B^T) $$
    This yields the lifted means and variances required for integrated positional encoding [S1, unofficial].

4.  **Integrated Positional Encoding (Density Input):**
    Compute the integrated positional encoding for the lifted statistics:
    $$ \mathbf{x}_{in} = \text{coord.integrated\_pos\_enc}(\mu_{lifted}, \Sigma_{lifted}, \text{min\_deg\_point}, \text{max\_deg\_point}) $$
    [S1, unofficial]

5.  **Density Network Forward Pass:**
    Initialize $x = \mathbf{x}_{in}$.
    Iterate $i$ from $0$ to `net_depth` - 1:
    *   Apply dense layer: $x = \text{net\_activation}(W_i x + b_i)$, where $W_i$ has shape `(net_width, current_dim)` and $b_i$ has shape `(net_width,)`.
    *   If $i > 0$ and $i \% \text{skip\_layer} == 0$:
        $$ x = \text{concat}(x, \mathbf{x}_{in}) $$
    [S1, unofficial]

6.  **Raw Density Output:**
    $$ \sigma_{raw} = \text{dense\_layer}_{\sigma}(x)[..., 0] $$
    where the final dense layer has width 1 [S1, unofficial].

7.  **Density Noise (Optional):**
    If `density_noise` > 0:
    $$ \sigma_{raw} \leftarrow \sigma_{raw} + \text{density\_noise} \cdot \mathcal{N}(0, 1) $$
    [S1, unofficial]

8.  **Normal Computation (Optional):**
    If `disable_density_normals` is False:
    *   Flatten $\mu$ and $\Sigma$ to match the batch shape.
    *   Compute the gradient of the raw density with respect to the input coordinates. This is typically done using automatic differentiation (e.g., `jax.value_and_grad`) on the density prediction function [S1, unofficial].
    *   Let $\nabla \sigma_{raw}$ be the gradient.
    *   Compute normals:
        $$ \mathbf{n}_{pred} = -\frac{\nabla \sigma_{raw}}{\|\nabla \sigma_{raw}\|_2} $$
    *   **Pitfall:** Normals are computed from the *raw* (pre-activation) density because it is numerically more stable than post-activation density when the activation function has a steep or flat gradient [S1, unofficial].

9.  **Final Density Activation:**
    $$ \sigma = \text{density\_activation}(\sigma_{raw} + \text{density\_bias}) $$
    [S1, unofficial]

10. **Directional Encoding:**
    *   Determine the direction vector $\mathbf{d}_{dir}$:
        *   If `use_reflections` is True:
            *   **Pitfall:** View directions point from camera to point. Reflection utilities often assume directions point toward the camera. Flip view directions: $\mathbf{v}_{flip} = -\mathbf{v}$.
            *   Compute reflection: $\mathbf{r} = \text{ref\_utils.reflect}(\mathbf{v}_{flip}, \mathbf{n}_{pred})$.
            *   Set $\mathbf{d}_{dir} = \mathbf{r}$.
        *   Else:
            *   Set $\mathbf{d}_{dir} = \mathbf{v}$.
    *   Compute directional encoding:
        $$ \mathbf{e}_{dir} = \text{dir\_enc\_fn}(\mathbf{d}_{dir}) $$
    *   Broadcast $\mathbf{e}_{dir}$ to match the bottleneck shape dimensions:
        $$ \mathbf{e}_{dir} = \text{broadcast\_to}(\mathbf{e}_{dir}, \text{bottleneck\_shape}[:-1] + (\text{e}_{dir}.shape[-1],)) $$
    [S1, unofficial]

11. **Color Network Input Construction:**
    Initialize list $x_{color} = [\text{bottleneck\_features}]$.
    *   Append $\mathbf{e}_{dir}$ to $x_{color}$.
    *   If `use_n_dot_v` is True:
        $$ \text{ndv} = \sum (\mathbf{n}_{pred} \cdot \mathbf{v}) $$
        Append $\text{ndv}$ to $x_{color}$.
    *   If `glo_vec` is not None:
        Broadcast `glo_vec` to match bottleneck shape and append to $x_{color}$.
    *   Concatenate all elements in $x_{color}$ along the last axis to form $\mathbf{x}_{color\_in}$ [S1, unofficial].

12. **Color Network Forward Pass:**
    Initialize $x = \mathbf{x}_{color\_in}$.
    Iterate $i$ from $0$ to `net_depth_viewdirs` - 1:
    *   Apply dense layer: $x = \text{net\_activation}(W_i x + b_i)$, where $W_i$ has shape `(net_width_viewdirs, current_dim)`.
    *   If $i > 0$ and $i \% \text{skip\_layer\_dir} == 0$:
        $$ x = \text{concat}(x, \mathbf{x}_{color\_in}) $$
    [S1, unofficial]

13. **Raw Color Output:**
    $$ \mathbf{c}_{raw} = \text{dense\_layer}_{\text{rgb}}(x) $$
    where the final dense layer has width `num_rgb_channels` (typically 3) [S1, unofficial].

14. **Specular/Diffuse Split (Optional):**
    If `use_diffuse_color` is True:
    *   Compute diffuse linear:
        $$ \mathbf{c}_{diffuse\_lin} = \sigma(\mathbf{c}_{raw\_diffuse} - \ln(3.0)) $$
    *   If `use_specular_tint` is True:
        $$ \mathbf{c}_{specular\_lin} = \text{tint} \cdot \mathbf{c}_{raw} $$
    *   Else:
        $$ \mathbf{c}_{specular\_lin} = 0.5 \cdot \mathbf{c}_{raw} $$
    *   Combine and tone map:
        $$ \mathbf{c} = \text{clip}(\text{linear\_to\_srgb}(\mathbf{c}_{specular\_lin} + \mathbf{c}_{diffuse\_lin}), 0.0, 1.0) $$
    [S1, unofficial]

15. **Final Color Output:**
    If not using split:
    $$ \mathbf{c} = \text{rgb\_activation}(\text{rgb\_premultiplier} \cdot \mathbf{c}_{raw} + \text{rgb\_bias}) $$
    Apply padding:
    $$ \mathbf{c} = \mathbf{c} \cdot (1 + 2 \cdot \text{rgb\_padding}) - \text{rgb\_padding} $$
    [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
import jax
from jax import random

def mipnerf360_mlp_forward(
    params,
    coords,  # Contains means (mu) and covariances (Sigma)
    viewdirs,
    normals=None,
    glo_vec=None,
    config=None,
    rng=None
):
    """
    Forward pass for Mip-NeRF 360 MLP Architecture.
    
    Args:
        params: Dictionary of network weights and biases.
        coords: Object containing 'means' (mu) and 'covs' (Sigma).
        viewdirs: View directions (camera to point).
        normals: Surface normals (required if use_reflections).
        glo_vec: Global appearance vector.
        config: Configuration object with hyperparameters.
        rng: JAX random key.
        
    Returns:
        Dictionary containing 'density', 'rgb', 'normals' (optional), 'roughness' (optional).
    """
    
    # 1. Setup Basis
    # Precomputed in __init__, accessed via config or params
    pos_basis_t = config.pos_basis_t 
    
    # 2. Linearization (Optional)
    if config.warp_fn is not None:
        mu_lin, Sigma_lin = coords.track_linearize(config.warp_fn, coords.means, coords.covs)
    else:
        mu_lin, Sigma_lin = coords.means, coords.covs
        
    # 3. Lift and Diagonalize
    lifted_means, lifted_vars = coords.lift_and_diagonalize(
        mu_lin, Sigma_lin, pos_basis_t
    )
    
    # 4. Integrated Positional Encoding
    x_in = coords.integrated_pos_enc(
        lifted_means, lifted_vars, 
        config.min_deg_point, config.max_deg_point
    )
    
    # 5. Density Network
    x = x_in
    for i in range(config.net_depth):
        # Dense layer
        W = params[f'density_dense_{i}']['kernel']
        b = params[f'density_dense_{i}']['bias']
        x = jnp.dot(x, W) + b
        x = config.net_activation(x)
        
        # Skip connection
        if i > 0 and i % config.skip_layer == 0:
            x = jnp.concatenate([x, x_in], axis=-1)
            
    # 6. Raw Density
    W_out = params['density_dense_out']['kernel']
    b_out = params['density_dense_out']['bias']
    sigma_raw = jnp.dot(x, W_out[..., 0]) + b_out[0]
    
    # 7. Density Noise
    if config.density_noise > 0 and rng is not None:
        noise = random.normal(rng, sigma_raw.shape)
        sigma_raw += config.density_noise * noise
        
    # 8. Normal Computation (Optional)
    normals_pred = None
    if not config.disable_density_normals:
        # Compute gradient of raw density w.r.t. input coordinates
        # Note: In JAX, this requires the input coordinates to be tracked
        # For simplicity, assuming a helper function exists that computes this
        # gradient using the original (non-lifted) coordinates if needed,
        # or the lifted ones depending on implementation details.
        # The source specifies flattening means/covs and using vmap/value_and_grad.
        
        # Placeholder for the actual gradient computation logic which depends
        # on the specific 'coords' object implementation.
        # raw_grad_density = jax.grad(predict_density, argnums=0)(coords.means, ...)
        
        # For the sake of self-contained pseudocode, we assume a function
        # compute_normals_from_raw_density exists that returns the gradient.
        raw_grad_density = compute_normals_from_raw_density(
            params, coords.means, config
        )
        
        # Normalize
        normals_pred = -jax.nn.normalize(raw_grad_density, axis=-1)
        
    # 9. Final Density
    sigma = config.density_activation(sigma_raw + config.density_bias)
    
    # 10. Directional Encoding
    if config.use_reflections:
        if normals_pred is None:
            raise ValueError("Normals are required for reflections")
        # Flip viewdirs because they point camera->point, but reflect assumes point->camera
        viewdirs_flip = -viewdirs
        refdirs = ref_utils.reflect(viewdirs_flip, normals_pred)
        dir_input = refdirs
    else:
        dir_input = viewdirs
        
    if config.use_directional_enc:
        dir_enc = ref_utils.generate_ide_fn(config.deg_view)(dir_input)
    else:
        dir_enc = coord.pos_enc(
            dir_input, 
            min_deg=0, 
            max_deg=config.deg_view, 
            append_identity=True
        )
        
    # Broadcast dir_enc
    bottleneck_shape = params[f'density_dense_{config.net_depth-1}']['kernel'].shape[0] # net_width
    # Actually, bottleneck is the output of the density network.
    # The source says: "Broadcast dir_enc to match bottleneck shape dimensions"
    # This implies broadcasting to the batch dimensions of the bottleneck vector.
    # Assuming x is the bottleneck vector here.
    dir_enc = jnp.broadcast_to(
        dir_enc[..., None, :], 
        x.shape[:-1] + (dir_enc.shape[-1],)
    )
    
    # 11. Color Network Input
    x_color_list = [x] # Bottleneck features
    x_color_list.append(dir_enc)
    
    if config.use_n_dot_v:
        if normals_pred is None:
            raise ValueError("Normals are required for n_dot_v")
        ndv = jnp.sum(normals_pred * viewdirs[..., None, :], axis=-1, keepdims=True)
        x_color_list.append(ndv)
        
    if glo_vec is not None:
        glo_vec_broadcast = jnp.broadcast_to(
            glo_vec[..., None, :], 
            x.shape[:-1] + (glo_vec.shape[-1],)
        )
        x_color_list.append(glo_vec_broadcast)
        
    x_color_in = jnp.concatenate(x_color_list, axis=-1)
    
    # 12. Color Network
    x_color = x_color_in
    for i in range(config.net_depth_viewdirs):
        W = params[f'color_dense_{i}']['kernel']
        b = params[f'color_dense_{i}']['bias']
        x_color = jnp.dot(x_color, W) + b
        x_color = config.net_activation(x_color)
        
        if i > 0 and i % config.skip_layer_dir == 0:
            x_color = jnp.concatenate([x_color, x_color_in], axis=-1)
            
    # 13. Raw Color
    W_rgb = params['color_dense_out']['kernel']
    b_rgb = params['color_dense_out']['bias']
    c_raw = jnp.dot(x_color, W_rgb) + b_rgb
    
    # 14. Specular/Diffuse Split (Optional)
    if config.use_diffuse_color:
        # Assuming c_raw is split or has separate outputs for diffuse/specular
        # The source implies c_raw_diffuse and c_raw_specular might be separate
        # or derived. For simplicity, assuming c_raw contains diffuse part
        # and specular is derived or also in c_raw.
        # Source: "diffuse_linear = nn.sigmoid(raw_rgb_diffuse - jnp.log(3.0))"
        # This suggests raw_rgb_diffuse is a specific output.
        # If not implemented, we assume standard path.
        c_diffuse_lin = jax.nn.sigmoid(c_raw[..., :3] - jnp.log(3.0))
        if config.use_specular_tint:
            # Tint is a vector, c_raw is color
            c_specular_lin = config.tint * c_raw
        else:
            c_specular_lin = 0.5 * c_raw
        c = jnp.clip(
            image.linear_to_srgb(c_specular_lin + c_diffuse_lin), 
            0.0, 1.0
        )
    else:
        # 15. Final Color
        c = config.rgb_activation(
            config.rgb_premultiplier * c_raw + config.rgb_bias
        )
        c = c * (1 + 2 * config.rgb_padding) - config.rgb_padding
        
    result = {
        'density': sigma,
        'rgb': c
    }
    
    if normals_pred is not None:
        result['normals'] = normals_pred
        
    if config.enable_pred_roughness:
        # Roughness prediction logic would follow similar pattern
        # Output from a specific layer or head
        roughness_raw = params['roughness_dense_out']['kernel'] # Placeholder
        roughness = config.roughness_activation(roughness_raw + config.roughness_bias)
        result['roughness'] = roughness
        
    return result
```

## Operating characteristics

*   **Time Complexity:** The forward pass is dominated by the matrix multiplications in the MLPs. With `net_depth=8` and `net_width=256`, the density network has approximately $8 \times 256 \times D$ operations, where $D$ is the dimension of the integrated positional encoding (typically $2 \times L \times 3$). The color network is smaller. The IPE computation adds overhead compared to standard positional encoding due to the lifting and diagonalization steps.
*   **Memory Complexity:** The model stores weights for two MLPs. The density network weights are roughly $8 \times 256 \times 256$ (plus input/output layers). The color network weights are smaller. The basis matrix $B^T$ is small and precomputed.
*   **Performance:** Mip-NeRF 360 achieves higher quality renders, particularly in high-frequency details and anti-aliasing, compared to standard NeRF, especially in unbounded scenes. It handles large depth ranges better due to the integrated encoding.
*   **Strengths:**
    *   Reduces aliasing artifacts.
    *   Supports advanced appearance models (specular, roughness, reflections).
    *   Robust to varying sampling densities along rays.
*   **Limitations:**
    *   More complex implementation than standard NeRF.
    *   Requires computation of normals for reflections, which adds computational cost and potential instability if not handled carefully (using raw density gradients).
    *   Hyperparameter tuning (e.g., `deg_view`, `basis_shape`) can be sensitive.

## Background

The Mip-NeRF 360 MLP architecture was introduced in the "Mip-NeRF 360" paper by Barron et al. (2022) [S1, unofficial]. It builds upon the original Mip-NeRF concept of integrating positional encoding over conical frustums to handle aliasing. The "360" variant extends this to unbounded scenes and introduces features like reflection directions and roughness prediction, making it suitable for a wider range of real-world captures. It is implemented in the `google-research/multinerf` repository [S1, unofficial].

## Sources

[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
