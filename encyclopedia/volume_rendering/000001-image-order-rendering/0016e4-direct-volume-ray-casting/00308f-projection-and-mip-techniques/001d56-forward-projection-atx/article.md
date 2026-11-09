# Forward Projection (aTx)

Also called: Fan Beam Projection, Parallel Beam Projection, X-ray projection, aTx

## Overview
Forward Projection (also known as X-ray projection or aTx) is a volume rendering technique that computes the line integral of the volume density along each viewing ray, simulating the attenuation of X-rays passing through an object [S1]. It solves the problem of generating 2D projection images from 3D volumetric data, which is fundamental in computed tomography (CT) and digital radiography [S1]. Unlike Maximum Intensity Projection (MIP) which highlights the highest density voxel, Forward Projection aggregates all encountered densities, providing a cumulative measure of material thickness that clearly represents dense structures like bones or vessels at the cost of losing depth information [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume $V$ of dimensions $W \times H \times D$, a ray origin $\mathbf{o}$, and a ray direction $\mathbf{d}$ for each pixel.
*   **Output:** A 2D image $I$ where each pixel value represents the integrated density along the corresponding ray.

**Data Structures:**
*   **Volume:** Stored as a 3D array. Access is typically linearized. For a batch of images, the index is calculated as `batch_index * channels * height * width + channel_index * height * width + img_i * width + img_j` [S2, unofficial].
*   **Accumulator:** A scalar value initialized to $0.0$ for each ray [G, parent].

**Steps:**
1.  **Ray Intersection:** Determine the entry and exit points of the ray $r(t) = \mathbf{o} + t\mathbf{d}$ with the volume bounding box [Parent].
2.  **Discretization:** Divide the ray segment inside the volume into $N$ samples with step size $\Delta t$ [Parent].
3.  **Sampling:** For each sample point $t_i$, retrieve the scalar value $s_i$ from the volume using trilinear interpolation [Parent].
4.  **Accumulation:** Update the accumulator $A$ by adding the product of the sample value and the step size:
    $$ A \leftarrow A + s_i \cdot \Delta t $$
    [S1][S2, unofficial]
5.  **Finalization:** The final value $A$ is the projection value for that pixel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_projection(volume, ray_origin, ray_direction, num_samples, step_size):
    """
    Computes the forward projection (line integral) for a single ray.
    
    Args:
        volume: 3D array representing the density field.
        ray_origin: 3D vector (x, y, z).
        ray_direction: 3D vector (x, y, z), assumed normalized.
        num_samples: Integer number of samples along the ray.
        step_size: Float distance between samples.
        
    Returns:
        float: The accumulated line integral.
    """
    accumulator = 0.0
    
    for i in range(num_samples):
        # Calculate current position along the ray
        t = i * step_size
        current_pos = ray_origin + ray_direction * t
        
        # Retrieve density via trilinear interpolation [Parent]
        # Note: Implementation of trilinear interpolation depends on volume bounds
        # and normalization. Here we assume a helper `sample_volume` exists.
        density = sample_volume(volume, current_pos)
        
        # Accumulate: sum of density * step size [S1][S2, unofficial]
        accumulator += density * step_size
        
    return accumulator
```

**Parameters:**
*   **num_samples:** Determines the resolution of the ray traversal. Higher values increase accuracy but cost more time.
*   **step_size ($\Delta t$):** The distance between samples. It should be small enough to avoid aliasing but large enough for performance. Typically related to the voxel size [Parent].

**Edge Cases and Pitfalls:**
*   **Loss of Depth Information:** The method sums all densities along the ray, meaning structures at different depths are indistinguishable in the final 2D image [S1].
*   **Boundary Conditions:** If the ray does not intersect the volume, the accumulator remains $0.0$ [G].
*   **Precision:** Accumulating many small values can lead to floating-point precision errors; using double precision for the accumulator is recommended for high-accuracy applications [G].

## Subtopics

### [Analytic 3D Gaussian Ray Tracing for Tomography](3017-analytic-3d-gaussian-ray-tracing-for-tomography.md)
This method replaces the local affine approximation of splatting with exact ray tracing to compute line integrals through 3D Gaussian primitives analytically. It solves the problem of quantitative accuracy degradation and geometric inflexibility in tomographic reconstruction, specifically addressing nonlinear corrections like arc-correction in PET scanners. Unlike splatting-based pipelines which project into 2D screen space, this approach directly computes ray-primitive intersections, avoiding the assumption of non-overlapping Gaussians.

### CT Initialization via Fan Beam Ray Transform and FBP
This variant initializes CT reconstruction by computing a reference image via Fan beam ray transform and filtered back projection (FBP), then creates a sparse sinogram by subsampling and resizing [S3, unofficial].
**Steps:**
1.  Convert input image to CPU numpy array [S3, unofficial].
2.  Initialize sinogram, sparse_sinogram, and ori_img arrays with shape `[batch, 720, 720]` [S3, unofficial].
3.  For each batch element: compute sinogram via `Fan_ray_trafo(img[i])` [S3, unofficial].
4.  Compute reference image `ori_img[i]` via `Fan_FBP(sinogram[i])` [S3, unofficial].
5.  Store sinogram max value [S3, unofficial].
6.  Subsample sinogram along first axis with stride `r`: `t = np.copy(sinogram[i,::r,:])` [S3, unofficial].
7.  Resize `t` to `[720, 720]` using `cv2.resize` and store in `sparse_sinogram` [S3, unofficial].
**Parameters:**
*   **r:** Subsampling stride for the sinogram along the angular/detector axis [S3, unofficial].
**Layout:**
*   Output arrays: `ori_img`, `sparse_sinogram`, `sinogram` all have shape `[batch, 720, 720]` and dtype `float32`. `sinogram_max` has shape `[batch, 1]` [S3, unofficial].

### Cylindrical Symmetry Projector
A forward/back projector pair for objects with cylindrical symmetry [S4, unofficial]. The Abel Transform is a special case of this applied to parallel-beam geometries [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of pixels (rays) [Parent].
*   **Memory Complexity:** $O(V)$ for the volume storage, $O(P)$ for the output image [Parent].
*   **Performance:** Generally faster than alpha blending because it does not require sorting or complex blending calculations [S1].
*   **Strengths:** Clearly represents dense structures like bones or vessels; computationally efficient; order-independent [S1, Parent].
*   **Limitations:** Loses depth information; static images may lack context compared to volume-rendered images [S1].
*   **When to use:** When generating 2D projection images for CT reconstruction initialization or quick visualization of dense structures [S1][S3, unofficial].
*   **When not to use:** When depth perception or transparency effects are required [S1].

## Background
Forward Projection is a fundamental technique in medical imaging, particularly in Computed Tomography (CT) and Digital Radiography [S1]. It simulates the physical process of X-ray attenuation, where the intensity of the X-ray beam decreases exponentially as it passes through matter [S1]. The method is widely used in the forward pass of differentiable tomography pipelines and for initializing reconstruction algorithms [S2, S3, S4, unofficial].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S3] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S4] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
