# INPC Inference & Rendering Pipeline

Also called: INPC Pipeline

## Overview
The INPC (Implicit Neural Point Cloud) Inference & Rendering Pipeline is a splatting-based method that renders high-quality radiance fields by blending multisampled splats into feature images, applying global sorting, and refining the output via a U-Net and tone mapping module [S1]. It distinguishes itself from explicit point-based siblings like 3D Gaussian Splatting [3DGS](3DGS.md) or TRIPS [TRIPS](TRIPS.md) by using an implicit neural representation for point attributes, which allows for superior image quality at the cost of higher computational overhead during the sorting and post-processing stages [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained implicit neural point cloud model (defined by an octree structure and neural networks for attributes), camera parameters (view matrix, projection matrix, resolution), and a configuration setting (e.g., 8M or 33M samples).
*   **Output:** A final rendered image (color and/or feature map).

**Data Structures:**
The method relies on an octree-based data structure to manage the point cloud. The specific field layouts of the octree nodes and the neural network weights are not detailed in the sources [S1]. The pipeline processes points defined by position $\mathbf{p}$ and associated implicit attributes.

**Steps:**
1.  **Sampling:** Generate a set of points for the current viewpoint. The method uses a view-independent sampling algorithm. For the default configuration, this involves 33 million samples per training iteration; for the fast configuration, 8 million samples [S1]. During inference, 4× multisampling is applied, meaning each point is sampled 4 times to reduce aliasing [S1].
2.  **Splatting:** Transform the sampled points into screen space and rasterize them as splats. The splats are blended into feature images. The number of blended splats depends on the configuration: up to 528 million splats (4×33M points) for the default configuration, or 128 million splats (4×8M points) for the fast configuration [S1].
3.  **Global Sorting:** Perform a global sort of the splats. This step is identified as the primary rendering bottleneck [S1].
4.  **Post-Processing:** Apply a U-Net architecture followed by a tone mapping module to the rendered feature images [S1]. This step corrects artifacts and enhances high-frequency details.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_inpc(camera, model, config):
    # 1. Sampling
    # Generate points based on the implicit neural point cloud model
    # config.num_samples determines the base count (e.g., 8M or 33M)
    # 4x multisampling is applied
    points = model.sample(camera, num_samples=config.num_samples, multisample=4)
    
    # 2. Splatting
    # Project points to screen space and rasterize
    # This involves transforming positions and blending contributions
    feature_images = splat(points, camera.resolution)
    
    # 3. Global Sorting
    # Sort splats globally. This is the bottleneck.
    # The sources do not provide the specific sorting algorithm details.
    feature_images = global_sort(feature_images)
    
    # 4. Post-Processing
    # Apply U-Net and tone mapping
    final_image = tone_mapping(unet(feature_images))
    
    return final_image
```

**Parameters:**
*   `num_samples`: The number of samples per training iteration. Default is 33M; Fast configuration is 8M [S1].
*   `multisample`: Set to 4 for inference to blend multiple splats per point [S1].
*   `resolution`: Standard resolutions tested include 2000×1085 pixels [S1].

**Edge Cases and Pitfalls:**
*   **Sorting Bottleneck:** Global sorting significantly impacts real-time performance, making it slower than explicit point-based methods [S1].
*   **Pixel Budget Limits:** Pre-extracting a global point cloud (e.g., 67M points) may be insufficient for high-resolution scenes like Tanks and Temples due to pixel budget limits [S1].
*   **Thin Structures:** The sampling algorithm may miss thin structures if multisampling is disabled [S1].
*   **Temporal Stability:** Minor temporal instability may occur during inference, a common issue with CNN-based post-processing [S1].
*   **Fine Detail Near Camera:** The method may struggle to reconstruct fine geometric details close to the camera, similar to 3DGS and Zip-NeRF [S1].

## Subtopics

### Implicit Neural Point Clouds
The underlying representation that allows for continuous attribute interpolation, distinguishing it from explicit point clouds. [Link to Implicit Neural Point Clouds](implicit-neural-point-clouds.md)

### U-Net Post-Processing
The use of a convolutional neural network to refine rasterized features, addressing artifacts and enhancing detail. [Link to U-Net Rendering](unet-rendering.md)

## Operating characteristics
*   **Time Complexity:** The global sorting step is the bottleneck, leading to higher latency compared to explicit point-based methods [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On an RTX 4090 GPU at 2000×1085 resolution with the 8M configuration:
    *   Sampling: 51ms
    *   Rendering: 16ms
    *   Total: 37ms (~9 fps) [S1].
    *   With pre-extracted global point cloud: Total 26ms [S1].
*   **Scaling:** The method scales with the number of samples (8M vs 33M), with the 33M configuration blending up to 528 million splats [S1].
*   **Strengths:** High image quality, outperforming Zip-NeRF and comparable to explicit methods in perceived quality [S1].
*   **Limitations:** Slower than 3DGS, TRIPS, and ADOP due to sorting and post-processing overhead [S1].

## Background
The INPC method was introduced in the paper "INPC: Implicit Neural Point Clouds for Radiance Field Rendering" [S1]. It builds upon the splatting family of volume rendering techniques, leveraging implicit neural representations to enhance the quality of point-based rendering. The method was evaluated on benchmarks such as Mip-NeRF360 and Tanks and Temples [S1].

## Sources
[S1] INPC: Implicit Neural Point Clouds for Radiance Field Rendering, https://doi.org/10.1109/3dv66043.2025.00021, fetched 2026-10-26
