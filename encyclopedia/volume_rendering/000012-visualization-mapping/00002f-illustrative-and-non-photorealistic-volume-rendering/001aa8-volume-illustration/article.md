# Volume Illustration

Also called: Ebert and Rheingans volume illustration, Non-photorealistic Volume Rendering

## Overview
Volume Illustration is a method that augments a physics-based volume rendering pipeline with non-photorealistic rendering (NPR) techniques to enhance the perception of structure, shape, orientation, and depth relationships in volumetric data [S1, unofficial][S3]. It solves the problem of ambiguous structural interpretation in traditional physics-based renderings by applying stylized enhancements such as boundary detection, silhouettes, and tone shading [S2, unofficial][S3]. Unlike surface-based NPR, which relies on well-defined geometric surfaces, Volume Illustration determines features through the analysis of local volumetric properties that vary continuously throughout 3D space [S3][S4, unofficial]. It is distinguished from its siblings by its unified framework that decouples illumination from physical light transport, using mathematical functions of view, light, and scalar field properties to modulate color and opacity [S3857].

## How it works
The method operates by modifying the standard volume rendering pipeline. It begins with a base renderer that implements physics-based illumination (e.g., atmospheric or Phong illumination with linear accumulation) [S3]. The core innovation lies in the transformation of voxel properties before compositing.

**Inputs and Outputs:**
*   **Input:** A 3D scalar field $V(x,y,z)$, a transfer function $TF$ mapping scalar values to base color $c_{base}$ and opacity $\alpha$, view direction $\vec{v}$, and light direction $\vec{l}$.
*   **Output:** A final 2D image where voxels contribute to the final color via standard volume compositing, but with modified color and opacity values derived from NPR transformations [S2, unofficial][S3].

**Data Structures:**
For each voxel sample $i$ along a ray, the renderer computes or accesses:
1.  **Scalar Value:** $s_i = V(x_i, y_i, z_i)$.
2.  **Gradient:** $\nabla f_i = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})_i$. This is normalized to $\nabla f_{n,i}$ to serve as a proxy for surface normal orientation [S3857].
3.  **Minimal Change Direction:** The direction of least variation in the scalar field, often derived from the gradient's orthogonal complement or principal component analysis of local neighborhoods [S3][S4, unofficial].
4.  **Base Color and Opacity:** $c_{base,i}$ and $\alpha_i$ obtained from the transfer function applied to $s_i$ [S2, unofficial].

**Numbered Steps:**
1.  **Classification:** Apply the transfer function to the voxel's scalar value to obtain initial color $c_{base}$ and opacity $\alpha$ [S2, unofficial].
2.  **Feature Analysis:** Compute local volumetric properties for the voxel:
    *   Calculate the gradient $\nabla f$.
    *   Determine the minimal change direction [S3][S4, unofficial].
    *   Retrieve view direction $\vec{v}$ and light direction $\vec{l}$ [S3].
3.  **NPR Transformation:** Apply transformation functions to $c_{base}$ and $\alpha$ using the properties from Step 2. These transformations encapsulate specific illustration techniques (e.g., boundary enhancement, tone shading). The transformed values are $c_{final}$ and $\alpha_{final}$ [S2, unofficial][S3].
    *   *Note:* The specific mathematical forms of these transformations vary by technique (see Subtopics). Common parameters include sample location, value, gradient magnitude, view angle, and light angle [S2, unofficial][S3].
4.  **Compositing:** Perform standard front-to-back volume compositing using the transformed $c_{final}$ and $\alpha_{final}$. The compositing equation remains unchanged from physics-based rendering:
    $$
    C_{out} = \sum_{i=1}^{N} c_{final,i} \cdot \alpha_{final,i} \cdot \prod_{j=1}^{i-1} (1 - \alpha_{final,j})
    $$
    All voxels in the volume still contribute to the final image [S2, unofficial].

