# Spacetime

Also called: Spacetime feature interpolation, temporal feature field extension

## Overview
Spacetime is a method that adds a temporal dimension to the feature field of 3D Gaussians, enabling real-time interpolation of both color and geometry between discrete keyframes [S1, unofficial]. It addresses the problem of rendering smooth, continuous dynamic scenes from sparse or discrete temporal samples, distinguishing itself from methods that rely solely on explicit particle tracking or optical flow by embedding time directly into the feature representation [S1, unofficial]. Unlike [Spacetime Gaussians (STG)](5997-spacetime-gaussians-stg.md), which uses explicit polynomial trajectories and opacity envelopes, Spacetime relies on learned feature fields conditioned on time [S1, unofficial].

## How it works
The method extends the static Gaussian feature representation by conditioning it on a time coordinate $t$. This allows the radiance field to reconstruct geometry and appearance for any time $t$ within the training range, not just the keyframes used during optimization.

### Inputs and Outputs
- **Inputs**: 
  - A set of 3D Gaussians $\mathcal{G}$, each with spatial parameters (position $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$).
  - A time feature $\gamma(t)$ derived from the current time $t$.
  - Camera parameters (view matrix, projection matrix, intrinsics).
- **Outputs**: 
  - Rendered image (color and depth) for the specified time $t$ and camera pose.

### Data Structures
The core data structure is the Gaussian packet, which includes:
- `get_features`: The base feature vector (e.g., SH coefficients or MLP input features) associated with each Gaussian.
- `unique_kfIDs`: An integer array indicating which keyframe(s) each Gaussian was primarily associated with or trained on.
- `d_xyz`: Deformation deltas, if applicable for the specific variant (e.g., in GToF).

### Steps
1. **Time Feature Generation**: Compute the time feature $\gamma(t)$ for the desired rendering time $t$. This is typically done using positional encoding or a learned embedding [S3].
2. **Feature Conditioning**: Input the spatial position $\mathbf{x}$, the time feature $\gamma(t)$, and potentially other attributes into the feature network (e.g., an MLP or grid-based field) to retrieve the color and deformation parameters for that specific time [S1, unofficial][S3].
3. **Deformation (Optional)**: If the scene is dynamic, compute the deformed position $\mathbf{x}' = \mathbf{x} + \Delta \mathbf{x}(t)$ using the learned deformation field [S3].
4. **Rendering**: Rasterize the Gaussians using the standard 3D Gaussian Splatting rasterizer, using the time-conditioned features for color and the deformed positions for geometry [S1, unofficial].

### Formulas
The attenuated time feature $\gamma_{time}(t)$ used in some implementations to prevent overfitting to time in deforming regions is given by [S3]:

$$
\gamma_{time}(t) = \gamma(t) e^{-\frac{\lambda}{2} \frac{L(\gamma(t))}{\Delta x}}
$$

Where:
- $\gamma(t)$ is the original time feature.
- $\Delta x$ is the norm of the deformation distance.
- $\lambda$ is the attenuation strength hyperparameter.
- $L(\gamma(t))$ is a term related to the feature magnitude (exact formulation garbled in source, but conceptually scales with feature energy) [S3].

### Parameters
- **Attenuation strength $\lambda$**: Controls the degree of time feature attenuation. Set to 60 in experiments [S3].
- **Blend alpha $\alpha$**: Used in conditional feature blending for visualization. Set to 0.1 [S4, unofficial].

