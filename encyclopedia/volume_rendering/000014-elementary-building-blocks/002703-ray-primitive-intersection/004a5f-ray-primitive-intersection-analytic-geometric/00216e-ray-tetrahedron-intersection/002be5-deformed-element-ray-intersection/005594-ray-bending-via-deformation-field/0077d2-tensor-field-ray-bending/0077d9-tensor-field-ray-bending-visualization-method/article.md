# Tensor Field Ray Bending (Visualization Method)

Also called: Lens Simulation, Photon Distribution

## Overview
Tensor Field Ray Bending (Visualization Method) is a rendering technique that visualizes the internal convergent and divergent properties of a tensor volume by treating the volume as an optical element (lens or prism) that distorts light rays [S1]. It solves the problem of visual clutter inherent in glyph-based methods by projecting the integrated effects of the tensor field onto a 2D receiving plane, revealing spatially continuous deformation patterns [S1]. Unlike path tracing which renders 1D line glyphs, this method uses image distortion or photon distribution to show 2D features and continuous changes through image deformation [S1].

## How it works
The method operates by tracing rays through a tensor volume using a specific bending rule derived from the local tensor values. It manifests in two primary variants: **Photon Distribution** (forward tracing) and **Lens Simulation** (backward tracing). Both variants rely on the same underlying ray bending mechanics but differ in ray direction, source, and output interpretation.

### Inputs and Outputs
*   **Input:** A 3D tensor field $T(\mathbf{x})$ defined over a volume; a receiving plane (screen) positioned relative to the volume; for Lens Simulation, an original source image $I_{src}$; for Photon Distribution, a light source emitting white light.
*   **Output:** A 2D image on the receiving plane. In Photon Distribution, this is a distribution pattern of photon deposits representing convergence/divergence. In Lens Simulation, this is a distorted version of $I_{src}$.

### Data Structures
*   **Tensor Field:** Stored as a 3D grid or sparse structure where each cell contains a tensor value (e.g., a $3 \times 3$ matrix). The exact layout depends on the underlying volume representation (e.g., regular grid, unstructured tetrahedra).
*   **Ray State:** A ray is defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and current position $\mathbf{x}$.
*   **Receiving Plane:** A 2D buffer or image array where ray intersections are recorded.
*   **Photon Deposit (Photon Distribution):** A data structure accumulating ray hits on the receiving plane, potentially weighted by color components (R, G, B).

### Ray Bending Rule
The core mechanism is the alteration of ray direction $\mathbf{d}$ as it traverses the tensor field. The bending force is derived from the local tensor value $T(\mathbf{x})$. While the specific mathematical formula for the bending force is not explicitly detailed in the provided notes, it is described as a "transforming force" that deforms the ray path [S1]. The path of a ray is the integral of these bending forces along the path [S1].

### Steps: Photon Distribution
This variant simulates the tensor volume as a prism dispersing white light.

1.  **Setup:** Position the tensor volume between a light source and a receiving plane.
2.  **Light Emission:** Shoot beams of white light toward the tensor field. White light is represented as having equal amounts of red, green, and blue components [S1].
3.  **Ray Tracing:** Trace rays through the tensor volume.
    *   Different wavelengths (colors) have different indices of refraction, causing their bent paths to differ slightly [S1].
    *   Apply the bending rule at each step within the volume to update the ray direction.
4.  **Photon Deposit:** When rays exit the tensor field, record their intersections on the receiving plane.
    *   Rays of different wavelengths originating from the same direction and position will land at different positions on the receiving plane [S1].
    *   These intersections act as photon deposits, forming a distribution pattern similar to photon mapping [S1].
5.  **Image Generation:** Combine the distribution patterns from different wavelengths (colors) to generate the final image. The resulting image represents the converging and diverging properties of the tensor field [S1].

### Steps: Lens Simulation
This variant simulates the tensor volume as a lens distorting an image.

