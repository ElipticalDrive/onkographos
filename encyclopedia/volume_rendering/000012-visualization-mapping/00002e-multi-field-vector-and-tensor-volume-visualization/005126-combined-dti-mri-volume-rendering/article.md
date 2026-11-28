# Combined DTI/MRI Volume Rendering

Also called: Combined MRI and DTI Volume Rendering, Interactive Volume Rendering of Diffusion Tensor Data, Magic Lens Selection for DTI/MRI, Multi-modal volume rendering

## Overview
Combined DTI/MRI Volume Rendering is a technique that integrates the visualization of conventional scalar-valued MRI (providing high-resolution gyral structure and blood vessels) with Diffusion Tensor Imaging (DTI) data in a single volume rendering pass to aid surgical planning and navigation [S1, book]. It sets itself apart from sibling methods by restricting the diffusion tensor color map to the standard RGB model to avoid misleading color mixing, while using MRI data solely to highlight object boundaries via a grayscale transfer function [S1, book]. Unlike methods that apply different transfer functions to a single dataset, this approach selects different transfer functions on different modalities to build a combined image without distorting geometry [S1, book].

## How it works
The method extends a standard GPU-based raycasting volume renderer to handle two distinct data modalities simultaneously [S1, book]. It relies on a single-pass rendering pipeline for interactive performance, utilizing a "magic lens" concept to allow interactive selection of the weighting between MRI context and DTI fiber data [S1, book].

### Inputs and Outputs
*   **Inputs:**
    *   Scalar MRI volume data representing anatomical structures [S1, book].
    *   DTI volume data representing diffusion tensors [S1, book].
    *   A "magic lens" region (defined by position and size) that determines where DTI data is emphasized versus where MRI context is dominant [S1, book].
*   **Outputs:**
    *   A composite RGB image where DTI fibers are color-coded and MRI structures provide geometric context [S1, book].

### Data Structures
The implementation uses a 3D texture-based approach for storing volume data [S1, book].
*   **MRI Data:** Stored as a 3D texture of scalar values [S1, book].
*   **DTI Data:** Stored as a 3D texture of tensor values [S1, book].
*   **Transfer Functions:**
    *   MRI Transfer Function: A 2D lookup table (LUT) mapping scalar value and gradient magnitude to opacity and grayscale color [S1, book].
    *   DTI Transfer Function: Evaluated analytically on the GPU based on tensor properties (e.g., Fractional Anisotropy) and potentially MRI scalar values [S1, book].
*   **Lighting Parameters:** OpenGL light sources are repurposed to encode parameters for the transfer functions [S1, book]. Specifically, unused light sources (typically five or more in medical imaging where only one or two are used) provide global parameters like cut-off angle and exponent for smooth step functions [S1, book].

### Numbered Steps
1.  **Compute Stencil:** Determine the spatial region of interest, often defined by the "magic lens" or clipping planes [S1, book].
2.  **Compute Tensor Volume Rendering:** Cast rays through the DTI volume, sample tensor values along the ray, compute color based on the principal eigenvector for directional coloring, and apply opacity based on FA [S1, book]. The tensor transfer function is evaluated analytically on the GPU to allow real-time manipulation without texture updates [S1, book].
3.  **Compute Conventional MRI Overlay:** Cast rays through the MRI volume, sample scalar values, apply a 2D grayscale transfer function based on value and gradient magnitude to highlight boundaries, and apply special material attributes and gradient-based shading to improve 3D perception of surfaces [S1, book].
4.  **Composite:** Combine the DTI rendering and MRI overlay in a single rendering pass [S1, book]. The magic lens allows the user to interactively adjust the weighting between the DTI fiber visualization and the MRI structural context [S1, book].

### Formulas and Parameters
*   **Tensor Color Mapping:** The color $C$ is derived from the principal eigenvector $\vec{v}$ of the diffusion tensor $\mathbf{D}$ [S1, book]. The specific mapping is restricted to the RGB model [S1, book].
*   **Transfer Function Evaluation:**
    *   MRI TF: $T_{MRI}(v, g) \rightarrow (r, g, b, \alpha)$, where $v$ is scalar value and $g$ is gradient magnitude [S1, book].
    *   DTI TF: $T_{DTI}(tensor, v_{MRI}) \rightarrow (r, g, b, \alpha)$ [S1, book]. The sources note that 2D functions can be based on two scalar values, e.g., MRI signal and FA [S1, book].
