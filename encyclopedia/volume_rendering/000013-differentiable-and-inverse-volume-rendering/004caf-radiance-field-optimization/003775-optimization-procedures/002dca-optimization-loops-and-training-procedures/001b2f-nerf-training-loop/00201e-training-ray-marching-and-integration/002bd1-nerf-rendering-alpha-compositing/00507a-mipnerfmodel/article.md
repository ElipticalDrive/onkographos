# MipNerfModel

Also called: Mip-NeRF 360 Model

## Overview
The MipNerfModel is a Neural Radiance Field (NeRF) architecture that approximates integrated positional encoding by treating rays as cones or cylinders rather than infinitesimally thin lines, thereby solving aliasing artifacts caused by high-frequency scene details [S1, unofficial]. It distinguishes itself from standard NeRF implementations by using multi-level sampling (coarse and fine) and integrated positional encoding to model the frequency content of the signal along the ray, which is essential for training on unbounded, outdoor scenes [S1, unofficial].

## How it works
The model implements a differentiable rendering pipeline that processes rays defined by origins, directions, radii, and viewing parameters. It utilizes a Multi-Layer Perceptron (MLP) to predict density and color, conditioned on integrated positional encodings of the sampled points.

### Inputs and Outputs
**Inputs:**
*   `rays`: A structure containing:
    *   `origins`: 3D coordinates of ray origins.
    *   `directions`: Normalized ray direction vectors.
    *   `viewdirs`: View direction vectors (often identical to `directions` in standard setups, but explicitly passed for conditioning).
    *   `radii`: The radius of the ray at the sample points (used for integrated encoding).
    *   `near`: Near plane distance for each ray.
    *   `far`: Far plane distance for each ray.
    *   `temperature`: Camera temperature parameter for ISP.
    *   `exposure`: Camera exposure parameter for ISP.
    *   `coc`: Circle of Confusion parameter (used to scale ray radii during training).
*   `rng`: Random number generator state.
*   `white_bkgd`: Boolean indicating if the background is white (affects alpha compositing).

**Outputs:**
*   `comp_rgb`: Computed RGB color per ray.
*   `distance`: Expected distance along the ray.
*   `acc`: Accumulated alpha (opacity) along the ray.

### Data Structures
The model maintains internal state for the MLP weights and biases. The `rays` object is typically a JAX `NamedTuple` or similar structure holding batched arrays:
*   `origins`: Shape `(B, 3)`
*   `directions`: Shape `(B, 3)`
*   `viewdirs`: Shape `(B, 3)`
*   `radii`: Shape `(B,)` or `(B, N)` depending on implementation stage (initially `(B,)` representing the radius at the pixel, scaled by `coc`).
*   `near`: Shape `(B,)`
*   `far`: Shape `(B,)`

### Parameters
*   `num_samples`: int, number of samples per level. Default: 128 [S1, unofficial].
*   `num_levels`: int, number of sampling levels (coarse and fine). Default: 2 [S1, unofficial].
*   `resample_padding`: float, Dirichlet/alpha padding added to weights before normalization during resampling. Default: 0.01 [S1, unofficial].
*   `stop_level_grad`: bool, if True, gradients do not backpropagate across levels. Default: True [S1, unofficial].
*   `use_viewdirs`: bool, if True, view directions are used as a condition for the MLP. Default: True [S1, unofficial].
*   `lindisp`: bool, if True, sampling is linear in disparity ($1/z$) rather than depth ($z$). Default: False [S1, unofficial].
*   `ray_shape`: str, shape of cast rays, must be 'cone' or 'cylinder'. Default: 'cone' [S1, unofficial].
*   `min_deg_point`: int, minimum degree of positional encoding for 3D points. Default: 0 [S1, unofficial].
*   `max_deg_point`: int, maximum degree of positional encoding for 3D points. Default: 16 [S1, unofficial].
*   `deg_view`: int, degree of positional encoding for view directions. Default: 4 [S1, unofficial].
*   `density_activation`: Callable, density activation function. Default: `softplus` [S1, unofficial].
*   `density_noise`: float, standard deviation of noise added to raw density. Default: 0.0 [S1, unofficial].
*   `density_bias`: float, shift added to raw densities pre-activation. Default: -1.0 [S1, unofficial].
*   `rgb_activation`: Callable, RGB activation function. Default: `sigmoid` [S1, unofficial].
*   `rgb_padding`: float, padding added to RGB outputs. Default: 0.001 [S1, unofficial].
*   `disable_integration`: bool, if True, use Point-wise Positional Encoding instead of Integrated Positional Encoding. Default: False [S1, unofficial].

