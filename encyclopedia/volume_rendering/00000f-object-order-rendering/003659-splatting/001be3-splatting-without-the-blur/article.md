# Splatting Without the Blur

Also called: Gradient splats, Gradient splatting, Post-Classified Splatting, Post-projection classification splatting, Post-shaded splatting

## Overview
Splatting Without the Blur is a volume rendering technique that performs classification and shading *after* voxel densities have been projected onto the screen, thereby avoiding the edge blurring inherent in traditional pre-shaded splatting [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. Unlike [Pre-Classified Splatting](5410-pre-classified-splatting.md), which classifies voxels before projection and results in fuzzy edges due to the convolution of the reconstruction kernel with the classification function, this method projects raw density and computes gradients on the image plane to yield crisp, well-preserved surface details [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. It differs from [Edge Preservation in Volume Rendering Using Splatting](6573-edge-preservation-in-volume-rendering-using-splatting.md) by using standard radial splats for density and either gradient splats or screen-space differencing for shading, rather than non-linear warping of footprints [S1, unofficial][S2, abstract only, unofficial][S3, unofficial].

## How it works
The method modifies the standard splatting pipeline by decoupling projection from classification. It requires an image-aligned sheet buffer for compositing, as axis-aligned buffers cannot support the per-pixel shading logic required for post-classification [S3, unofficial].

### Inputs and Outputs
*   **Input:** A 3D volume of raw scalar density values $\rho(x,y,z)$.
*   **Output:** A 2D image with crisp edges, where pixels below a density threshold are discarded.

### Data Structures
1.  **Density Volume:** A 3D grid of scalar values $\rho$.
2.  **Gradient Volume (Optional):** If using gradient splats, a 3D grid of vector values $\nabla \rho$. If using central differencing, this is not pre-computed.
3.  **Image-Aligned Sheet Buffer:** A 2D buffer aligned with the screen coordinates, storing the accumulated density and potentially intermediate shading data for the current sheet.
4.  **Accumulation Sheet:** The working image buffer where splats are composited.

### Parameters
*   **Iso-threshold ($\tau$):** The density value below which pixels are classified as background (set to zero) [S2, abstract only, unofficial].
*   **Splat Kernel:** Radially symmetric interpolation kernel (e.g., Gaussian) for projecting density [S1, unofficial][S2, abstract only, unofficial].
*   **Gradient Splat Kernel:** Radially symmetric kernel for projecting gradient vectors [S1, unofficial][S2, abstract only, unofficial].

### Steps

1.  **Project Density:**
    For each voxel in the current sheet, project its raw density $\rho$ onto the image-aligned sheet buffer using a regular, radially symmetric splat [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. This creates a blurred density field on the screen.

2.  **Compute Screen Gradients:**
    There are two alternative methods to obtain the gradient information required for shading:
    *   **Method A: Gradient Splats [S1, unofficial][S2, abstract only, unofficial]:** Project the pre-computed gradient volume $\nabla \rho$ onto the screen using "gradient splats." Each splat contributes a vector value to the screen buffer.
    *   **Method B: Central Differencing [S1, unofficial][S2, abstract only, unofficial]:** Compute the gradient directly on the projection plane from the splatted density buffer. For a pixel at $(u,v)$, the gradient is estimated using central differences on the accumulated density values $D(u,v)$:
        $$ \nabla D(u,v) \approx \left( \frac{D(u+1,v) - D(u-1,v)}{2}, \frac{D(u,v+1) - D(u,v-1)}{2} \right) $$
        This method reduces the number of footprint rasterizations by a factor of four compared to Method A, as only the scalar density volume needs to be projected [S1, unofficial][S2, abstract only, unofficial].

3.  **Classify and Shade:**
    Perform per-pixel shading on the image plane using the computed screen gradients [S1, unofficial][S2, abstract only, unofficial][S3, unofficial].
    *   Calculate the normalized gradient direction $\mathbf{n} = \frac{\nabla D}{\|\nabla D\|}$ [S1, unofficial][S2, abstract only, unofficial].
    *   Apply a lighting model (e.g., Phong) using $\mathbf{n}$ to determine the color and opacity of the pixel.
    *   **Classification:** If the accumulated density at the pixel is below the iso-threshold $\tau$, set the pixel color and opacity to zero (background) [S2, abstract only, unofficial][S3, unofficial]. This step ensures that volume contributions outside the iso-range do not affect the final image, preventing the "fuzzy" appearance of pre-shaded methods [S1, unofficial][S2, abstract only, unofficial].

4.  **Composite:**
    Composite the shaded sheet buffer into the final accumulation sheet [S3, unofficial]. Advance the sheet buffer to the next slice [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatting_without_blur(volume, iso_threshold, image_width, image_height):
    """
    Renders a volume using post-shaded splatting.
    
    Args:
        volume: 3D array of density values rho(x,y,z)
        iso_threshold: Scalar density threshold for classification
        image_width: Width of the output image
        image_height: Height of the output image
        
    Returns:
        final_image: 2D array of RGB values
    """
    # Initialize accumulation sheet with zeros
    accumulation_sheet = zeros((image_height, image_width, 3))
    
    # Determine sheet orientation (e.g., x-y planes)
    # Iterate through sheets from front to back
    for sheet_index in range(volume.depth):
        # 1. Project Density
        # Create a temporary sheet buffer for density accumulation
        density_sheet = zeros((image_height, image_width))
        
        for x in range(volume.width):
            for y in range(volume.height):
                rho = volume[x, y, sheet_index]
                if rho == 0:
                    continue
                
                # Project voxel density using regular splat
                # This adds a blurred contribution to density_sheet
                project_splat(density_sheet, x, y, sheet_index, rho)
        
        # 2. Compute Screen Gradients
        # Option A: Gradient Splats (requires pre-computed gradient volume)
        # Option B: Central Differencing (uses density_sheet)
        # We use Option B as it is more efficient (factor of 4 fewer rasterizations)
        gradient_sheet_x = zeros((image_height, image_width))
        gradient_sheet_y = zeros((image_height, image_width))
        
        for u in range(1, image_height - 1):
            for v in range(1, image_width - 1):
                # Central difference on the splatted density
                gradient_sheet_x[u, v] = (density_sheet[u+1, v] - density_sheet[u-1, v]) / 2.0
                gradient_sheet_y[u, v] = (density_sheet[u, v+1] - density_sheet[u, v-1]) / 2.0
        
        # 3. Classify and Shade
        # Create a temporary sheet buffer for the final color of this sheet
        sheet_color = zeros((image_height, image_width, 3))
        
        for u in range(image_height):
            for v in range(image_width):
                # Check if density is above threshold
                if density_sheet[u, v] < iso_threshold:
                    continue # Discard pixel
                
                # Compute gradient magnitude and normal
                gx = gradient_sheet_x[u, v]
                gy = gradient_sheet_y[u, v]
                mag = sqrt(gx*gx + gy*gy)
                
                if mag < 1e-6:
                    # Flat region, use default color or ambient lighting
                    color = ambient_color
                else:
                    # Normalize gradient for lighting
                    nx = gx / mag
                    ny = gy / mag
                    # Apply lighting model (simplified)
                    color = lighting_model(nx, ny, rho=density_sheet[u, v])
                
                sheet_color[u, v] = color
        
        # 4. Composite
        # Composite sheet_color into accumulation_sheet
        # Using standard alpha blending or additive blending depending on optical model
        composite_sheet(accumulation_sheet, sheet_color)
        
    return accumulation_sheet

def project_splat(sheet, voxel_x, voxel_y, sheet_z, density):
    """
    Projects a voxel's density onto the sheet using a radially symmetric kernel.
    Implementation detail: Maps voxel (voxel_x, voxel_y, sheet_z) to screen coordinates (u, v).
    Adds kernel-weighted density to sheet[u, v] for all pixels in the splat footprint.
    """
    # Transform voxel to screen space (u, v)
    u, v = transform_to_screen(voxel_x, voxel_y, sheet_z)
    
    # Define footprint radius (e.g., 2.0 sigma)
    radius = 2.0
    
    for dy in range(-radius, radius + 1):
        for dx in range(-radius, radius + 1):
            u_splat = u + dx
            v_splat = v + dy
            
            if 0 <= u_splat < sheet.shape[0] and 0 <= v_splat < sheet.shape[1]:
                # Calculate kernel weight
                dist_sq = dx*dx + dy*dy
                weight = exp(-dist_sq / (2.0 * sigma*sigma)) # Assuming Gaussian kernel
                sheet[u_splat, v_splat] += weight * density
```

### Edge Cases and Pitfalls
*   **Pre-shaded + Post-classification Artifacts:** Combining pre-shaded voxel projection with post-classification (setting pixels to zero if density is below iso-threshold) causes severe artifacts [S2, abstract only, unofficial]. The method must use raw density projection for the classification step, not pre-shaded color/opacity [S2, abstract only, unofficial].
*   **Sheet Buffer Alignment:** Post-classified rendering requires image-aligned sheet buffers; it cannot be done with axis-aligned sheet buffers because the shading logic depends on screen-space gradients [S3, unofficial].
*   **Gradient Noise:** Central differencing on the splatted density can be sensitive to noise in the volume or aliasing in the splat projection. Lookup tables are recommended for shading to handle the overhead and potential discontinuities [S1, unofficial][S2, abstract only, unofficial].

## Subtopics

### [Pre-Classified Splatting](5410-pre-classified-splatting.md)
A rendering strategy where classification and shading are applied to the raw density volume before splatting, resulting in a blurred edge image upon reconstruction.

### [Edge Preservation in Volume Rendering Using Splatting](6573-edge-preservation-in-volume-rendering-using-splatting.md)
A method to preserve sharp edge details in splatting for volume rendering by applying a non-linear warping to the footprints of conventional splats and building a table of footprints for different possible edge positions and edge strengths.

## Operating characteristics
*   **Time Complexity:** The central differencing scheme reduces the number of footprint rasterizations by a factor of four compared to using gradient splats, as only voxel densities need to be projected [S2, abstract only, unofficial]. However, it adds the overhead of per-pixel gradient calculation and shading [S1, unofficial][S2, abstract only, unofficial].
*   **Memory Complexity:** Requires storing the density sheet buffer and, if using central differencing, temporary gradient buffers. Image-aligned buffers may require more complex memory management than axis-aligned buffers [S3, unofficial].
*   **Performance:** Rendered at 512 x 512 resolution for the full UNC head dataset [S2, abstract only, unofficial]. The method provides significant visual improvement in edge crispness, especially at high magnifications, at the cost of increased per-pixel computation [S1, unofficial][S2, abstract only, unofficial].
*   **Strengths:** Produces crisp edges and well-preserved surface detail; prevents volume contributions outside the iso-range from affecting the image [S1, unofficial][S2, abstract only, unofficial].
*   **Limitations:** Requires image-aligned sheet buffers; central differencing may introduce artifacts if the splatted density is noisy; lookup tables are needed for efficient shading [S1, unofficial][S2, abstract only, unofficial][S3, unofficial].

## Background
The method was introduced by Frank Mueller in 1999 as a response to the blurring issues inherent in traditional pre-shaded splatting [S1, unofficial][S2, abstract only, unofficial][S3, unofficial]. It manipulates the rendering pipeline by performing classification and shading after projection, leveraging the fact that shading requires gradients, which can be computed either via gradient splats or screen-space central differencing [S1, unofficial][S2, abstract only, unofficial]. The technique was presented at the 1999 Visualization conference and demonstrated on datasets such as the UNC head and ganglion nerve [S2, abstract only, unofficial].

## Sources
[S1] (PDF) EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
[S2] Splatting Without The Blur - Stony Brook University, https://www3.cs.stonybrook.edu/~mueller/research/blur/, fetched 2026-10-28 (abstract only, unofficial)
[S3] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