*   **Light Parameterization:** Smooth step functions for transfer functions are parameterized using OpenGL light parameters: cut-off angle ($\theta_c$) and exponent ($e$) [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_ray_from_camera(pixel, camera_origin, camera_direction, camera_up, camera_right, pixel_spacing):
    """Generates a ray origin and direction for a given pixel."""
    origin = camera_origin
    direction = camera_direction + (pixel.x * camera_right + pixel.y * camera_up) * pixel_spacing
    return origin, direction / math.linalg.norm(direction)

def sample_3d_texture(volume, point):
    """Performs trilinear interpolation on a 3D texture."""
    # [G] Standard trilinear interpolation implementation
    return volume.interpolate(point)

def compute_gradient(volume, point):
    """Computes gradient magnitude using central differences."""
    # [G] Standard central difference implementation
    dx = (sample_3d_texture(volume, point + (1,0,0)) - sample_3d_texture(volume, point - (1,0,0))) / 2
    dy = (sample_3d_texture(volume, point + (0,1,0)) - sample_3d_texture(volume, point - (0,1,0))) / 2
    dz = (sample_3d_texture(volume, point + (0,0,1)) - sample_3d_texture(volume, point - (0,0,1))) / 2
    return math.sqrt(dx*dx + dy*dy + dz*dz)

def compute_fractional_anisotropy(tensor):
    """Computes FA from a 3x3 symmetric tensor."""
    # [G] Standard FA formula using eigenvalues
    eigenvalues = math.linalg.eigvals(tensor)
    l1, l2, l3 = sorted(eigenvalues, reverse=True)
    numerator = math.sqrt(1.5) * math.sqrt((l1-l2)**2 + (l2-l3)**2 + (l3-l1)**2)
    denominator = math.sqrt(l1**2 + l2**2 + l3**2)
    return numerator / denominator if denominator > 1e-8 else 0.0

def get_principal_eigenvector(tensor):
    """Returns the eigenvector corresponding to the largest eigenvalue."""
    # [G] Standard eigenvector extraction
    eigenvalues, eigenvectors = math.linalg.eigh(tensor)
    return eigenvectors[:, math.argmax(eigenvalues)]

def evaluate_mri_tf(value, gradient):
    """Returns grayscale color and opacity based on 2D LUT lookup."""
    # [G] Standard 2D LUT mapping to grayscale (r=g=b)
    return (value, value, value, gradient)

def evaluate_dti_tf(tensor, fa, light_params):
    """Returns RGB color and opacity based on analytical evaluation."""
    eigenvector = get_principal_eigenvector(tensor)
    color = (abs(eigenvector.x), abs(eigenvector.y), abs(eigenvector.z))
    cutoff = light_params['cut_off_angle']
    exponent = light_params['exponent']
    t = max(0.0, min(1.0, (fa - (1.0 - cutoff)) / cutoff))
    opacity = t * t * (3 - 2 * t) ** exponent
    return (color[0], color[1], color[2], opacity)

def is_inside_lens(point, lens_center, lens_radius):
    """Checks if a point is within the magic lens sphere."""
    return math.linalg.norm(point - lens_center) <= lens_radius

def front_to_back_composite(acc_color, acc_alpha, new_color, new_alpha):
    """Performs standard front-to-back alpha compositing."""
    out_alpha = acc_alpha + new_alpha * (1.0 - acc_alpha)
    out_color = (
        acc_color[0] * acc_alpha + new_color[0] * new_alpha * (1.0 - acc_alpha),
        acc_color[1] * acc_alpha + new_color[1] * new_alpha * (1.0 - acc_alpha),
        acc_color[2] * acc_alpha + new_color[2] * new_alpha * (1.0 - acc_alpha)
    )
    return out_color, out_alpha

def render_combined_dti_mri(mri_volume, dti_volume, lens_center, lens_radius, light_params, camera_params):
    """Renders a combined view of MRI and DTI data in a single pass."""
    image_width, image_height = 1920, 1080
    image = [[(0.0, 0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]
    
    for y in range(image_height):
        for x in range(image_width):
            ray_origin, ray_direction = generate_ray_from_camera((x, y), **camera_params)
            acc_color = (0.0, 0.0, 0.0)
            acc_alpha = 0.0
            
            for t in range(0, 1000, 2):
                point = ray_origin + ray_direction * t
                if not mri_volume.contains(point):
                    break
                    
                mri_val = sample_3d_texture(mri_volume, point)
                mri_grad = compute_gradient(mri_volume, point)
                mri_tf = evaluate_mri_tf(mri_val, mri_grad)
                
                dti_tensor = sample_3d_texture(dti_volume, point)
                fa = compute_fractional_anisotropy(dti_tensor)
                dti_tf = evaluate_dti_tf(dti_tensor, fa, light_params)
                
                in_lens = is_inside_lens(point, lens_center, lens_radius)
                weight_dti = 1.0 if in_lens else 0.2
                weight_mri = 0.2 if in_lens else 1.0
                
                combined_color = (
                    mri_tf[0] * weight_mri + dti_tf[0] * weight_dti,
                    mri_tf[1] * weight_mri + dti_tf[1] * weight_dti,
                    mri_tf[2] * weight_mri + dti_tf[2] * weight_dti
                )
                combined_alpha = (mri_tf[3] * weight_mri + dti_tf[3] * weight_dti) * 0.5
                
                acc_color, acc_alpha = front_to_back_composite(acc_color, acc_alpha, combined_color, combined_alpha)
                if acc_alpha > 0.99:
                    break
            
            image[y][x] = (acc_color[0] / max(acc_alpha, 1e-8), acc_color[1] / max(acc_alpha, 1e-8), acc_color[2] / max(acc_alpha, 1e-8), acc_alpha)
    return image
```

### Edge Cases and Pitfalls
*   **Gray/White Matter Interface:** There is an area of approximately two to three millimeters in the outer gyri where streamline tracking algorithms fail due to ambiguity of tensor information at the interface between gray and white matter [S1, book]. Volume rendering combined with cortex extraction shows the limits of these algorithms [S1, book].
*   **Misleading Line Structures:** Pre-computed line structures (streamlines) can be misleading because they are often started and stopped based on parameters like FA, implying single lines with defined start and end points [S1, book].
*   **Color Model Restriction:** The diffusion tensor color map is restricted to the RGB model [S1, book]. Introducing additional colors from MRI would be misleading [S1, book].

## Operating characteristics
*   **Performance:** The method provides interactive frame rates on a full-screen application [S1, book].
*   **Scalability:** The speed of the algorithm is independent of the size of the magic lens [S1, book]. No change in frame rate was experienced for different sizes of DTI and MRI data sets [S1, book].
*   **Strengths:**
    *   Context-preserving visualization: MRI provides gyral structure and blood vessels, while DTI provides fiber connectivity [S1, book].
    *   Interactive: The magic lens allows real-time adjustment of the weighting between modalities [S1, book].
    *   Single-pass rendering: Efficient compositing of both datasets [S1, book].
*   **Limitations:**
    *   MRI data is volume rendered only to highlight boundaries of objects using a standard 2D grayscale transfer function [S1, book].
    *   Gradient computation can be performed either on the GPU or on the CPU, depending on the available graphics board or user choice [S1, book].

## Background
This method was developed to address the limitations of pseudo-colored slices and fiber tracking algorithms in neuro-surgery and brain connectivity research [S1, book]. It specifically targets the need to understand neural fiber connectivity in brain gyri while maintaining the anatomical context provided by conventional MRI [S1, book]. It was applied to study conditions like Deletion 22q Syndrome, characterized by increased FA and ADC in certain brain areas [S1, book]. The approach contrasts with earlier methods like Wang et al. [WZMK05] by using separate transfer functions for separate modalities rather than modifying a single dataset's transfer function [S1, book].

## Sources
[S1] Interactive Volume Rendering of Diffusion Tensor Data, https://doi.org/10.1007/978-3-540-88378-4_8, fetched 2026-10-26 (book)