### Steps
1.  **Initialization**: Initialize the MLP and Image Signal Processing (ISP) modules. Create an empty result list `ret`.
2.  **Level Loop**: Iterate `i_level` from 0 to `num_levels - 1`.
3.  **Randomness**: Split the random number generator `rng` into `key` and a new `rng`.
4.  **Sampling**:
    *   If `i_level == 0` (Coarse): Perform stratified sampling along rays.
        *   Calculate effective radius: `effective_radius = rays.radii * train_coc`.
        *   Call `sample_along_rays(key, rays.origins, rays.directions, effective_radius, num_samples, rays.near, rays.far, randomized, lindisp, ray_shape)`.
        *   This returns `t_vals` (sample positions along the ray) and `samples` (3D coordinates and radii at those positions).
    *   If `i_level > 0` (Fine): Resample along rays based on previous weights.
        *   Call `resample_along_rays(key, rays.origins, rays.directions, effective_radius, t_vals, weights, randomized, ray_shape, stop_level_grad, resample_padding)`.
        *   This returns updated `t_vals` and `samples`.
5.  **Integration Check**: If `disable_integration` is True, replace `samples` with `(samples[0], jnp.zeros_like(samples[1]))` to disable integrated encoding (effectively reverting to point-wise PE).
6.  **Positional Encoding**:
    *   Compute integrated positional encoding for the 3D samples using `integrated_pos_enc(samples, min_deg_point, max_deg_point)`. This function accounts for the cone/cylinder shape of the ray segment by integrating the standard positional encoding over the interval defined by the sample's radius.
    *   If `use_viewdirs` is True, compute view direction encoding using `pos_enc(rays.viewdirs, min_deg=0, max_deg=deg_view, append_identity=True)`.
7.  **MLP Forward Pass**:
    *   Concatenate point encoding and view direction encoding (if used) to form the input to the MLP.
    *   The MLP outputs `raw_density` and `raw_rgb`.
    *   If `randomized` is True and `density_noise > 0`, add Gaussian noise: `raw_density += density_noise * random.normal(key, raw_density.shape, dtype=raw_density.dtype)`.
8.  **Activations**:
    *   Apply RGB activation: `rgb = rgb_activation(raw_rgb)`.
    *   Apply RGB padding: `rgb = rgb * (1 + 2 * rgb_padding) - rgb_padding`.
    *   Apply density activation: `density = density_activation(raw_density + density_bias)`.
9.  **Volume Rendering**:
    *   Perform volumetric rendering using `volumetric_rendering(rgb, density, t_vals, rays.directions, white_bkgd)`.
    *   This step computes the final color `comp_rgb`, expected distance, accumulated alpha `acc`, and per-sample weights `weights` using alpha compositing.
10. **ISP**: Apply the learned Image Signal Processing pipeline to `comp_rgb` using `isp(comp_rgb, rays.temperature, rays.exposure)` to get `rgb_out`.
11. **Result Storage**: Append `(rgb_out, distance, acc)` to `ret`.
12. **Return**: Return the list `ret`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
import jax.random as random
from typing import Tuple, List, Any

# Assuming helper functions are defined elsewhere in the library:
# mip.sample_along_rays(...)
# mip.resample_along_rays(...)
# mip.integrated_pos_enc(...)
# mip.pos_enc(...)
# mip.volumetric_rendering(...)
# isp(...)
# nn.softplus, nn.sigmoid

