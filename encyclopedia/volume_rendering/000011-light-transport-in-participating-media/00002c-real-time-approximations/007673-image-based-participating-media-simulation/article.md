# Image-based Participating Media Simulation

Also called: Image-based fog simulation, Image-based participating media

## Overview
This method simulates perceptually plausible participating media effects (such as fog or smoke) in a single, clean high dynamic range (HDR) image without requiring 3D scene geometry or physical light transport simulation [S1, unofficial]. It addresses the underconstrained nature of the problem—where depth and reflectance are unknown—by leveraging human visual system assumptions and minimal user input for depth estimation [S1, unofficial]. Unlike methods that require multiple images or polarization filters [S1, unofficial], this approach operates on a single image using image processing techniques, making it suitable for interactive image editing tools [S1, unofficial].

## How it works
The method transforms the input image into a perceptual representation, applies depth-dependent radiance transfer, and simulates scattering blur.

### Inputs and Outputs
- **Input:** A single HDR image $I_0$ in RGB space.
- **User Inputs:**
  - Per-pixel depth $\Delta t$ (estimated via segmentation, perspective gradients, or Shape from Shading).
  - In-scattering constant $InScat$ (user-defined).
  - Extinction coefficient $k$ (assumed constant).
  - In-scattering coefficient $\sigma$ (assumed constant).
  - Optional: Perlin noise parameters for non-homogeneous media.
- **Output:** An HDR image $L$ with participating media effects applied.

### Data Structures
- **Image Buffer:** Stores pixel values in HLS (Hue, Luminance, Saturation) space.
- **Depth Map:** A 2D array of floating-point values $\Delta t(x, y)$ representing the estimated distance along the viewing ray for each pixel.
- **Histogram:** Used for highlight detection and shadow definition.

### Steps

1.  **Color Space Conversion:**
    Transform the input image $I_0$ from RGB to HLS color space. Let $L_0(x,y)$, $H(x,y)$, and $S(x,y)$ be the Luminance, Hue, and Saturation channels respectively [S1, unofficial].

2.  **Highlight Detection and Attenuation:**
    - Compute the histogram of the Luminance channel $L_0$.
    - Find the minimum in the derivative of the histogram to identify the start of highlights [S1, unofficial].
    - For pixels identified as highlights, attenuate the luminance $L_0$ to recover original colors in high dynamic range scenarios [S1, unofficial].

3.  **Relighting Simulation (Perceptual Filters):**
    Simulate the effect of the medium on the original image using histogram manipulation [S1, unofficial]:
    - **Desaturation:** Reduce saturation $S$ in proportion to $\sigma$ and $k$.
    - **Contrast Reduction:** Reduce contrast globally.
    - **Shadow Brightening:** Increase luminance in shadowed areas (defined as pixels below five f-stops in the histogram).
    - **Highlight Attenuation:** Apply further attenuation to highlights if not fully processed in Step 2.

4.  **Extinction and Airlight Application:**
    Apply the simplified Radiance Transfer Equation integration for each pixel using the estimated depth $\Delta t$ [S1, unofficial]:
    $$ L(x_s, y_s) = \sigma \cdot InScat \cdot \Delta t + L_{processed}(x_s, y_s) \cdot e^{-k \cdot \Delta t} $$
    Where:
    - $L(x_s, y_s)$ is the final radiance.
    - $L_{processed}(x_s, y_s)$ is the luminance after Step 3.
    - $\Delta t$ is the per-pixel depth.
    - $InScat$ is the user-defined constant in-scattering value.
    - $\sigma$ is the in-scattering coefficient.
    - $k$ is the extinction coefficient.

5.  **Blur and Detail Loss:**
    Simulate multiple scattering blur and detail loss by applying a Point Spread Function (PSF) to the resulting image [S1, unofficial].

