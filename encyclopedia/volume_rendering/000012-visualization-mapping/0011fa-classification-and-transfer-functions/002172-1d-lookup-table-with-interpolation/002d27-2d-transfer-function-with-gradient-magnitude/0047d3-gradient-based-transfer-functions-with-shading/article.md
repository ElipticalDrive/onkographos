# Gradient-Based Transfer Functions with Shading

Also called: Gradient-Based Transfer Functions, Ray Caster with Gradient-Based Transfer Functions, Shaded Gradient TF

## Overview
This method enhances volume visualization by precomputing surface normals and shaded intensity values to drive a 2D opacity transfer function based on gradient magnitude, while using a 1D transfer function for color/intensity. It solves the problem of flat, featureless volume rendering by providing shading cues that reveal internal structure and boundaries, distinguishing it from simple 1D transfer functions [S1, unofficial]. It builds on [2D Transfer Function with Gradient Magnitude](5491-2d-transfer-function-with-gradient-magnitude.md) by explicitly incorporating diffuse lighting calculations into the data preparation phase.

## How it works
**Inputs:**
- 3D scalar dataset $D$ of size $W \times H \times D$, where each voxel stores a byte intensity value.
- Light direction vector $\mathbf{l}$ (normalized).
- User-defined 1D transfer function $TF_{1D}(v)$ mapping voxel value to intensity.
- User-defined 2D transfer function $TF_{2D}(v, g)$ mapping voxel value and gradient magnitude to opacity.
- Ray casting parameters: image resolution, number of sub-samples per pixel, step size.

**Outputs:**
- Final 2D image with pixel values in integer range $[0, 255]$.

**Data Structures:**
1. **Original Data ($D_{raw}$):** The input 3D byte array.
2. **Gradient Data ($D_{grad}$):** A 3D byte array of the same dimensions as $D_{raw}$, storing the gradient magnitude at each voxel.
3. **Shaded Data ($D_{shaded}$):** A 3D byte array of the same dimensions as $D_{raw}$, storing the diffuse lighting intensity at each voxel.
4. **Compositing Scratchpad:** Per-pixel (or per-sub-sample) storage containing two floating-point numbers: current intensity $I_{comp}$ and current opacity $\alpha_{comp}$.

**Preprocessing Steps:**

1. **Compute Normals and Gradient Magnitude:**
   For each voxel at position $(x, y, z)$, compute the normal vector $\mathbf{n}$ by considering all 26 neighbors. The contribution of neighbors is attenuated based on their distance class:
   - 6 immediate neighbors (distance 1): weight 1.
   - 18 face/diagonal neighbors (distance $\sqrt{2}$ or $\sqrt{3}$? Note: Source says "attenuating by 18-neighbors and 26-neighbors"). The source text states: "considers all 26 neighbors, attenuating by the 18-neighbors and by the 26-neighbors." [S1, unofficial].
   - Let $\mathbf{v}_{neighbor}$ be the vector from the center voxel to the neighbor.
   - Compute the gradient vector $\mathbf{g}$ as the weighted sum of differences between the center voxel value and neighbor voxel values, weighted by the inverse distance or a specific attenuation factor defined by the source's "attenuating by" phrasing. *Note: The exact attenuation formula is not explicitly detailed in the provided text beyond the mention of 18 and 26 neighbors.* [G: Standard practice implies weighting by inverse distance or solid angle, but we follow the source's instruction to use all 26 neighbors with attenuation].
   - Calculate the gradient magnitude $g = \|\mathbf{g}\|$.
   - Store $g$ in $D_{grad}(x, y, z)$. Since $D_{grad}$ is a byte array, $g$ must be normalized or scaled to fit in $[0, 255]$.

2. **Compute Shaded Intensity:**
   For each voxel, calculate the shaded intensity $s$ using diffuse lighting:
   $$ s = v \cdot \cos(\theta) $$
   where:
   - $v$ is the raw intensity $D_{raw}(x, y, z)$.
   - $\theta$ is the angle between the light direction $\mathbf{l}$ and the computed normal $\mathbf{n}$.
   - $\cos(\theta) = \mathbf{n} \cdot \mathbf{l}$.
   - The result is normalized to the range $[0, 1]$ before being stored in $D_{shaded}(x, y, z)$ as a byte value.
   - Light intensity is assumed to be 1. [S1, unofficial]

3. **Ray Casting and Compositing:**
   For each pixel in the image:
   a. Initialize compositing scratchpad: $I_{comp} = 0$, $\alpha_{comp} = 0$.
   b. Shoot a ray through the volume.
   c. For each sub-sample along the ray at position $\mathbf{p}$:
      i. Perform trilinear interpolation on $D_{raw}$, $D_{shaded}$, and $D_{grad}$ to obtain the sample triple $(v, s, g)$.
      ii. Calculate transparency $\alpha$ and intensity $I$ for this sample:
          $$ \alpha = TF_{2D}(v, g) $$
          $$ I = TF_{1D}(v) $$
      iii. Composite this sample into the scratchpad using front-to-back compositing:
          $$ \alpha_{new} = \alpha_{comp} + (1 - \alpha_{comp}) \cdot \alpha $$
          $$ I_{new} = I_{comp} + (1 - \alpha_{comp}) \cdot \alpha \cdot I $$
          $$ I_{comp} = I_{new} $$
          $$ \alpha_{comp} = \alpha_{new} $$
   d. After all samples for the pixel are processed, the final floating-point intensity $I_{comp}$ is averaged with other sub-samples (if multi-sampling is used for anti-aliasing) and transformed to the integer range $[0, 255]$. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_normal_and_gradient(data, x, y, z, width, height, depth):
    """
    Computes the normal vector and gradient magnitude for a voxel.
    Considers all 26 neighbors with attenuation.
    """
    center_val = data[x, y, z]
    grad_vec = [0.0, 0.0, 0.0]
    
    # Iterate over all 26 neighbors
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                
                nx = (x + dx) % width
                ny = (y + dy) % height
                nz = (z + dz) % depth
                
                neighbor_val = data[nx, ny, nz]
                diff = neighbor_val - center_val
                
                # Vector from center to neighbor
                vec = [dx, dy, dz]
                dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                
                # Attenuation: Source mentions attenuating by 18 and 26 neighbors.
                # Assuming inverse distance weighting as a standard interpretation of "attenuating"
                # when specific weights aren't given. [G]
                weight = 1.0 / dist
                
                grad_vec[0] += diff * vec[0] * weight
                grad_vec[1] += diff * vec[1] * weight
                grad_vec[2] += diff * vec[2] * weight
                
    magnitude = math.sqrt(grad_vec[0]**2 + grad_vec[1]**2 + grad_vec[2]**2)
    return grad_vec, magnitude

