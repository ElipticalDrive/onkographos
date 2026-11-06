# NeRF Rendering (Alpha Compositing)
Also called: Volume Rendering, Alpha Blending, Ray Marching Integration

## Overview
Generates RGB images and depth maps from neural network predictions by performing volume rendering along camera rays using alpha compositing [S1]. It integrates the predicted volume density and color of sampled points into a final pixel value, accounting for occlusion via transmittance [S1]. Unlike sibling methods that focus on evaluation metrics or network query structures, this method specifically handles the mathematical integration of radiance and opacity into observable quantities [S1][S2, unofficial].

## How it works
**Inputs:**
- `model`: Trained MLP predicting RGB and density [S1].
- `rays_flat`: Flattened ray origins and directions [S1].
- `t_vals`: 1D array of sample distances along rays, shape `(NUM_SAMPLES,)` [S1].
- `rand`: Boolean flag for sampling strategy (True for training with jitter, False for deterministic evaluation) [S1].
- `train`: Boolean flag for model execution mode (True for training, False for inference) [S1].

**Outputs:**
- `rgb`: Rendered RGB image, shape `(BATCH_SIZE, H, W, 3)` [S1].
- `depth_map`: Rendered depth map, shape `(BATCH_SIZE, H, W)` [S1].

**Data Structures:**
- Predictions tensor: Shape `(BATCH_SIZE, H, W, NUM_SAMPLES, 4)`. The last dimension contains `[R, G, B, sigma_a]` [S1].
- Delta tensor: Interval distances between adjacent samples [S1].
- Transmittance tensor: Cumulative product of $(1 - \alpha)$ [S1].
- Weights tensor: Product of alpha and transmittance [S1].

**Steps:**
1. **Model Prediction:** Pass `rays_flat` through the `model`. If `train` is True, use forward pass; otherwise, use prediction mode [S1]. Reshape output to `(BATCH_SIZE, H, W, NUM_SAMPLES, 4)` [S1].
2. **Slicing and Activation:** Slice the last dimension into RGB (first 3 channels) and volume density $\sigma_a$ (last channel) [S1]. Apply sigmoid activation to RGB to bound values to $[0, 1]$ [S1]. Apply ReLU activation to $\sigma_a$ to ensure non-negative density [S1].
3. **Interval Calculation:** Compute distances $\delta$ between adjacent sample points: $\delta = t_{\text{vals}}[i+1] - t_{\text{vals}}[i]$ [S1].
4. **Boundary Handling:** Append a large value ($10^{10}$) to $\delta$ to represent the infinite tail of the ray [S1].
   - If `rand` is True, append shape `(BATCH_SIZE, H, W, 1)` [S1].
   - If `rand` is False, append shape `(BATCH_SIZE, 1)` [S1].
5. **Alpha Calculation:** Compute opacity $\alpha$ for each sample using the formula:
   $$ \alpha = 1 - e^{-\sigma_a \cdot \delta} $$
   - If `rand` is True, multiply $\sigma_a$ and $\delta$ directly [S1].
   - If `rand` is False, broadcast $\delta$ to match spatial dimensions before multiplication [S1].
6. **Transmittance Calculation:** Compute transmittance $T$ as the cumulative product of $(1 - \alpha)$ [S1].
   - Add epsilon ($\epsilon = 10^{-10}$) to $(1 - \alpha)$ to prevent numerical instability [S1].
   - Use exclusive cumulative product to start with $T=1.0$ at the beginning of the ray [S1].
7. **Weight Calculation:** Compute weights $w$ for each sample:
   $$ w = \alpha \cdot T $$
   [S1].
8. **RGB Integration:** Compute final RGB by summing weighted colors:
   $$ \text{RGB} = \sum_{i} w_i \cdot c_i $$
   [S1].