class MipNerfModel:
    def __init__(
        self,
        num_samples: int = 128,
        num_levels: int = 2,
        resample_padding: float = 0.01,
        stop_level_grad: bool = True,
        use_viewdirs: bool = True,
        lindisp: bool = False,
        ray_shape: str = 'cone',
        min_deg_point: int = 0,
        max_deg_point: int = 16,
        deg_view: int = 4,
        density_activation: Any = nn.softplus,
        density_noise: float = 0.0,
        density_bias: float = -1.0,
        rgb_activation: Any = nn.sigmoid,
        rgb_padding: float = 0.001,
        disable_integration: bool = False,
    ):
        self.num_samples = num_samples
        self.num_levels = num_levels
        self.resample_padding = resample_padding
        self.stop_level_grad = stop_level_grad
        self.use_viewdirs = use_viewdirs
        self.lindisp = lindisp
        self.ray_shape = ray_shape
        self.min_deg_point = min_deg_point
        self.max_deg_point = max_deg_point
        self.deg_view = deg_view
        self.density_activation = density_activation
        self.density_noise = density_noise
        self.density_bias = density_bias
        self.rgb_activation = rgb_activation
        self.rgb_padding = rgb_padding
        self.disable_integration = disable_integration

    def __call__(
        self,
        rays: Any,
        rng: Any,
        train_coc: float = 1.0,
        white_bkgd: bool = False,
        randomized: bool = True,
    ) -> List[Tuple[Any, Any, Any]]:
        """
        Renders a batch of rays.
        
        Args:
            rays: Object containing origins, directions, viewdirs, radii, near, far, temperature, exposure.
            rng: JAX random key.
            train_coc: Circle of Confusion multiplier for ray radii during training.
            white_bkgd: Whether to composite against a white background.
            randomized: Whether to use randomized sampling.
            
        Returns:
            List of tuples (rgb_out, distance, acc) for each level.
        """
        ret = []
        
        # Pre-calculate effective radii for the first level
        # Note: In subsequent levels, radii are handled within resample_along_rays
        effective_radii = rays.radii * train_coc
        
        for i_level in range(self.num_levels):
            # Split RNG
            key, rng = random.split(rng)
            
            if i_level == 0:
                # Coarse sampling
                t_vals, samples = mip.sample_along_rays(
                    key=key,
                    origins=rays.origins,
                    directions=rays.directions,
                    radii=effective_radii,
                    num_samples=self.num_samples,
                    near=rays.near,
                    far=rays.far,
                    randomized=randomized,
                    lindisp=self.lindisp,
                    ray_shape=self.ray_shape,
                )
                weights = None # Weights not available for coarse level initially
            else:
                # Fine resampling
                # Note: 'weights' comes from the previous level's rendering output
                # In the source, 'weights' is passed from the previous iteration's result
                # We need to access the weights from the previous step's output
                # Assuming 'ret' stores the outputs and we need to extract weights
                # However, the source pseudocode implies 'weights' is available.
                # In a standard implementation, we store weights in the return tuple.
                # Let's assume we have access to 'prev_weights' from the previous loop iteration.
                # Since we are inside the loop, we need to handle the first iteration's lack of weights.
                # The source code shows: `mip.resample_along_rays(..., weights, ...)`
                # This implies weights are passed. For i_level > 0, we use weights from i_level-1.
                
                # Retrieve weights from previous level's result
                # ret[i_level-1] contains (rgb, distance, acc) but we need weights.
                # The source pseudocode snippet:
                # `mip.resample_along_rays(..., t_vals, weights, ...)`
                # It seems `weights` is an output of `volumetric_rendering`.
                # We must store weights in the return value or a separate list.
                # Let's adjust the return structure to include weights for resampling.
                # But the source says: `Append (rgb_out, distance, acc) to ret`.
                # This is a discrepancy. Standard Mip-NeRF returns weights for resampling.
                # We will assume the implementation stores weights internally or returns them.
                # For the sake of the pseudocode matching the source exactly:
                # We will assume `weights` is available from the previous step.
                # If this is the first fine level, we need weights from the coarse level.
                
                # Re-reading source:
                # `mip.resample_along_rays(..., previous t_vals, weights, ...)`
                # The source code snippet for `render_eval_pfn` calls `models.render_image`.
                # The `MipNerfModel` class logic is described in the notes.
                # The notes say: `If i_level > 0, resample ... with inputs ... previous t_vals, weights`.
                # This implies `weights` must be carried over.
                # We will modify the return to include weights for the next level's resampling.
                
                # To strictly follow the source's `ret` append, we might need a separate storage for weights.
                # However, for a complete implementation, we must return weights.
                # Let's assume the `ret` list is not the only storage, or we return a tuple including weights.
                # Given the constraint "Pseudocode ... rewritten in one clean common style",
                # we will include weights in the return value for correctness, as it's required for the next step.
                
                # Accessing previous weights:
                # In the source, `weights` is an output of `volumetric_rendering`.
                # We need to store it.
                pass # Logic handled below

            # ... Sampling logic needs to be integrated with weight storage ...
            
            # Let's rewrite the loop to handle weight storage properly for resampling
            # The source notes are slightly ambiguous on where `weights` comes from for i_level > 0
            # if it's not returned. It must be returned from the previous `volumetric_rendering`.
            
            # Revised Loop Structure for Pseudocode:
            
            prev_t_vals = None
            prev_weights = None
            
            # We need to re-initialize the loop logic to capture weights
            # Since I cannot change the source notes, I will implement the logic as described,
            # assuming `weights` is available from the previous iteration's rendering.
            
            # To make the pseudocode self-contained and correct:
            
            current_t_vals = None
            current_samples = None
            current_weights = None
            
            for i_level in range(self.num_levels):
                key, rng = random.split(rng)
                
                if i_level == 0:
                    current_t_vals, current_samples = mip.sample_along_rays(
                        key=key,
                        origins=rays.origins,
                        directions=rays.directions,
                        radii=effective_radii,
                        num_samples=self.num_samples,
                        near=rays.near,
                        far=rays.far,
                        randomized=randomized,
                        lindisp=self.lindisp,
                        ray_shape=self.ray_shape,
                    )
                else:
                    # Resample using previous t_vals and weights
                    current_t_vals, current_samples = mip.resample_along_rays(
                        key=key,
                        origins=rays.origins,
                        directions=rays.directions,
                        radii=effective_radii, # Note: source uses rays.radii*train_coc
                        t_vals=prev_t_vals,
                        weights=prev_weights,
                        randomized=randomized,
                        ray_shape=self.ray_shape,
                        stop_level_grad=self.stop_level_grad,
                        resample_padding=self.resample_padding,
                    )
                
                # Disable integration if specified
                if self.disable_integration:
                    # samples is a tuple (points, radii)
                    # Replace radii with zeros
                    current_samples = (current_samples[0], jnp.zeros_like(current_samples[1]))
                
                # Integrated Positional Encoding
                point_encoding = mip.integrated_pos_enc(
                    samples=current_samples,
                    min_deg=self.min_deg_point,
                    max_deg=self.max_deg_point,
                )
                
                # View Direction Encoding
                if self.use_viewdirs:
                    view_encoding = mip.pos_enc(
                        dirs=rays.viewdirs,
                        min_deg=0,
                        max_deg=self.deg_view,
                        append_identity=True,
                    )
                    # Concatenate point and view encodings
                    # Assuming MLP expects concatenated input
                    mlp_input = jnp.concatenate([point_encoding, view_encoding], axis=-1)
                else:
                    mlp_input = point_encoding
                
                # MLP Forward Pass
                raw_density, raw_rgb = self.mlp(mlp_input) # Assuming self.mlp is defined
                
                # Density Noise
                if randomized and self.density_noise > 0:
                    noise = self.density_noise * random.normal(key, raw_density.shape, dtype=raw_density.dtype)
                    raw_density = raw_density + noise
                
                # Activations
                rgb = self.rgb_activation(raw_rgb)
                rgb = rgb * (1 + 2 * self.rgb_padding) - self.rgb_padding
                
                density = self.density_activation(raw_density + self.density_bias)
                
                # Volumetric Rendering
                comp_rgb, distance, acc, current_weights = mip.volumetric_rendering(
                    rgb=rgb,
                    density=density,
                    t_vals=current_t_vals,
                    directions=rays.directions,
                    white_bkgd=white_bkgd,
                )
                
                # ISP
                rgb_out = isp(comp_rgb, rays.temperature, rays.exposure)
                
                # Store results
                # Note: We need to store weights for the next level's resampling
                # The source notes say: Append (rgb_out, distance, acc) to ret.
                # This implies weights are not returned in `ret` but are used internally.
                # However, for the next level, we need `prev_weights`.
                # We will store them in a separate variable or extend the return.
                # To be faithful to the source's `ret` structure, we might assume
                # the model class holds `prev_weights` as state or returns them separately.
                # Given the instruction "complete enough to implement", we must return weights.
                
                ret.append((rgb_out, distance, acc, current_weights))
                
                # Prepare for next level
                prev_t_vals = current_t_vals
                prev_weights = current_weights

        # Return only the requested outputs if strictly following source, 
        # but for implementation, we return weights too.
        # The source notes say: `Append (rgb_out, distance, acc) to ret`.
        # We will return the list as described, but note that weights are needed for training.
        # In a real implementation, the training loop would extract weights.
        # Here, we return the tuple including weights for clarity.
        
        # If strict adherence to source notes is required for the return value:
        # return [(r, d, a) for r, d, a, w in ret]
        # But this breaks the resampling logic for the next level if not stored.
        # We will return the full tuple.
        
        return ret
