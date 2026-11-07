# GPU-based Shear-Warp

Also called: GPU Shear-Warp, Shear-Warp on the GPU

## Overview
GPU-based Shear-Warp is a hardware-accelerated implementation of the Shear-Warp algorithm [S1, unofficial]. It solves the problem of efficient real-time volume rendering by decomposing the viewing transformation into a 3D shear, a projection, and a 2D warp, utilizing vertex and fragment shaders to perform these operations in parallel [S1, unofficial]. Unlike traditional CPU-based implementations, this method leverages graphics hardware to achieve interactive frame rates, though it is generally slower than GPU-based Ray Marching for equivalent image quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (opacity values) stored in a 3D texture; view parameters (modelview and projection matrices).
*   **Output:** A final 2D image rendered to the screen.

**Data Structures:**
*   **3D Texture:** Stores opacity values and precomputed normals for each voxel. The volume is represented as a set of $n$ quadrilaterals along the z-axis, where $n$ is the number of slices [S1, unofficial].
*   **Vertex Coordinates:** Each vertex of a slice quadrilateral has texture coordinates where one component $d$ is defined as $d = \text{sliceNumber} / \text{sliceCount}$, representing the slice index in the 3D texture [S1, unofficial].

**Parameters:**
*   **Sampling Rate ($s$):** The number of virtual slices inserted between successive input slices using linear interpolation to improve visual quality [S1, unofficial]. Typical values range from 1 to 20 [S1, unofficial].
*   **Lighting Model:** Phong shading is used, requiring ambient ($k_a$), diffuse ($k_d$), and specular ($k_s$) coefficients [S1, unofficial].

**Steps:**

1.  **Preprocessing:**
    *   Read opacity values from storage.
    *   Estimate normals for each voxel $(x, y, z)$ using a simple gradient approach based on neighboring opacity values [S1, unofficial]:
        $$ n(x, y, z) = \frac{1}{2} \begin{pmatrix} op(x + 1, y, z) - op(x - 1, y, z) \\ op(x, y + 1, z) - op(x, y - 1, z) \\ op(x, y, z + 1) - op(x, y, z - 1) \end{pmatrix} $$
    *   Store opacity and normals in GPU memory as a 3D texture [S1, unofficial].

2.  **Pass 1 (Shearing and Shading):**
    *   Construct the view matrix $M_{view}$ as the product of the OpenGL modelview and projection matrices:
        $$ M_{view} = M_{GLModelView} \times M_{GLProjection} $$
    *   Derive the 3D shearing matrix $M_{shear3D}$ from $M_{view}$ [S1, unofficial].
    *   Send $M_{shear3D}$ to the GPU Vertex Shader.
    *   The Vertex Shader applies $M_{shear3D}$ to the quadrilaterals representing the volume slices [S1, unofficial].
    *   The Fragment Shader fetches opacity and normal data from the 3D texture for each fragment.
    *   Compute shading using the Phong model. The outgoing radiance $L_o(p, \omega_o)$ at point $p$ with normal $n$ is:
        $$ L_o(p, \omega_o) = k_a + \frac{k_d}{\pi} L_i(p, \omega_i) (\omega_i \cdot n) + k_s (R(\omega_i, n) \cdot \omega_o)^n $$
        where $\omega_i$ and $\omega_o$ are incoming and outgoing directions, $L_i$ is incident light, and $R(\omega_i, n)$ is the reflection direction [S1, unofficial].
    *   Composite slices back-to-front using hardware alpha-blending to produce an intermediate, distorted image [S1, unofficial].