1.  **Setup:** Position the tensor volume between an original image $I_{src}$ and a receiving plane (viewer).
2.  **Ray Casting:** Cast a set of parallel rays from the pixels of $I_{src}$ toward the viewer, passing through the tensor volume. This is analogous to backward ray tracing [S1].
3.  **Ray Bending:** While rays are within the tensor field, alter their directions using the bending force defined by the local tensors [S1].
4.  **Intersection:** Let rays leave the field in different locations and orientations. Record where they hit the receiving plane [S1].
5.  **Image Reconstruction:** Generate a distorted image on the receiving plane.
    *   Color each pixel on the receiving plane according to the color of the corresponding pixel from the original image $I_{src}$ [S1].
    *   Assign texture coordinates of each hit point to the original pixel, preserving neighborhood information [S1].

### Parameters and Typical Values
*   **Ray Flexibility:** The sensitivity of the ray paths to the bending rule parameters. Changing ray flexibility results in different paths [S1].
*   **Wavelengths:** For Photon Distribution, at least three wavelengths (Red, Green, Blue) are used to simulate white light dispersion [S1].
*   **Viewing Angles:** Multiple images from different angles are recommended to determine local tensor properties, as each ray represents an integrated sum of effects [S1].

### Edge Cases and Pitfalls
*   **Sensitivity to Parameters:** The algorithm is sensitive to the parameters used in the bending rule. Paths differ when ray flexibility changes [S1].
*   **Interpretation Difficulty:** Each ray represents the integrated sum of tensor effects along its entire path. It is difficult to determine local tensor properties from a single image. Multiple images from different angles are required for accurate local analysis [S1].
*   **Visual Clutter:** While this method reduces clutter compared to path tracing (glyphs), improper parameter tuning can still lead to ambiguous patterns [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tensor_field_ray_bending(tensor_field, method, receiving_plane, original_image=None):
    """
    Visualize tensor field effects using ray bending.
    
    Args:
        tensor_field: 3D structure containing tensor values T(x).
        method: 'photon_distribution' or 'lens_simulation'.
        receiving_plane: 2D buffer for recording ray hits.
        original_image: Source image for lens simulation (required if method is 'lens_simulation').
    
    Returns:
        2D image on the receiving plane.
    """
    if method == 'photon_distribution':
        return photon_distribution_algorithm(tensor_field, receiving_plane)
    elif method == 'lens_simulation':
        if original_image is None:
            raise ValueError("Original image required for lens simulation")
        return lens_simulation_algorithm(tensor_field, receiving_plane, original_image)
    else:
        raise ValueError("Unknown method")

def photon_distribution_algorithm(tensor_field, receiving_plane):
    """
    Simulates tensor volume as a prism dispersing white light.
    """
    # Initialize receiving plane buffer for R, G, B components
    photon_buffer_R = zeros_like(receiving_plane)
    photon_buffer_G = zeros_like(receiving_plane)
    photon_buffer_B = zeros_like(receiving_plane)
    
    # Define wavelengths for white light
    wavelengths = [
        {'color': 'R', 'index_of_refraction': n_red},   # Not given in sources, placeholder for implementation
        {'color': 'G', 'index_of_refraction': n_green}, # Not given in sources, placeholder for implementation
        {'color': 'B', 'index_of_refraction': n_blue}   # Not given in sources, placeholder for implementation
    ]
    
    for wl in wavelengths:
        # Shoot beams of white light (equal components, but traced separately for dispersion)
        # Note: Sources say "shoot beams of white light... equal amounts of R, G, B"
        # but then "different wavelengths have different indices... end up with different positions".
        # This implies tracing R, G, B rays separately from the same source points.
        
        rays = generate_white_light_rays() # Not given in sources, assumes uniform grid or random sampling
        
        for ray in rays:
            current_ray = ray
            # Trace through tensor volume
            while current_ray.is_inside(tensor_field):
                # Apply bending force based on local tensor T(x)
                # Formula for bending force not given in sources [G]
                bending_force = compute_bending_force(tensor_field.get_tensor(current_ray.position), wl['index_of_refraction'])
                current_ray.direction += bending_force * dt
                
                # Advance ray
                current_ray.position += current_ray.direction * dt
            
            # Record intersection on receiving plane
            hit_point = current_ray.intersect(receiving_plane)
            if hit_point is not None:
                if wl['color'] == 'R':
                    photon_buffer_R[hit_point] += 1
                elif wl['color'] == 'G':
                    photon_buffer_G[hit_point] += 1
                elif wl['color'] == 'B':
                    photon_buffer_B[hit_point] += 1

    # Combine distribution patterns
    final_image = combine_buffers(photon_buffer_R, photon_buffer_G, photon_buffer_B)
    return final_image

def lens_simulation_algorithm(tensor_field, receiving_plane, original_image):
    """
    Simulates tensor volume as a lens distorting an image.
    Uses backward ray tracing.
    """
    distorted_image = zeros_like(receiving_plane)
    
    # Cast parallel rays from original image toward viewer
    for y in range(original_image.height):
        for x in range(original_image.width):
            # Ray origin is on the original image plane
            ray_origin = get_pixel_position(original_image, x, y)
            ray_direction = vector_to_viewer # Not given in sources, assumes fixed view direction
            
            current_ray = Ray(ray_origin, ray_direction)
            
            # Trace through tensor volume
            while current_ray.is_inside(tensor_field):
                # Alter ray direction using bending force defined on tensors
                # Formula for bending force not given in sources [G]
                tensor_val = tensor_field.get_tensor(current_ray.position)
                bending_force = compute_bending_force(tensor_val)
                current_ray.direction += bending_force * dt
                
                # Advance ray
                current_ray.position += current_ray.direction * dt
            
            # Ray leaves field, hits receiving plane
            hit_point = current_ray.intersect(receiving_plane)
            
            if hit_point is not None:
                # Color pixel according to original image pixel
                # Assign texture coordinates to preserve neighborhood info
                distorted_image[hit_point] = original_image[x, y]
                
    return distorted_image
```

## Subtopics

### Path Tracing (Tensor Field)
Renders 1D line glyphs along bent ray paths. Prone to visual clutter. [Link](23804-tensor-field-ray-bending.md)

### Photon Distribution
Forward tracing variant. Treats volume as a prism. Good for showing convergence/divergence via color dispersion. [Link](23804-tensor-field-ray-bending.md)

### Lens Simulation
Backward tracing variant. Treats volume as a lens. Good for showing spatially continuous deformation of an image. [Link](23804-tensor-field-ray-bending.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays and the number of steps per ray within the tensor volume. Since rays are traced through the entire volume, complexity is proportional to $N_{rays} \times N_{steps}$.
*   **Memory Complexity:** Proportional to the size of the tensor field and the receiving plane buffer. For Photon Distribution, buffers for R, G, B components are needed.
*   **Strengths:**
    *   Reveals 2D features and spatially continuous depiction of tensor properties [S1].
    *   Shows continuous changes through image deformation [S1].
    *   Can work directly with non-symmetric tensor fields without decomposition [S1].
    *   Lens Simulation avoids the visual clutter of path tracing [S1].
*   **Limitations:**
    *   Interpretation difficulty: Rays represent integrated effects, making local analysis hard without multiple views [S1].
    *   Sensitive to bending rule parameters (ray flexibility) [S1].
*   **When to Use:** When the goal is to visualize the global convergent/divergent nature of a tensor field or to see how it distorts an image, rather than tracking individual streamlines.
*   **When Not to Use:** When local tensor properties need to be determined from a single view, or when high precision in ray path tracking is required without parameter tuning.

## Background
The method was introduced in the paper "Interaction of Light and Tensor Fields" by the authors of the VISSYM 2003 proceedings [S1]. It is part of a suite of three new techniques (Path Tracing, Photon Distribution, Lens Simulation) designed to analyze complex tensor fields. The approach is inspired by optical phenomena such as caustics, photon mapping, and lens distortion. It allows for the visualization of non-symmetric tensor fields and leverages the continuity of ray paths to reveal internal properties.

## Sources
[S1] Interaction of Light and Tensor Fields, https://doi.org/10.2312/vissym/vissym03/157-166, fetched 2026-10-26
