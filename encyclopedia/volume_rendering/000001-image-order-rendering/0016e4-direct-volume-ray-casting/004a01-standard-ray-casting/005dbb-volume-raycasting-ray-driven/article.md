# Volume Raycasting (Ray Driven)

Also called: Basic Ray Casting, Ray Driven Ray Casting

## Overview
Volume Raycasting (Ray Driven) is a direct volume rendering method that samples along rays through the entire dataset to render three-dimensional scalar fields [S1, unofficial]. It differs from splatting by accumulating samples interpolated between voxels surrounding the ray [S2, unofficial]. While it builds on the standard front-to-back compositing of [Standard Ray Casting](12643-standard-ray-casting.md), this specific implementation variant is characterized by its simplicity, sometimes omitting interpolation in favor of direct voxel access, and supporting specific modes like limited depth raycasting [S3].

## How it works
**Inputs and Outputs**:
- Input: A 3D scalar volume $V$, camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and rendering mode (Basic or Limited Depth) [S3, G].
- Output: A 2D image with accumulated color and opacity per pixel [G].

**Data Structures**:
- Volume: A 3D array of scalar values. The non-interpolated implementation accesses voxel values directly without interpolation [S3].
- Ray: Defined by entry $t_{near}$ and exit $t_{far}$ bounds within the volume bounding box [G].

**Steps**:
1. **Prefiltering**: Apply a threshold and cutoff limit to the volume data. Voxels with values exceeding the cutoff limit are replaced with zeros (black) to eliminate reconstruction noise [S3].
2. **Ray Initialization**: For each pixel, define the ray $r(t) = \mathbf{o} + t\mathbf{d}$ and compute intersection with the volume bounding box to find $t_{near}$ and $t_{far}$ [G].
3. **Sampling and Compositing**: Iterate along the ray from $t_{near}$ to $t_{far}$ [G].
   - If **Basic Mode** is selected, cast all voxels along the ray trace [S3].
   - If **Limited Depth Mode** is selected, start raycasting with a voxel value threshold condition. The ray continues only up to the selected depth where the condition is met, resulting in a partially transparent model [S3].
   - Retrieve the scalar value $V(\mathbf{p}_i)$ at the current sample position [G].
   - Map $V(\mathbf{p}_i)$ to color $\mathbf{c}_i$ and opacity $\alpha_i$ using a transfer function [G].
   - Composite using the front-to-back equation:
     $$
     \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{c}_i \cdot \alpha_i
     $$
     $$
     \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i
     $$
4. **Post-filtering**: Apply gamma correction and saturation level to the final image. Optionally disable automatic white level control for manual white level, or invert the image [S3].

**Parameters**:
- Cutoff limit: Threshold for prefiltering noise [S3].
- Depth threshold: Condition for stopping ray in Limited Depth mode [S3].
- Gamma: Post-processing gamma correction value [S3].

**Edge Cases and Pitfalls**:
- **Basic Mode Transparency**: In Basic mode, casting all voxels along the ray trace results in a fully transparent model [S3].
- **Bounding Box**: Reducing the bounding area may increase performance but risks clipping valid data if not aligned with the volume extent [S3].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ray_casting(volume, camera_origin, camera_direction, volume_bounds, mode, cutoff_threshold, depth_threshold, gamma):
    # Initialize output image with zeros
    image = zeros_like(camera_image_shape)
    # Pre-filter volume: replace values exceeding cutoff with zero
    volume_filtered = where(volume > cutoff_threshold, 0, volume)
    for each pixel (x, y):
        ray_o = camera_origin(x, y)
        ray_d = camera_direction(x, y)
        t_near, t_far = intersect_box(ray_o, ray_d, volume_bounds)
        C_acc = (0.0, 0.0, 0.0)
        alpha_acc = 0.0
        t = t_near
        # Step size not given in sources; typically derived from ray length and sample count [G]
        dt = (t_far - t_near) / sample_count
        while t < t_far:
            p = ray_o + t * ray_d
            v = sample_volume(volume_filtered, p)
            if mode == "limited_depth" and v < depth_threshold:
                break
            c, alpha = transfer_function(v)
            C_acc = C_acc + (1.0 - alpha_acc) * c * alpha
            alpha_acc = alpha_acc + (1.0 - alpha_acc) * alpha
            t += dt
        image[x, y] = apply_post_filter(C_acc, gamma)
    return image
```

## Operating characteristics
- **Time Complexity**: $O(n^3)$ for an $n \times n \times n$ volume, as it requires sampling along rays through the entire dataset [S1, unofficial].
- **Memory**: Large volumes demand substantial storage; a $512^3$ dataset requires ~0.5 GB for 32-bit precision [S1, unofficial].
- **Performance**: Rendering a $256^3$ volume at 1080p can achieve interactive frame rates (20–60 fps) on modern GPUs with optimized pipelines [S1, unofficial].
- **Strengths**: Simple implementation; effective for viewing 3D volumes without external tools [S3].
- **Limitations**: High computational cost for large volumes; undersampling leads to aliasing and blur [S1, unofficial].

## Background
- **Origin**: The ray casting model was introduced by Levoy in 1988 [S2, unofficial].
- **Development**: It is a fundamental technique in volume rendering, often contrasted with splatting [Westover89] [S2, unofficial].
- **Usage**: Used in various visualization tools, including custom implementations to avoid dependencies on external software like 3D Slicer [S3].

## Subtopics

### [Volume Ray Casting](2350-volume-ray-casting.md)
A broader category that may include optimizations and variations.

### [VCP Ray-Casting Algorithm](10664-vcp-ray-casting-algorithm.md)
A specific optimized ray-casting algorithm.

### [Front-to-Back Ray Casting with Iso-value Thresholding](19677-front-to-back-ray-casting-with-iso-value-thresholding.md)
A variant using iso-value thresholding for early ray termination.

## Sources
[S1] volume rendering, https://grokipedia.com/page/Volume_rendering, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering concepts for 3D GEEKs., https://www.jch.com/jch/volurend.html, fetched 2026-10-27 (unofficial)
[S3] Cone-beam backprojection tool, https://elektronika.kvalitne.cz/SW/graphics/cone_beam_backprojection/cone_beam_backprojection_eng.html, fetched 2026-10-29