**Formulas and Parameters:**
*   **Gradient Normalization:** $\nabla f_n = \frac{\nabla f}{\|\nabla f\|}$ [S3857].
*   **Boundary Enhancement:** Often triggered when $\|\nabla f\|$ exceeds a threshold, increasing opacity or altering color to highlight edges [S3][S4, unofficial].
*   **Tone Shading:** Modulates brightness based on the dot product $\nabla f_n \cdot \vec{l}$ and $\nabla f_n \cdot \vec{v}$ [S3, S3857].
*   **Parameters:** Tunable parameters include thresholds for gradient magnitude, intensity of halos, and color blending factors. These parameters are dataset-dependent [S2, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_illustration_render(ray_origin, ray_direction, volume_data, transfer_function, light_direction):
    """
    Renders a single ray using the Volume Illustration framework.
    """
    color_accum = [0.0, 0.0, 0.0]
    alpha_accum = 0.0
    
    # Iterate along the ray
    for sample in ray_samples(ray_origin, ray_direction, volume_data):
        # 1. Classification
        scalar_val = sample.value
        base_color, base_alpha = transfer_function(scalar_val)
        
        # 2. Feature Analysis
        # Compute gradient at sample position
        gradient = compute_gradient(sample.position, volume_data)
        gradient_norm = normalize(gradient)
        
        # Get view direction (from sample to camera)
        view_direction = normalize(camera_position - sample.position)
        
        # 3. NPR Transformation
        # Apply transformations based on local properties
        # Example: Boundary enhancement increases opacity if gradient is high
        if magnitude(gradient) > boundary_threshold:
            final_alpha = base_alpha * boundary_enhancement_factor
            final_color = base_color * edge_color_multiplier
        else:
            final_alpha = base_alpha
            final_color = base_color
            
        # Example: Tone shading modulates color based on light and view
        # Note: Specific shading models vary; this is a generic placeholder for the transformation step
        lighting_factor = dot(gradient_norm, light_direction)
        final_color = final_color * (ambient_light + diffuse_coeff * max(0, lighting_factor))
        
        # 4. Compositing
        # Standard front-to-back compositing
        contribution = final_color * final_alpha * (1.0 - alpha_accum)
        color_accum = color_accum + contribution
        alpha_accum = alpha_accum + final_alpha * (1.0 - alpha_accum)
        
        if alpha_accum > 1.0 - epsilon:
            break
            
    return color_accum
```
*Note: The specific transformation logic in Step 3 is highly variable depending on the chosen NPR technique (e.g., stippling, hatching, halos). The pseudocode above illustrates the pipeline structure. For specific techniques, refer to the Subtopics.* [S2, unofficial][S3]

## Subtopics

### [Interest-Based NPR Toolkit](1158-interest-based-npr-toolkit.md)
A flexible framework that associates user-defined "interest labels" with subsets of rendering techniques, allowing variable detail levels and artistic styles per segment. It solves the problem of programmatic bias by prioritizing communicative abstraction over physical accuracy.

### [Interactive Volume Illustration (Hatching)](5272-interactive-volume-illustration-hatching.md)
Simulates traditional pen-and-ink hatching styles interactively, using local volumetric properties to determine stroke placement and density.

### [Stipple Volume Rendering](6321-stipple-volume-rendering.md)
Generates stipple drawing effects by varying dot density and size based on scalar values and gradients, creating a textured, illustrative appearance.

### [Line and Edge Detection](5341-line-and-edge-detection.md)
Focuses on extracting and enhancing contours and edges from the volume data, often using gradient analysis to identify feature boundaries for sketch-like rendering.

### [Feature Halos](799-feature-halos.md)
Adds glowing or darkened outlines around features to enhance depth perception and separation between structures, using gradient and view direction information.

### [Geometric Buffers (G-buffers)](781-geometric-buffers-g-buffers.md)
Uses intermediate buffers storing geometric information (normals, depth) to guide NPR techniques, though adapted for volumetric data by deriving normals from gradients.

### [Base Gaseous Illumination Renderer](794-base-gaseous-illumination-renderer.md)
Provides the underlying physics-based illumination model (e.g., atmospheric scattering) that Volume Illustration augments with NPR effects.

### [Orientation-Based Color Modification](798-orientation-based-color-modification.md)
Modifies colors based on the orientation of the gradient relative to the view and light directions, enhancing structural cues.

### [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md)
Maintains global context while applying local NPR enhancements, often using focus+context techniques like two-level rendering.

### [Sketch Rendering via Silhouette Transparency](816-sketch-rendering-via-silhouette-transparency.md)
Enhances silhouettes and sketch lines by modulating transparency and color along boundaries, creating a pen-and-ink look.

### [Exact Projection Full Opacity](1882-exact-projection-full-opacity.md)
A technique for rendering fully opaque features with precise boundaries, often used in conjunction with NPR to define clear feature extents.

### [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md)
Defines the mathematical models for lighting and shading used in Volume Illustration, decoupling illumination from physical transport.

### [Multi-object volume rendering with ghosting](6421-multi-object-volume-rendering-with-ghosting.md)
Renders multiple objects with ghosting effects to show internal structure while maintaining external context, often using NPR techniques for clarity.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. The additional cost comes from computing gradients and applying NPR transformations per sample, which is constant time relative to $N$ but increases the constant factor [S2, unofficial][S3].
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(1)$ per sample for temporary storage of gradient and transformed properties [S3].
*   **Measured Performance:** Tested on datasets such as $256 \times 256 \times 128$ abdominal CT volumes [S3]. Performance is comparable to standard volume rendering, with overhead dependent on the complexity of the NPR transformations [S2, unofficial].
*   **Scaling:** Scales linearly with resolution and dataset size. Interactive frame rates are achievable with optimization techniques like adaptive sampling or GPU acceleration [S1, unofficial][S3].
*   **Strengths:** Enhances structural perception significantly more than physics-based rendering alone; requires less manual tuning of transfer functions than traditional methods; flexible framework allowing combination of multiple NPR techniques [S2, unofficial][S3].
*   **Limitations:** Local implementation requires significant parameter tuning dependent on exact voxel intensity values and dataset characteristics; features are not as well-defined as in surface rendering, leading to potential ambiguity in feature detection [S2, unofficial][S3][S4, unofficial].
*   **When to Use:** When the goal is to create illustrative, textbook-quality images that emphasize structure and depth; when traditional volume rendering fails to convey internal structure clearly [S2, unofficial][S3].
*   **When Not to Use:** When physical accuracy is required; when real-time performance is critical and computational resources are limited; when the dataset has low contrast or noisy gradients that make feature detection unreliable [S2, unofficial][S3].

## Background
Volume Illustration was introduced by Penny Rheingans and David Ebert in 2000 (Vis '00) and published in TVCG in July 2001 [S1, S4, unofficial]. It combines the familiarity of physics-approximated illumination with the expressive power of NPR techniques [S1, unofficial][S3]. The method was developed to address the limitations of traditional volume rendering in conveying complex 3D structures, particularly in medical imaging and scientific visualization [S2, unofficial][S3]. Popular styles include stippling, hatching, and silhouettes, drawing inspiration from the pen-and-ink family of illustration [S1, unofficial][S3]. The framework allows for the augmentation of a physics-based renderer with various NPR techniques, providing a unified approach to enhancing volumetric data [S3].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S3] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S4] PDF Transfer Functions for Direct Volume Rendering, https://www.cs.purdue.edu/homes/xmt/classes/slides/CS530/Transfer_Functions.pdf, fetched 2026-10-28 (unofficial)