### Edge Cases and Pitfalls
- **Noisy Reconstructions**: In monocular settings, relying solely on the time feature can cause the radiance field to overfit to time rather than learning correct geometry, leading to noisy novel view synthesis [S3]. The attenuation method mitigates this [S3].
- **Discrete Keyframe Artifacts**: Without interpolation, rendering between keyframes may result in popping or discontinuities. Temporal interpolation of features or deformations is required [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_spacetime(gaussians, camera, time_t, lambda_atten=60):
    """
    Renders a scene at time t using spacetime feature conditioning.
    """
    # 1. Generate time feature
    # gamma_t is the encoded time feature, e.g., via positional encoding
    gamma_t = encode_time(time_t)
    
    # 2. Query deformation and features
    # This step depends on the specific implementation (MLP/Grid)
    # For GToF variant, we might interpolate deformation deltas
    if use_gtof_interpolation:
        d_xyz = interpolate_deformation(gaussians, time_t)
    else:
        # Query deformation from network conditioned on time
        d_xyz = deformation_network(gaussians.positions, gamma_t)
        
    # 3. Apply attenuation if deformation is large (Ced-NeRF style)
    # Calculate deformation distance norm
    delta_x_norm = torch.norm(d_xyz, dim=-1)
    
    # Calculate attenuation factor
    # Note: L(gamma(t)) is approximated or retrieved from feature magnitude
    # Assuming L is a scalar or per-gaussian feature norm
    feature_norm = get_feature_norm(gaussians) 
    attenuation_factor = torch.exp(-0.5 * lambda_atten * feature_norm / (delta_x_norm + 1e-6))
    
    # Attenuated time feature
    gamma_time = gamma_t * attenuation_factor
    
    # 4. Get final features and positions
    # In a full implementation, the feature network takes (x, gamma_time)
    # and returns color and updated covariance/opacity
    final_positions = gaussians.positions + d_xyz
    final_features = feature_network(final_positions, gamma_time)
    
    # 5. Rasterize
    # Standard 3DGS rasterization with final_positions and final_features
    image = rasterize(final_positions, final_features, camera)
    
    return image
```

## Subtopics

### Temporal Gaussian Interpolation
A variant used in the GToF (Gaussian Flow) context for the ToRF scene type. It ensures smooth motion between discrete keyframes by linearly interpolating deformation deltas ($d\_xyz$) across time steps [S2, unofficial]. It calculates current and next interpolation frame IDs based on `view.frame_id // 4` and blends the deltas if the frame is not a multiple of 4 [S2, unofficial]. This is chosen when discrete keyframe-based training is used but smooth temporal rendering is required [S2, unofficial].

### Attenuating Time Feature
A technique introduced in Ced-NeRF to reduce the frequency of the time feature for deforming parts of the scene. It prevents the radiance field from relying solely on time features, which causes noisy novel view synthesis [S3]. It is chosen when using monocular cameras or when deformation is significant, as it forces the network to learn geometry from coordinates rather than just time [S3].

### Conditional Feature Blending for Time Shader
A visualization/debugging method that blends Gaussian features with a colormap based on keyframe IDs. It is controlled by a UI checkbox and uses a blend alpha of 0.1 [S4, unofficial]. It is not a rendering method for production but a tool for inspecting which keyframes influence which Gaussians [S4, unofficial].

## Operating characteristics
- **Time Complexity**: Rendering time is similar to standard 3D Gaussian Splatting, as the time conditioning is typically a lightweight lookup or small MLP evaluation per Gaussian [S1, unofficial]. Interpolation adds minimal overhead [S2, unofficial].
- **Memory Complexity**: Requires storing time-conditioned features or a deformation network. If using explicit interpolation (GToF), no extra memory beyond deformation deltas is needed [S2, unofficial]. If using a neural field (Ced-NeRF), the network weights add memory overhead [S3].
- **Measured Performance**: Real-time frame rates are achievable, as demonstrated by the ability to interpolate between keyframes in real-time [S1, unofficial]. Ced-NeRF reports efficient performance on a single RTX 3090 GPU [S3].
- **Strengths**: Enables smooth, continuous rendering of dynamic scenes from sparse temporal data; real-time performance [S1, unofficial][S3].
- **Limitations**: Can suffer from noisy reconstructions in novel views if not properly regularized (e.g., via attenuation) [S3]. Relies on the quality of the time feature encoding [S3].

## Background
Spacetime feature extension was introduced as part of the broader evolution of 4D Gaussian Splatting methods. It is cited in the context of "Flow, Tracking, and Control" methods, distinguishing it from Gaussian-Flow (which uses velocity fields) and Gaussian Splatting Lucas-Kanade (which uses optical flow) [S1, unofficial]. The concept of adding a temporal dimension to feature fields has been explored in dynamic NeRFs (e.g., Ced-NeRF) and adapted for Gaussian Splatting to enable real-time interpolation [S1, unofficial][S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
[S3] Ced-NeRF: A Compact and Efficient Method for Dynamic Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i4.28138, fetched 2026-10-26
[S4] muskie82/MonoGS, https://github.com/muskie82/MonoGS, fetched 2026-10-27 (unofficial)