9. **Depth Integration:** Compute depth map by summing weighted sample distances:
   $$ \text{Depth} = \sum_{i} w_i \cdot t_i $$
   [S1].
   - If `rand` is True, use `t_vals` directly [S1].
   - If `rand` is False, broadcast `t_vals` to match spatial dimensions $(BATCH\_SIZE, H, W)$ before multiplication [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import tensorflow as tf
import numpy as np

def render_rgb_depth(model, rays_flat, t_vals, rand=True, train=True):
    """Generates the RGB image and depth map from model prediction.

    Args:
        model: The MLP model that is trained to predict the rgb and
            volume density of the volumetric scene.
        rays_flat: The flattened rays that serve as the input to
            the NeRF model.
        t_vals: The sample points for the rays.
        rand: Choice to randomise the sampling strategy.
        train: Whether the model is in the training or testing phase.

    Returns:
        Tuple of rgb image and depth map.
    """
    # Ensure t_vals is a tensor for consistent broadcasting
    t_vals = tf.constant(t_vals, dtype=tf.float32)
    
    # Get the predictions from the nerf model and reshape it.
    if train:
        predictions = model(rays_flat)
    else:
        predictions = model.predict(rays_flat)
        
    # Extract dimensions from the reshaped predictions
    BATCH_SIZE, H, W, NUM_SAMPLES, _ = predictions.shape
    predictions = tf.reshape(predictions, shape=(BATCH_SIZE, H, W, NUM_SAMPLES, 4))

    # Slice the predictions into rgb and sigma.
    rgb = tf.sigmoid(predictions[..., :-1])
    sigma_a = tf.nn.relu(predictions[..., -1])

    # Get the distance of adjacent intervals.
    delta = t_vals[..., 1:] - t_vals[..., :-1]
    
    # Boundary handling: append a large value to handle the end of the ray
    if rand:
        delta = tf.concat([delta, tf.broadcast_to(tf.constant(1e10), shape=(BATCH_SIZE, H, W, 1))], axis=-1)
        alpha = 1.0 - tf.exp(-sigma_a * delta)
    else:
        delta = tf.concat([delta, tf.broadcast_to(tf.constant(1e10), shape=(BATCH_SIZE, 1))], axis=-1)
        alpha = 1.0 - tf.exp(-sigma_a * delta[:, None, None, :])

    # Get transmittance.
    exp_term = 1.0 - alpha
    epsilon = 1e-10
    transmittance = tf.math.cumprod(exp_term + epsilon, axis=-1, exclusive=True)
    weights = alpha * transmittance
    
    # RGB Integration
    rgb = tf.reduce_sum(weights[..., None] * rgb, axis=-2)

    # Depth Integration
    if rand:
        depth_map = tf.reduce_sum(weights * t_vals, axis=-1)
    else:
        depth_map = tf.reduce_sum(weights * t_vals[:, None, None], axis=-1)
        
    return (rgb, depth_map)
```

## Subtopics

### NeRF Rendering Pipeline
Iterates over a list of camera poses to generate a sequence of rendered images and depth maps [S2, unofficial]. It supports downsampling for faster preview rendering and optional PSNR calculation against ground truth images [S2, unofficial]. It handles saving rendered frames as 8-bit PNGs [S2, unofficial].
- **Parameters:**
  - `render_factor`: Downsampling factor for speed. Default 0 (no downsampling). Typical values 4 or 8 for preview [S2, unofficial].
  - `chunk`: Number of rays processed in parallel during `render`. Default 32768. Decrease if running out of memory [S2, unofficial].
- **Steps:**
  1. Iterate over `render_poses` array [S2, unofficial].
  2. For each pose `c2w`, call `render(H, W, focal, chunk=chunk, c2w=c2w[:3, :4], **render_kwargs)` to get `rgb`, `disp`, `acc` [S2, unofficial].
  3. If `render_factor != 0`, downsample `H`, `W`, and `focal` by dividing by `render_factor` before rendering [S2, unofficial].
  4. Append `rgb.numpy()` and `disp.numpy()` to `rgbs` and `disps` lists [S2, unofficial].
  5. If `gt_imgs` is not None and `render_factor == 0`, compute PSNR: `p = -10. * np.log10(np.mean(np.square(rgb - gt_imgs[i])))` [S2, unofficial].
  6. If `savedir` is not None, convert `rgb` to 8-bit with `to8b()`, save as `'{:03d}.png'` using `imageio.imwrite` [S2, unofficial].
  7. After loop, stack `rgbs` and `disps` along axis 0 and return them [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Differentiable, allowing gradient flow back to the network parameters for training [S1]. Simple and efficient integration of volumetric data [S1].
- **Limitations:** Requires careful handling of numerical stability (epsilon addition) to prevent NaNs [S1].

## Background
Originated from the Neural Radiance Fields (NeRF) paper by Mildenhall et al. (2020) [S1][S2, unofficial]. The method is a core component of the NeRF training loop, enabling the optimization of the radiance field by comparing rendered images with ground truth [S1].

## Sources
[S1] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
