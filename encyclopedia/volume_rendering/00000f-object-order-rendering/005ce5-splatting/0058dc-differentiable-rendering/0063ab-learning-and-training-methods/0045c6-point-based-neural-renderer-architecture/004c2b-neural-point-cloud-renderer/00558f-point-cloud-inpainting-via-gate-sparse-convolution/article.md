# Point Cloud Inpainting via Gate/Sparse Convolution

Also called: Gate Convolution Inpainting, Image Inpainting Baseline for Point Cloud Rendering, Point Cloud Inpainting, Sparse Convolution Inpainting

## Overview
This method serves as a baseline for evaluating neural point cloud rendering by treating the completion of sparse point cloud projections as a standard 2D image inpainting problem [S1]. It differs from the parent [Neural Point Cloud Renderer](13165-neural-point-cloud-renderer.md) by bypassing explicit 3D-to-2D splatting and blending logic in favor of direct 2D convolutional synthesis, and from [Implicit Neural Point Clouds (INPC)](18226-implicit-neural-point-clouds-inpc.md) by operating on discrete image grids rather than implicit fields [G].

## How it works
**Inputs and Outputs:**
- Input: A 2D image representation of a sparse point cloud projected from a target viewpoint, containing valid pixel colors where points project and empty/invalid regions where no points exist [S1].
- Output: A completed 2D image with filled-in colors for the missing regions [S1].

**Data Structures:**
- The input is treated as a standard 2D tensor (image) with channels (e.g., RGB) [G].
- No explicit 3D data structures are used in this specific baseline step; the 3D point cloud $P_i$ has already been projected to form the 2D input [G].

**Method Steps:**
1. **Projection:** Project the 3D point cloud $P_i$ onto the 2D image plane using the pinhole camera model: $u = f_x \frac{x}{z} + c_x$, $v = f_y \frac{y}{z} + c_y$, where $(x, y, z)$ are the 3D coordinates, $(f_x, f_y)$ are the focal lengths, $(c_x, c_y)$ is the principal point, and $(u, v)$ are the pixel coordinates. For each pixel, if a point projects to it, store its color; otherwise, mark the pixel as invalid/empty [G].
2. **Backbone Selection:** Adopt a U-Net architecture as the backbone network [S1].
3. **Convolution Replacement:** Replace standard convolution operations within the U-Net with Gate Convolution (GateConv) [6] and Sparse Convolution (SparseConv) [5], which were proposed in state-of-the-art image inpainting methods to fill up irregular holes [S1].
4. **Training/Inference:** The network is trained or applied to predict the missing pixel values based on the surrounding valid context [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_cloud_inpainting_baseline(point_cloud, camera_params):
    h, w = camera_params['image_height'], camera_params['image_width']
    input_image = [[None, None, None] for _ in range(h)]
    
    for x, y, z, r, g, b in point_cloud:
        u = int(camera_params['fx'] * x / z + camera_params['cx'])
        v = int(camera_params['fy'] * y / z + camera_params['cy'])
        if 0 <= u < w and 0 <= v < h:
            input_image[v][u] = [r, g, b]
            
    # Initialize U-Net with GateConv or SparseConv layers replacing standard convolutions
    # (U-Net architecture details are not specified in the sources)
    completed_image = model(input_image)
    return completed_image
```

**Parameters:**
- **U-Net Depth:** Not given in the sources.
- **Convolution Type:** Either GateConv or SparseConv.
- **Training Data:** Not given in the sources.

**Edge Cases and Pitfalls:**
- Directly inpainting sparse point cloud images generates blurry results compared to the proposed method [S1]. This occurs because the color of directly projected points differs from ground truth images due to view-dependent effects (e.g., specular highlights) and different exposure times [S1].
- Irregular holes in the projected image correspond to missing parts of the 3D object [S1].

## Subtopics

### [Completion Framework](18918-completion-framework.md)
The Completion Framework repairs and complements incomplete point clouds derived from RGB or RGB-D frames that only partially capture an object. It distinguishes itself from standard completion methods by using two separate encoders to produce distinct latent representations for the existing and missing parts of the point cloud, which are then concatenated and decoded to output the weights of a Neural Radiance Field (NeRF). This approach generates a complete, colored point cloud by enforcing a Gaussian prior on the missing part's latent space while maintaining multi-view consistency with scanned images.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The baseline produces blurry results compared to the proposed method in the source paper [S1].
- **Strengths:** Simple to implement as it leverages existing 2D image inpainting architectures (U-Net) and operations (GateConv, SparseConv) [G].
- **Limitations:** Fails to capture 3D geometric consistency and view-dependent appearance, leading to blurriness [S1].
- **When to use:** As a baseline for comparison against more advanced neural point cloud rendering methods [S1].
- **When not to use:** When high-fidelity, sharp rendering of specular or view-dependent effects is required [S1].

## Background
This method was introduced as a baseline in the paper "Neural Point Cloud Rendering via Multi-Plane Projection" [S1]. It utilizes techniques from state-of-the-art image inpainting, specifically Gate Convolution [6] and Sparse Convolution [5], to address the problem of incomplete point cloud projections [S1]. The authors found that direct 2D inpainting of projected point clouds resulted in blurry outputs, motivating the development of their more advanced neural point cloud rendering approach [S1].

## Sources
[S1] Neural Point Cloud Rendering via Multi-Plane Projection, https://doi.org/10.1109/cvpr42600.2020.00785, fetched 2026-10-30