def compute_shaded_intensity(normal, light_dir, raw_intensity):
    """
    Computes diffuse shading intensity.
    """
    # Normalize normal
    n_len = math.sqrt(normal[0]**2 + normal[1]**2 + normal[2]**2)
    if n_len == 0:
        return 0.0
    normal = [n[0]/n_len for n in normal]
    
    # Cosine of angle
    cos_theta = normal[0]*light_dir[0] + normal[1]*light_dir[1] + normal[2]*light_dir[2]
    
    # Clamp to [0, 1] for diffuse lighting (back-facing surfaces get 0)
    cos_theta = max(0.0, min(1.0, cos_theta))
    
    # Intensity = raw * cos(theta)
    shaded = raw_intensity * cos_theta
    
    # Normalize to [0, 1] assuming raw_intensity is in [0, 255] or similar
    # The source says "normalized to the range [0,1]". 
    # If raw_intensity is byte [0,255], we divide by 255.
    shaded_normalized = shaded / 255.0
    return shaded_normalized

def trilinear_interpolate(data, x, y, z, width, height, depth):
    """
    Performs trilinear interpolation on a 3D byte array.
    """
    # Implementation details for trilinear interpolation
    # ... (standard implementation)
    pass

def render_volume(data, light_dir, tf_1d, tf_2d, width, height, depth, ray_steps, sub_samples):
    """
    Main rendering loop.
    """
    # Precompute gradients and shaded data
    grad_data = [[0]*depth for _ in range(height)]
    shaded_data = [[0]*depth for _ in range(height)]
    
    for x in range(width):
        for y in range(height):
            for z in range(depth):
                normal, grad_mag = compute_normal_and_gradient(data, x, y, z, width, height, depth)
                grad_data[x][y][z] = grad_mag # Store normalized/byte-scaled
                shaded_data[x][y][z] = compute_shaded_intensity(normal, light_dir, data[x][y][z])
                
    # Render
    image = [[0]*width for _ in range(height)]
    
    for px in range(width):
        for py in range(height):
            total_intensity = 0.0
            for s in range(sub_samples):
                # Generate ray origin and direction for sub-sample
                ray_origin, ray_dir = generate_ray(px, py, s, width, height)
                
                comp_intensity = 0.0
                comp_opacity = 0.0
                
                for step in range(ray_steps):
                    # Interpolate along ray
                    t = step * step_size
                    pos = ray_origin + ray_dir * t
                    
                    v = trilinear_interpolate(data, pos.x, pos.y, pos.z, width, height, depth)
                    s_val = trilinear_interpolate(shaded_data, pos.x, pos.y, pos.z, width, height, depth)
                    g = trilinear_interpolate(grad_data, pos.x, pos.y, pos.z, width, height, depth)
                    
                    # Apply Transfer Functions
                    alpha = tf_2d(v, g)
                    intensity = tf_1d(v)
                    
                    # Front-to-back compositing
                    alpha_new = comp_opacity + (1 - comp_opacity) * alpha
                    intensity_new = comp_intensity + (1 - comp_opacity) * alpha * intensity
                    
                    comp_intensity = intensity_new
                    comp_opacity = alpha_new
                
                total_intensity += comp_intensity
            
            # Average sub-samples and convert to byte
            avg_intensity = total_intensity / sub_samples
            image[py][px] = int(max(0, min(255, avg_intensity * 255)))
            
    return image
```

## Operating characteristics
- **Memory:** Requires two additional memory banks of the same size as the original dataset for storing the shaded data and gradient information [S1, unofficial]. This can be avoided by computing normals and shading on-the-fly, which also allows for view-dependent effects like specular lighting [S1, unofficial].
- **Performance:** In the cited example, shading took 5 seconds, gradient calculation took 5 seconds, and rendering (35 seconds for all images at 512x341 resolution with one ray per pixel) took 35 seconds [S1, unofficial]. Gas rendering images at 512x341 resolution took 83 seconds each [S1, unofficial].
- **Strengths:** Provides better visualization of 3D flow and internal structures compared to non-shaded methods [S1, unofficial]. Allows interactive definition of transfer functions.
- **Limitations:** Precomputed shading is view-dependent and static; changing the view requires re-rendering or on-the-fly computation. The attenuation method for gradient calculation is not fully specified in the source, potentially leading to implementation ambiguity [S1, unofficial].

## Background
This method was developed for Computational Fluid Dynamics (CFD) visualization, specifically for visualizing gas flow fields. It was presented by David Ebert et al. in the paper "Volume Rendering Methods for Computational Fluid Dynamics Visualization" [S1, unofficial]. The technique was used to visualize static pressure generated using computed Navier-Stokes equations.

## Sources
[S1] Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
