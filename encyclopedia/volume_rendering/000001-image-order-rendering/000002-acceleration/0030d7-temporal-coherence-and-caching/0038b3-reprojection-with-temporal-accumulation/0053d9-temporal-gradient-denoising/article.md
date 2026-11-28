# Temporal Gradient Denoising

Also called: Temporal denoising of gradient image, Temporal denoising on gradients

## Overview
Temporal Gradient Denoising reduces noise in volume rendering by applying temporal reprojection and blending to gradient images rather than the final radiance image [S1, S2]. It solves the problem of high noise levels in Monte Carlo volume rendering, particularly in high-order scattering-dominant media, while avoiding the visible bias and artifacts that occur when denoising the original image directly [S1, S2]. It sets itself apart from standard temporal accumulation by operating on the derivative of the radiance with respect to a source parameter (albedo), leveraging the fact that denoising artifacts in the gradient domain are not directly visible in the final color [S1, S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of rendered gradient images $G_k$ for $k \in [0, N-1]$, where $k=0$ is the target frame; motion vector $d$ and velocity $v_d$ in screen space; smoothing factor $w_a$; number of frames $N$; time offset $t$ [S1, S2].
- **Outputs:** A denoised gradient image $G_{\text{denoised}}$ for the target frame [S1, S2].

**Data Structures:**
- **Gradient Image:** A 2D array of floating-point values representing the gradient of radiance with respect to albedo at each pixel [S1, S2]. Layout is standard screen-space raster format [G].
- **Motion Vector:** A 2D vector $(d_x, d_y)$ per pixel or globally, derived from camera movement [S2].

**Steps:**
1. **Compute Motion Parameters:** Determine the motion vector $d$ and velocity $v_d$ in screen space based on the camera's moving direction and velocity [S2].
2. **Reproject Pixels:** For each pixel $i$ in the target frame, calculate the corresponding pixel coordinate in neighboring frames using the inverse reprojection function $f_{\text{proj}}^{-1}$ [S1, S2].
3. **Blend Gradients:** Compute the weighted average of the reprojected gradient values from neighboring frames to produce the denoised gradient [S1, S2].

**Formulas:**
The reprojection function maps a pixel $i$ in the target frame to a coordinate in a neighboring frame at distance $k$:
$$ f_{\text{proj}}(i) = i + d \cdot v_d \cdot t $$
where $t$ is the time offset [S1, S2].

The denoised gradient $G_{\text{denoised}}(i)$ is calculated as:
$$ G_{\text{denoised}}(i) = \frac{\sum_{k=0}^{N-1} w_a^k (1-w_a)^{-1} G_k(f_{\text{proj}}^{-1}(i))}{1 - (1-w_a)^N} $$
where:
- $k$ is the frame distance (number of frames between the source image and the target image, with $k=0$ being the target) [S1, S2].
- $G_k$ is the gradient image at distance $k$ [S1, S2].
- $w_a$ is the smoothing factor [S1, S2].
- $N$ is the number of frames used for denoising [S1, S2].
- $f_{\text{proj}}^{-1}$ is the reciprocal reprojection function, mapping the target pixel $i$ back to the source frame coordinate [S2].

**Parameters:**
- **Smoothing factor ($w_a$):** Set to 0.1 in all test scenes [S1, S2].
- **Number of frames ($N$):** Set to 11 in the implementation [S1, S2].

**Pitfalls:**
- **Direct Image Denoising:** Denoising the original radiance image directly causes visible bias and artifacts. Denoising the gradient prevents this because such artifacts are not directly visible in the gradient domain [S1, S2].
- **Boundary Handling:** Not given in the sources. [G] Implementation requires clamping or zero-padding when $f_{\text{proj}}^{-1}(i)$ falls outside image bounds.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_gradient_denoise(target_gradient, neighbor_gradients, motion_vector, velocity, smoothing_factor, num_frames):
    # target_gradient: 2D list of floats, G_0
    # neighbor_gradients: list of 2D lists, G_1 to G_{N-1}
    # motion_vector: tuple (dx, dy) for d
    # velocity: tuple (vx, vy) for v_d
    # smoothing_factor: float w_a
    # num_frames: int N
    # Returns: 2D list of floats G_denoised

    h, w = len(target_gradient), len(target_gradient[0])
    weighted_sum = [[0.0] * w for _ in range(h)]
    normalization_factor = 1.0 - (1.0 - smoothing_factor) ** num_frames

    # Precompute weight scaling factor (1-w_a)^{-1}
    inv_one_minus_wa = 1.0 / (1.0 - smoothing_factor)

    # Process all frames k from 0 to N-1
    for k in range(num_frames):
        if k == 0:
            grad_frame = target_gradient
        else:
            if k - 1 < len(neighbor_gradients):
                grad_frame = neighbor_gradients[k - 1]
            else:
                break

        # Calculate reprojection offset for frame k
        # t is implicitly 1 per frame step in the sequence
        offset_x = motion_vector[0] * velocity[0] * k
        offset_y = motion_vector[1] * velocity[1] * k

        # Weight for this frame: w_a^k * (1-w_a)^{-1}
        weight = (smoothing_factor ** k) * inv_one_minus_wa

        # Accumulate weighted gradient
        for y in range(h):
            for x in range(w):
                # Inverse reprojection: source coordinate = target coordinate - offset
                src_x = x - offset_x
                src_y = y - offset_y

                # Sample gradient with bilinear interpolation and boundary clamping
                val = sample_gradient(grad_frame, src_x, src_y)
                weighted_sum[y][x] += weight * val

    # Normalize
    denoised_gradient = [[weighted_sum[y][x] / normalization_factor for x in range(w)] for y in range(h)]
    return denoised_gradient

def sample_gradient(image, x, y):
    h, w = len(image), len(image[0])
    # Clamp to valid range
    x0 = max(0, min(int(x), w - 1))
    y0 = max(0, min(int(y), h - 1))
    x1 = min(x0 + 1, w - 1)
    y1 = min(y0 + 1, h - 1)

    dx = x - x0
    dy = y - y0

    val00 = image[y0][x0]
    val01 = image[y0][x1]
    val10 = image[y1][x0]
    val11 = image[y1][x1]

    return (1 - dx) * (1 - dy) * val00 + dx * (1 - dy) * val01 + (1 - dx) * dy * val10 + dx * dy * val11
```

## Subtopics

### [Spatiotemporal Volume Denoising with Background Fusion](15105-spatiotemporal-volume-denoising-with-background-fusion.md)
This method denoises Monte Carlo volume rendering images by combining temporal reprojection with a learned kernel prediction network and fusing the result with the background using an alpha map. It addresses the specific challenge of preserving semi-transparent structures and sharp edges in volumetric data while maintaining real-time performance. Unlike direct prediction methods (e.g., FDAE) that rely solely on pixel-wise regression, this approach generates weighted combinations of neighboring samples, offering better structural preservation at the cost of struggling with extreme high-intensity outliers.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and the number of frames $N$ [S1, S2].
- **Memory Complexity:** Requires storing $N$ gradient images and their corresponding motion vectors [S1, S2].
- **Measured Performance:** In test scenes (Bunny, Candle, Lucy) with resolution 256x256, the method produced less error than denoised path tracing and path tracing without denoising, both visually and quantitatively [S1]. The method was implemented with $N=11$ and $w_a=0.1$ [S1, S2].
- **Strengths:** Effectively reduces noise in high-order scattering-dominant media; avoids visible bias/artifacts associated with direct image denoising; improves quality over standard path tracing with equal time [S1, S2].
- **Limitations:** Requires a sequence of frames (temporal coherence); performance gain over path tracing is specific to gradient denoising, as temporal denoising on path tracing itself does not perform better than path tracing with equal time [S1].
- **When to use:** When rendering sequences of volume data with high noise levels, particularly in participating media with high albedo, and when temporal coherence is available [S1, S2].
- **When not to use:** For single-image rendering; when extreme high-intensity outliers are present and structural preservation is critical without learned kernels [S1].

## Background
The method was proposed in the context of efficient participating media rendering using differentiable volumetric path tracing [S1, S2]. It was introduced to address the high noise levels in gradient images computed with respect to albedo, which are used to predict radiance for different albedo values without re-rendering [S2]. The approach leverages the common practice of rendering image sequences with moving cameras [S1, S2]. The work was published in 2023 [S1, S2].

## Sources
[S1] Efficient Participating Media Rendering with Differentiable ..., https://iccvm.org/2023/papers/s8-3-284-CVMJ.pdf, fetched 2026-10-29
[S2] Efficient participating media rendering with differentiable ..., https://www.sciopen.com/article/10.1007/s41095-023-0372-2?issn=2096-0433, fetched 2026-10-29
