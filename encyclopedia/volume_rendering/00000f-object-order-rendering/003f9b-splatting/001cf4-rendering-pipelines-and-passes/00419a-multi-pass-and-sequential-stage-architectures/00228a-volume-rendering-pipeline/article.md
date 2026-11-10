# Volume Rendering Pipeline

Also called: CRIO Process, Classification, Reflection, Illumination, Opacity, Rendering Stages, Shading and Classification Pipeline, VolumeRenderer forward method

## Overview
The Volume Rendering Pipeline is a conceptual and implementation structure that decomposes the volume rendering process into a fixed sequence of stages, where the output of one stage serves as the input for the next [S1, S2, S3, unofficial]. It solves the problem of mapping a 3D scalar field to a 2D image by separating geometric traversal and data preparation from optical property assignment and final image composition [S1, S2, unofficial]. Unlike methods that fit geometric primitives to the data, this pipeline directly shades each sample and projects it onto the picture plane, using local gradient vectors as surface normals for shading and separate classification operators for opacity [S2, unofficial]. It is distinct from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by its explicit separation of shading and classification calculations to ensure undistorted visualization, and from the [Deferred Rendering Pipeline](10584-deferred-rendering-pipeline.md) by its direct application of shading models to volume samples rather than deferred geometry-based shading [S2, unofficial].

## How it works
The pipeline typically consists of six stages: data traversal, interpolation, gradient computation, classification, shading and illumination, and compositing [S1, unofficial]. The process begins with acquired scalar values and ends with a final pixel color and opacity.

### Inputs and Outputs
*   **Input:** An array of acquired scalar values $f_0(x_i)$ at voxel locations $x_i = (x_i, y_j, z_k)$ [S2, S3, unofficial].
*   **Output:** A single color $C_\lambda(u, v)$ and opacity $a(u, v)$ for each pixel location $(u, v)$ [S2, S3, unofficial].

### Data Structures
*   **Voxel Database:** Stores the scalar field values. In the pre-classification model, this is often extended or duplicated to store pre-computed colors and opacities [S2, S3, unofficial].
*   **Ray Bundle:** A collection of rays cast from the observer eyepoint through the image plane [S5, unofficial].
*   **Intermediate Arrays:**
    *   Prepared values $f_1(x_i)$: Output of data preparation/traversal [S2, S3, unofficial].
    *   Voxel colors $c_\lambda(x_i)$: Output of the shading model, where $\lambda \in \{r, g, b\}$ [S2, S3, unofficial].
    *   Voxel opacities $\alpha(x_i)$: Output of the classification procedure [S2, S3, unofficial].
    *   Resampled vectors: For each ray, a vector of sample colors $c_\lambda(\tilde{x}_i)$ and opacities $\alpha(\tilde{x}_i)$ computed at evenly spaced locations along the ray [S2, S3, unofficial].

### Steps

1.  **Data Traversal and Preparation:**
    Traverse the 3D scalar field based on its topology (e.g., regular grid, irregular grid) [S1, unofficial]. Perform data preparation which may include correction for non-orthogonal sampling grids, patient motion, contrast enhancement, and interpolation of additional samples [S2, S3, unofficial]. The output is an array of prepared values $f_1(x_i)$ [S2, S3, unofficial].

2.  **Interpolation:**
    Since sampling positions on light rays usually differ from the discrete voxel locations, values on the light ray are interpolated using scalar values close to the actual sampling position [S1, unofficial]. This is typically done using tri-linear interpolation from the eight voxels closest to the sample location [S2, S3, unofficial].

3.  **Gradient Computation:**
    Compute the gradient of the scalar field at sample positions to add visual cues [S1, unofficial]. This allows the use of directional light or Phong shading [S1, S2, unofficial]. The gradient is typically approximated by central differences [S1, unofficial].

4.  **Classification:**
    Map properties of the traversed data set (e.g., scalar value, gradient magnitude) to optical properties using a transfer function [S1, unofficial]. This yields an array of voxel opacities $\alpha(x_i)$ [S2, S3, unofficial]. Classification determines discrete values for primary properties: emitted color, reflected color, and opacity [S4, unofficial].