```

### Edge Cases and Pitfalls
*   **Precision**: The integrated positional encoding involves integration over the cone/cylinder. Numerical stability depends on the radius size. Very small radii may lead to precision issues in the integration.
*   **Boundaries**: The `near` and `far` planes must be correctly set for each ray. Incorrect bounds can lead to sampling outside the scene or missing geometry.
*   **Degenerate Input**: If `radii` are zero (e.g., at the camera origin), the cone shape degenerates to a line. The `integrated_pos_enc` function should handle this gracefully, typically by falling back to point-wise encoding or ensuring the integration interval is non-zero.
*   **Gradient Flow**: If `stop_level_grad` is True, gradients do not flow from the fine level to the coarse level. This is a design choice to stabilize training but must be respected in the implementation.

## Subtopics

### Depth-Guided Sampling
Uses depth estimates to guide sample placement, focusing on areas of interest. [Depth-Guided Sampling](7490-depth-guided-sampling.md)

### Depth-Prior Guided Stratified Sampling
Combines stratified sampling with depth priors for more efficient sampling. [Depth-Prior Guided Stratified Sampling](11292-depth-prior-guided-stratified-sampling.md)

### Progressive Blur Estimation
Estimates blur parameters to improve rendering quality in out-of-focus regions. [Progressive Blur Estimation](14151-progressive-blur-estimation.md)

## Operating characteristics
*   **Time Complexity**: $O(L \cdot N)$, where $L$ is the number of levels and $N$ is the number of samples per level. The integrated positional encoding adds a constant factor overhead per sample.
*   **Memory Complexity**: $O(L \cdot N)$ for storing samples, encodings, and intermediate rendering results.
*   **Measured Performance**: The model is designed for unbounded, outdoor scenes (Mip-NeRF 360 dataset). It provides higher quality images with fewer artifacts compared to standard NeRF, especially at high frequencies.
*   **Scaling**: Scales linearly with the number of samples and levels.
*   **Strengths**: Reduces aliasing, handles high-frequency details better, suitable for unbounded scenes.
*   **Limitations**: More complex implementation, slightly higher computational cost per sample due to integration.
*   **When to Use**: When training on unbounded scenes or scenes with high-frequency details where standard NeRF exhibits aliasing.
*   **When Not to Use**: For simple, bounded scenes with low-frequency content, standard NeRF may be sufficient and faster.

## Background
The MipNerfModel was introduced in the Mip-NeRF paper by Barron et al. (2021) and extended in Mip-NeRF 360 (Barron et al., 2022). It addresses the issue of aliasing in NeRF by modeling rays as cones/cylinders and integrating positional encodings over these volumes. The implementation described here is based on the JAX-based codebase from the CaSE-NeRF repository [S1, unofficial].

## Sources
[S1] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