### Depth Estimation Methods
The method requires per-pixel depth $\Delta t$. Three approaches are supported [S1, unofficial]:
1.  **Depth Simplification (Discrete):** User segments the image into up to three planes (close-up, middle, background).
2.  **Depth Simplification (Smooth):** User drags the mouse to define a perspective plane for receding surfaces (e.g., ground), generating a smooth depth gradient.
3.  **Shape from Shading (SFS):** If manual segmentation is impractical, use the assumption that brighter pixels are closer to the camera and darker pixels are farther. This leverages human visual system assumptions [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_participating_media(input_image_rgb, depth_map, params):
    """
    Simulates participating media in a single HDR image.
    
    Args:
        input_image_rgb: 2D array of shape (H, W, 3) in RGB space.
        depth_map: 2D array of shape (H, W) with per-pixel depth delta_t.
        params: Dictionary containing:
            - InScat: float, user-defined in-scattering constant.
            - sigma: float, in-scattering coefficient.
            - k: float, extinction coefficient.
            - use_sfs: bool, whether to use Shape from Shading if depth_map is None.
            - perlin_params: dict, optional parameters for non-homogeneous noise.
            
    Returns:
        output_image: 2D array of shape (H, W, 3) in RGB space.
    """
    # Step 1: Color Space Conversion
    # Convert RGB to HLS. Implementation depends on library, assumed available.
    image_hls = rgb_to_hls(input_image_rgb)
    L = image_hls[:,:,1] # Luminance
    H = image_hls[:,:,0] # Hue
    S = image_hls[:,:,2] # Saturation
    
    # Step 2: Highlight Detection and Attenuation
    # Compute histogram of Luminance
    hist, bin_edges = compute_histogram(L)
    # Find minimum in derivative of histogram to identify highlights
    hist_deriv = np.diff(hist)
    highlight_threshold = bin_edges[np.argmin(hist_deriv)]
    
    # Attenuate luminance for highlights
    highlight_mask = L > highlight_threshold
    L[highlight_mask] = attenuate_highlights(L[highlight_mask])
    
    # Step 3: Relighting Simulation (Perceptual Filters)
    # Shadow Brightening: Define shadows as pixels below 5 f-stops
    # Assuming linear light, 5 f-stops is a factor of 2^5 = 32 in intensity? 
    # Or simply a threshold in the histogram. We use a fixed threshold for simplicity.
    shadow_threshold = compute_fstop_threshold(L, stops=5)
    shadow_mask = L < shadow_threshold
    L[shadow_mask] = brighten_shadows(L[shadow_mask])
    
    # Desaturation and Contrast Reduction
    # Modify S and L based on sigma and k
    S = S * (1.0 - params['sigma'] * 0.5) # Example desaturation
    L = reduce_contrast(L)
    
    # Step 4: Extinction and Airlight Application
    # Formula: L_final = sigma * InScat * delta_t + L_processed * exp(-k * delta_t)
    InScat = params['InScat']
    sigma = params['sigma']
    k = params['k']
    
    airlight_term = sigma * InScat * depth_map
    extinction_term = L * np.exp(-k * depth_map)
    
    L_final = airlight_term + extinction_term
    
    # Step 5: Blur and Detail Loss
    # Apply Point Spread Function (PSF)
    # Assuming a Gaussian blur for simplicity
    psf = create_gaussian_psf(kernel_size=15, sigma=2.0)
    L_final = convolve_2d(L_final, psf)
    
    # Reconstruct RGB
    image_hls[:,:,1] = L_final
    output_image_rgb = hls_to_rgb(image_hls)
    
    return output_image_rgb

def attenuate_highlights(highlights):
    # Recover original colors by attenuating high luminance values
    # Implementation detail: scale down values above a certain threshold
    return highlights / 2.0 # Placeholder for specific attenuation logic

def brighten_shadows(shadows):
    # Increase luminance in shadowed areas
    return shadows * 1.5 # Placeholder for specific brightening logic

def reduce_contrast(luminance):
    # Reduce contrast globally
    mean_val = np.mean(luminance)
    return mean_val + 0.8 * (luminance - mean_val)

def compute_fstop_threshold(luminance, stops):
    # Compute threshold corresponding to 'stops' below the mean or max
    # Simplified: assume mean is reference
    return np.mean(luminance) / (2**stops)

def create_gaussian_psf(kernel_size, sigma):
    # Create a 2D Gaussian kernel
    ax = np.arange(-kernel_size//2 + 1., kernel_size//2 + 1.)
    xx, yy = np.meshgrid(ax, ax)
    kernel = np.exp(-(xx**2 + yy**2) / (2. * sigma**2))
    return kernel / np.sum(kernel)

def convolve_2d(image, kernel):
    # 2D convolution
    return scipy.signal.convolve2d(image, kernel, mode='same')
```

### Parameters and Typical Values
- **$InScat$**: User-defined constant representing integrated in-scattering. No specific typical value given; depends on desired fog density [S1, unofficial].
- **$\sigma$**: In-scattering coefficient. Assumed constant for homogeneous media [S1, unofficial].
- **$k$**: Extinction coefficient. Assumed constant [S1, unofficial].
- **$\Delta t$**: Per-pixel depth. Values depend on scene scale and user estimation [S1, unofficial].
- **Perlin Noise Parameters**: Octaves and period, used to shape noise for non-homogeneous media [S1, unofficial].

### Edge Cases and Pitfalls
- **Underconstrained Problem:** Physical accuracy is impossible due to lack of depth and reflectance data. The method targets perceptual plausibility, relying on the fact that human visual systems do not detect all physical inaccuracies [S1, unofficial].
- **Non-homogeneous Media:** The base method assumes homogeneity. Non-homogeneous media are simulated by adding Perlin noise to the algorithm [S1, unofficial].
- **Shape from Shading Assumptions:** The SFS fallback assumes brighter pixels are closer. This is not universally true but leverages common human visual assumptions [S1, unofficial].

## Subtopics

### [Multiple Scattering Approximation via Light Volume Lookup](1188-multiple-scattering-approximation-via-light-volume-lookup.md)
Uses light volumes to approximate multiple scattering, requiring 3D scene data. Choose when physical accuracy is needed and geometry is available.

### [Spherical Harmonic Volumetric Visibility](1225-spherical-harmonic-volumetric-visibility.md)
Compresses directional visibility using Spherical Harmonics. Choose for real-time global illumination in complex scenes.

### [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md)
Discretizes volume into froxels for efficient ray marching. Choose for high-quality real-time volumetric rendering with geometry.

### [Hardware Ray Tracing Global Illumination](3648-hardware-ray-tracing-global-illumination.md)
Uses hardware ray tracing for accurate GI. Choose for offline or high-end real-time rendering with ray-tracing hardware.

### [Volumetric Light Scattering via Froxel Clustering](4460-volumetric-light-scattering-via-froxel-clustering.md)
Clusters froxels for efficient light scattering. Choose for large-scale volumetric effects.

### [Voxel Cone Tracing](4558-voxel-cone-tracing.md)
Traces cones through voxelized scene for GI. Choose for real-time GI in voxelized environments.

### [Deferred Voxel Shading](4685-deferred-voxel-shading.md)
Combines deferred shading with voxelization for GI. Choose for deferred rendering pipelines.

### [Light Beam Tracing](10717-light-beam-tracing.md)
Traces light beams for accurate scattering. Choose for offline rendering with high accuracy.

### [Precomputed Radiance Transfer](13182-precomputed-radiance-transfer.md)
Precomputes light transport for static scenes. Choose for static scenes with complex lighting.

### [Flax Volumetric Fog](13880-flax-volumetric-fog.md)
Engine-specific volumetric fog solution. Choose when using the Flax engine.

### [Voxel Number Map with LUT for Translucent Rendering](14179-voxel-number-map-with-lut-for-translucent-rendering.md)
Uses voxel number maps and LUTs for translucency. Choose for translucent materials.

### [Cinematic Quality Volumetric Clouds](14321-cinematic-quality-volumetric-clouds.md)
Specialized for cinematic clouds. Choose for cloud rendering.

### [Voxelisation based model for global illumination](19817-voxelisation-based-model-for-global-illumination.md)
General voxel-based GI approach. Choose for voxel-based GI.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels $O(N)$ for the main radiance transfer step, plus overhead for histogram manipulation and convolution.
- **Memory Complexity:** $O(N)$ for storing the image and depth map.
- **Measured Performance:** Designed for short computational times and easy interactivity, suitable for image editing tools like Photoshop [S1, unofficial].
- **Strengths:** No 3D scene knowledge required; works on single HDR images; perceptually plausible results; fast.
- **Limitations:** Physically inaccurate; relies on user input for depth; assumes homogeneous media (unless noise is added); limited to isotropic scattering.
- **When to Use:** Image editing, post-processing, or when 3D data is unavailable.
- **When Not to Use:** When physical accuracy is required, or when dynamic lighting and geometry are present.

## Background
The method was proposed by researchers in Spain, presented at the Computer Graphics International Conference (CEIG) in 2008 [S1, unofficial]. It builds on the work of Debevec et al. for image-based lighting [Deb98] and Ramanarayanan et al. on perceptual plausibility [RFWB07]. It differs from Nayar's haze removal methods which require multiple images or polarization filters [S1, unofficial]. The method is designed to extend image editing tools, providing easy interactivity and short computational times [S1, unofficial].

## Sources
[S1] Image-based Participating Media, https://doi.org/10.2312/localchapterevents/ceig/ceig08/179-187, fetched 2026-10-26 (unofficial)