5.  **Shading and Illumination:**
    Use the prepared values $f_1(x_i)$ (or the original scalar values) as input to a shading model to yield an array of voxel colors $c_\lambda(x_i)$ [S2, S3, unofficial]. Surface shading calculations are performed at every voxel, with local gradient vectors serving as surface normals [S2, unofficial]. Illumination models range from simple identity functions to directional lighting like Phong shading [S4, unofficial].

6.  **Ray Casting and Resampling:**
    Cast rays from the observer eyepoint [S2, S3, unofficial]. For each ray, compute a vector of sample colors $c_\lambda(\tilde{x}_i)$ and opacities $\alpha(\tilde{x}_i)$ by re-sampling the voxel database at $K$ evenly spaced locations $\tilde{x}_i = (x_0 + j \frac{d}{K})$ along the ray [S2, S3, unofficial]. Here, $K$ is the number of evenly spaced sample locations, and $d$ is the ray segment length within the volume [S2, unofficial]. Tri-linear interpolation is used from the pre-computed colors and opacities in the eight closest voxels [S2, S3, unofficial].

7.  **Compositing:**
    Drape a fully opaque background of color $c_{bkg, \lambda}$ behind the dataset [S3, unofficial]. Composite the resampled colors and opacities in back-to-front order to yield a single color $C_\lambda(u, v)$ and opacity $a(u, v)$ for each pixel [S2, S3, unofficial].

### Formulas

**Transparency Formula (Compositing):**
The color $C_{out, \lambda}(\tilde{u}_i)$ of the ray as it leaves a sample location is related to the color $C_{in, \lambda}(\tilde{u}_i)$ as it enters, the sample color $c_\lambda(\tilde{x}_i)$, and the sample opacity $\alpha(\tilde{x}_i)$ by [S2, S3, unofficial]:

$$ C_{out, \lambda}(\tilde{u}_i) = C_{in, \lambda}(\tilde{u}_i)(1 - \alpha(\tilde{x}_i)) + c_\lambda(\tilde{x}_i)\alpha(\tilde{x}_i) $$

The final pixel color $C_\lambda(u, v)$ is obtained by solving this recurrence relation from the back of the volume to the front, starting with the background color [S2, S3, unofficial].

**Gradient Approximation:**
The gradient $\nabla f(x_i)$ is typically approximated using central differences [S1, unofficial]. For a scalar field $f$, the gradient components are computed from neighboring voxel values.

### Parameters
*   **K:** Number of evenly spaced sample locations along a viewing ray [S2, unofficial].
*   **Primary Properties:** Emitted color, reflected color, and opacity [S4, unofficial].