3.  **Pass 2 (Warping):**
    *   Apply the warping matrix $M_{warp}$ to a textured quadrilateral representing the intermediate image in the GPU Vertex Shader [S1, unofficial].
    *   The Fragment Shader maps the intermediate image onto this warped quadrilateral to yield the final image [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def preprocess_volume(volume_data):
    """
    Loads volume data and computes normals.
    Returns a 3D texture containing (opacity, normal) for each voxel.
    """
    texture_size = volume_data.shape
    gpu_texture = allocate_3d_texture(texture_size)
    
    for x in range(texture_size[0]):
        for y in range(texture_size[1]):
            for z in range(texture_size[2]):
                opacity = volume_data[x, y, z]
                # Estimate normal using central differences
                nx = volume_data[x + 1, y, z] - volume_data[x - 1, y, z]
                ny = volume_data[x, y + 1, z] - volume_data[x, y - 1, z]
                nz = volume_data[x, y, z + 1] - volume_data[x, y, z - 1]
                normal = normalize((nx, ny, nz)) / 2.0
                gpu_texture.set_voxel(x, y, z, (opacity, normal))
    return gpu_texture

def render_gpu_shear_warp(view_matrix, sampling_rate, gpu_texture):
    """
    Renders the volume using the GPU-based Shear-Warp algorithm.
    """
    # Derive shear matrix from view matrix
    M_shear3D = compute_shear_matrix(view_matrix)
    
    # Pass 1: Shearing and Shading
    # Create slice quadrilaterals
    slices = create_slice_quadrilaterals(gpu_texture, sampling_rate)
    
    # Vertex Shader: Apply shear
    sheared_slices = []
    for slice in slices:
        sheared_vertices = apply_matrix(slice.vertices, M_shear3D)
        sheared_slices.append(sheared_vertices)
        
    # Fragment Shader: Shade and Composite
    intermediate_image = clear_framebuffer()
    for slice in reversed(sheared_slices): # Back-to-front
        for fragment in slice.fragments:
            opacity, normal = gpu_texture.fetch(fragment.texture_coords)
            # Phong shading calculation
            radiance = phong_shading(fragment.position, normal, fragment.texture_coords)
            intermediate_image.composite(fragment.position, radiance, opacity)
            
    # Pass 2: Warping
    # Create warped quadrilateral for the intermediate image
    M_warp = compute_warp_matrix(view_matrix)
    warped_quad = create_warped_quadrilateral(intermediate_image, M_warp)
    
    # Final mapping
    final_image = render_textured_quad(warped_quad, intermediate_image)
    return final_image
```

## Subtopics

### [VolumePro 1000 Implementation](8162-volumepro-1000-implementation.md)
A hardware-accelerated method using a "shear-image order" approach that casts rays through pixel centers in an intermediate sample space, eliminating the post-rendering warping step and reducing artifacts.

## Operating characteristics
*   **Performance:** The method achieves real-time rendering for small volumes. For example, a skull model (320×320×46 voxels) renders at 95 fps with a sampling rate of 1 [S1, unofficial]. A knee model (512×512×73 voxels) renders at 134 fps [S1, unofficial].
*   **Scaling:** Frame rate drops linearly as the sampling rate (number of interpolated slices) increases. For the skull model, increasing the sampling rate from 1 to 20 reduces the frame rate from 95 fps to 11 fps (opacity only) or 15 fps (with Phong shading) [S1, unofficial].
*   **Comparison:** It is slower than GPU-based Ray Marching for equivalent quality. For the skull model, Shear-Warp at sampling rate 10 achieves 15 fps, whereas Ray Marching with 400 steps achieves 48 fps [S1, unofficial].
*   **Strengths:** Provides a more convincing volumetric appearance than simple opacity rendering when combined with Phong shading [S1, unofficial].
*   **Limitations:** Slice visibility artifacts appear at low sampling rates (e.g., s=1) [S1, unofficial]. Normal estimation using simple gradients may lack precision compared to more complex methods [S1, unofficial].

## Background
This method was presented as the first GPU implementation of the Shear-Warp algorithm for volume rendering [S1, unofficial]. It utilizes the programmability of graphics hardware (vertex and fragment shaders) to execute the shear, shade, and warp steps in parallel [S1, unofficial]. The approach is particularly suited for medical imaging systems where data is provided as opacity values per voxel [S1, unofficial].

## Sources
[S1] GPU-Based Volume Rendering for Medical Imagery, https://scholarly.org/pdf/display/gpu-based-volume-rendering-for-medical-imagery, fetched 2026-10-28 (unofficial)