### Edge Cases and Pitfalls
*   **Partial Voluming:** Post-classification shading pipelines are distinct in their handling of partial voluming, which can lead to blurring at high image resolutions if not handled correctly [S1, Subtopics, unofficial].
*   **Gradient Noise:** Gradient computation at sample positions can be sensitive to noise in the scalar field, affecting shading quality [S1, unofficial].
*   **Non-Orthogonal Grids:** Data preparation must correct for non-orthogonal sampling grids to ensure accurate interpolation and shading [S2, S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_rendering_pipeline(acquired_values, voxel_locations, camera, image_size):
    """
    Implements the standard Volume Rendering Pipeline.
    
    Args:
        acquired_values: 3D array of scalar values f0(xi).
        voxel_locations: 3D grid coordinates of the voxels.
        camera: Camera object with position and orientation.
        image_size: Tuple (width, height) of the output image.
        
    Returns:
        final_color: 2D array of RGB colors.
        final_opacity: 2D array of opacity values.
    """
    
    # Step 1: Data Traversal and Preparation
    # Correct for non-orthogonal grids, motion, etc.
    prepared_values = prepare_data(acquired_values, voxel_locations)
    
    # Step 2 & 3: Gradient Computation (for shading normals)
    gradients = compute_gradients(prepared_values, voxel_locations)
    
    # Step 4: Classification
    # Map scalar values to opacities using a transfer function
    opacities = classify(prepared_values)
    
    # Step 5: Shading
    # Map scalar values and gradients to colors
    colors = shade(prepared_values, gradients)
    
    # Step 6: Ray Casting and Resampling
    # Cast rays from camera through each pixel
    rays = cast_rays(camera, image_size)
    
    # Pre-compute resampled colors and opacities for each ray
    # This involves tri-linear interpolation from the colors and opacities arrays
    ray_samples = []
    for ray in rays:
        samples = []
        # K evenly spaced locations along the ray segment within the volume
        for j in range(K):
            sample_pos = ray.origin + j * (ray.direction * ray_length / K)
            # Tri-linear interpolation from the 8 closest voxels in colors and opacities
            c = tri_linear_interpolate(colors, sample_pos)
            a = tri_linear_interpolate(opacities, sample_pos)
            samples.append((c, a))
        ray_samples.append(samples)
        
    # Step 7: Compositing
    final_color = np.zeros((image_size[1], image_size[0], 3))
    final_opacity = np.zeros((image_size[1], image_size[0]))
    
    for i, samples in enumerate(ray_samples):
        # Back-to-front compositing
        # Start with background
        current_color = background_color
        current_opacity = 0.0
        
        for c_sample, a_sample in samples:
            # Transparency formula
            # C_out = C_in * (1 - a) + c_sample * a
            current_color = current_color * (1 - a_sample) + c_sample * a_sample
            current_opacity = current_opacity + a_sample * (1 - current_opacity)
            
        # Map ray index to pixel coordinates
        u, v = ray_to_pixel(i, camera, image_size)
        final_color[v, u] = current_color
        final_opacity[v, u] = current_opacity
        
    return final_color, final_opacity

def prepare_data(values, locations):
    # Correction for non-orthogonal grids, motion, contrast enhancement
    # Interpolation of additional samples
    return values # Simplified

def compute_gradients(values, locations):
    # Central differences approximation
    # Return gradient vectors for each voxel
    pass

def classify(values):
    # Transfer function mapping scalar values to opacity
    pass

def shade(values, gradients):
    # Shading model (e.g., Phong) using gradients as normals
    # Return RGB colors for each voxel
    pass

def cast_rays(camera, image_size):
    # Generate rays for each pixel
    pass

def tri_linear_interpolate(data_array, point):
    # Interpolate from 8 closest voxels
    pass
```

## Subtopics

### [Post-classification Shading Pipeline](7545-post-classification-shading-pipeline.md)
The Post-classification Shading Pipeline (also known as Post-DVRI) is a volume rendering method where the original density volume is interpolated, sample values are classified via transfer functions to yield material, opacity, and color, and shading follows immediately after classification. It is distinct from the Pre-classification Shading Pipeline primarily in its handling of partial voluming and its susceptibility to blurring at high image resolutions.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray ($O(P \cdot K)$), where $P$ is the number of pixels and $K$ is the number of samples per ray [S2, S3, unofficial].
*   **Memory Complexity:** Proportional to the size of the volume data and the intermediate arrays (colors, opacities) [S2, S3, unofficial].
*   **Strengths:** Separation of shading and classification allows for flexible transfer functions and shading models without re-traversing the volume [S2, unofficial]. Direct shading of samples ensures undistorted visualization of 3-D shape [S2, unofficial].
*   **Limitations:** Susceptible to blurring at high resolutions if partial voluming is not handled correctly (in post-classification variants) [S1, unofficial]. Gradient computation can be noisy [S1, unofficial].
*   **When to use:** When flexible optical property assignment and shading models are required, and the volume data is regularly sampled [S2, S3, unofficial].
*   **When not to use:** When real-time performance is critical and the volume is very large, as the pipeline may be computationally expensive due to multiple passes over the data [S1, unofficial].

## Background
The volume rendering pipeline was first reported in June 1987 and presented at the SPIE Medical Imaging II conference in January 1988 [S2, S3, unofficial]. It was developed to address the need for visualizing surfaces from sampled scalar functions of three spatial dimensions without fitting geometric primitives [S2, unofficial]. The pipeline structure has become a common implementation framework for volume rendering, with variations in the order and combination of stages [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA193649: Rendering of Surfaces from Volume Data,, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S3] [PDF] Display of Surfaces from Volume Data, https://graphics.stanford.edu/papers/volume-cga88/volume.pdf, fetched 2026-10-26 (unofficial)
[S4] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
