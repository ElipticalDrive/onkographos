# MIP map filtering for multiple scattering

Also called: Elek et al. MIP map filtering, Filtering-based real-time multiple scattering, MIP map Gaussian blurring, Real-time GPU filtering method

## Overview
This method efficiently approximates multiple-scattering light transport in non-homogeneous, analytically integrable participating media by applying Gaussian blurring via MIP maps [S1, unofficial]. It solves the high computational cost of neighborhood filtering in screen-space rendering, enabling real-time performance on GPUs by reducing time complexity from $O(n^4)$ to $O(n^2 \log_2 n)$ [S1, unofficial]. Unlike naive gathering which iterates over a fixed neighborhood for every pixel, this approach leverages hierarchical image representations to accumulate contributions across scales [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A screen-space image representing single-scattering or vacuum-rendered lighting, where each pixel is treated as an independent light source [S1, unofficial].
*   **Input:** A density function $\varrho(x)$ that is analytically integrable, allowing for non-homogeneous media (e.g., exponential density variations) [S1, unofficial].
*   **Output:** A blurred image representing the multiple-scattering contribution [S1, unofficial].

**Data Structures:**
*   **MIP Map Pyramid:** A set of images $L_k$ where $L_0$ has resolution $n \times n$. Higher levels $L_k$ have resolutions decreasing exponentially (typically halving in each dimension) until $1 \times 1$ [S1, unofficial].
*   **Kernel:** A constant-sized Gaussian kernel $G$ used for convolution at each level [S1, unofficial].

**Parameters:**
*   **$n$:** Image resolution (width or height).
*   **$m$:** Neighborhood radius. In the context of HDR images where a pixel blurs into the whole image, $m = n$ [S1, unofficial].
*   **Target Performance:** > 25 FPS at 1280x720 resolution [S1, unofficial].

**Steps:**
1.  **Generate MIP Map Pyramid:** Construct the MIP map levels $L_0$ to $L_{max}$ from the input image. The total number of levels is $O(\log_2 n)$ [S1, unofficial].
2.  **Iterate Levels:** For each MIP map level $k > 0$:
    a.  Perform convolution of the current level's image with the constant-sized Gaussian kernel $G$ [S1, unofficial].
    b.  For each pixel in the level, iterate through the constant-sized neighborhood defined by the kernel [S1, unofficial].
    c.  Accumulate weighted contributions from the neighborhood pixels [S1, unofficial].
3.  **Combine Results:** Aggregate the blurred contributions from all MIP map levels to form the final multiple-scattering approximation [S1, unofficial].
4.  **Handle Non-Homogeneous Media:** Use analytically integrable density functions (e.g., derived from the barometric formula) to calculate transmittance efficiently between points [S1, unofficial].

**Formulas:**
*   **Barometric Formula for Pressure:** Used to derive density functions for non-homogeneous media [S1, unofficial]:
    $$p(z) = p(0) \cdot \exp\left(-\frac{mgz}{kT}\right)$$
    Where:
    *   $p(z)$ is pressure at altitude $z$.
    *   $p(0)$ is pressure at $z=0$.
    *   $m$ is molecular mass.
    *   $g$ is gravitational constant.
    *   $k$ is Boltzmann constant.
    *   $T$ is temperature.
    *   Density $\varrho(z)$ is derived similarly, as it is directly related to pressure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mip_map_filtering(input_image, kernel_size, max_levels):
    """
    Applies MIP map filtering for multiple scattering.
    
    Args:
        input_image: 2D array of resolution n x n.
        kernel_size: Size of the Gaussian kernel G.
        max_levels: Number of MIP map levels (O(log2 n)).
        
    Returns:
        blurred_image: Resulting multiple-scattering approximation.
    """
    n = input_image.shape[0]
    mip_levels = [input_image]
    
    # Step 1: Generate MIP Map Pyramid
    current_image = input_image
    for k in range(1, max_levels):
        # Downsample current image to create next level
        # Resolution decreases exponentially
        next_resolution = current_image.shape[0] // 2
        if next_resolution < 1:
            break
        # Simple averaging downsampling for MIP map generation
        downsampled = np.zeros((next_resolution, next_resolution))
        for i in range(0, current_image.shape[0], 2):
            for j in range(0, current_image.shape[1], 2):
                # Average 2x2 block
                block = current_image[i:i+2, j:j+2]
                downsampled[i//2, j//2] = np.mean(block)
        mip_levels.append(downsampled)
        current_image = downsampled

    blurred_image = np.zeros_like(input_image)
    
    # Step 2: Iterate Levels and Convolve
    # Note: The source describes convolving each level k > 0 with a constant kernel G
    # and accumulating contributions. The exact combination strategy (e.g., summing 
    # blurred levels or using them as weights) is not fully specified in the provided notes.
    # We assume a summation of blurred contributions from all levels for the approximation.
    
    for k in range(1, len(mip_levels)):
        level_image = mip_levels[k]
        level_res = level_image.shape[0]
        
        # Create a blurred version of this level using the constant kernel G
        blurred_level = np.zeros_like(level_image)
        half_kernel = kernel_size // 2
        
        for i in range(level_res):
            for j in range(level_res):
                # Iterate through constant-sized neighborhood
                contribution = 0.0
                for di in range(-half_kernel, half_kernel + 1):
                    for dj in range(-half_kernel, half_kernel + 1):
                        ni = i + di
                        nj = j + dj
                        # Handle boundaries (clamp or wrap, not specified)
                        if 0 <= ni < level_res and 0 <= nj < level_res:
                            # Apply Gaussian weight G(di, dj)
                            weight = gaussian_weight(di, dj, kernel_size)
                            contribution += level_image[ni, nj] * weight
                blurred_level[i, j] = contribution
        
        # Upsample blurred level back to original resolution for accumulation
        # [G] Standard bilinear upsampling is assumed as not specified
        upscaled_blur = upscale_to_original_resolution(blurred_level, n)
        
        # Accumulate contribution
        blurred_image += upscaled_blur

    return blurred_image

def gaussian_weight(dx, dy, kernel_size):
    """
    Computes Gaussian weight for a given offset.
    [G] Standard 2D Gaussian kernel assumed.
    """
    sigma = kernel_size / 6.0  # Typical sigma for kernel size
    exponent = -((dx**2 + dy**2) / (2 * sigma**2))
    return np.exp(exponent)

def upscale_to_original_resolution(blurred_level, target_n):
    """
    [G] Bilinear upsampling to match original resolution n x n.
    """
    h, w = blurred_level.shape
    scale_h = target_n / h
    scale_w = target_n / w
    upscaled = np.zeros((target_n, target_n))
    for i in range(target_n):
        for j in range(target_n):
            src_i = i / scale_h
            src_j = j / scale_w
            # Bilinear interpolation
            i0, i1 = int(np.floor(src_i)), int(np.ceil(src_i))
            j0, j1 = int(np.floor(src_j)), int(np.ceil(src_j))
            i0, i1 = np.clip(i0, 0, h-1), np.clip(i1, 0, h-1)
            j0, j1 = np.clip(j0, 0, w-1), np.clip(j1, 0, w-1)
            fi, fj = src_i - i0, src_j - j0
            val = (1-fi)*(1-fj)*blurred_level[i0, j0] + \
                  fi*(1-fj)*blurred_level[i1, j0] + \
                  (1-fi)*fj*blurred_level[i0, j1] + \
                  fi*fj*blurred_level[i1, j1]
            upscaled[i, j] = val
    return upscaled
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The pseudocode assumes clamping or wrapping for neighborhood iteration; the source does not specify. [G]
*   **Non-Analytic Densities:** The method requires analytically integrable density functions. Arbitrary density functions cannot be handled efficiently. [S1, unofficial]
*   **Artifacts:** While MIP map filtering reduces artifacts compared to naive gathering, some visual artifacts may still be present, though significantly reduced. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(n^2 \log_2 n)$, where $n$ is the image resolution. This is a significant improvement over naive gathering's $O(n^4)$ to $O(n^6)$ for HDR images where $m=n$ [S1, unofficial].
*   **Memory Complexity:** $O(n^2)$ for storing the MIP map pyramid, as the sum of geometric series $n^2 + (n/2)^2 + ...$ converges to $O(n^2)$ [S1, unofficial].
*   **Measured Performance:** On a GeForce GTX 760M, the method achieved 2 milliseconds per frame for a sample image, compared to 30 seconds for the gathering algorithm and 15 hours for path-tracing reference [S1, unofficial].
*   **Target Performance:** Higher than 25 FPS at 1280x720 resolution (HD) [S1, unofficial].
*   **Strengths:** Real-time performance on GPUs; handles non-homogeneous media; significantly faster than naive gathering.
*   **Limitations:** Requires analytically integrable density functions; screen-space approach disregards attenuation/scattering before surface reflection; potential for residual visual artifacts compared to more accurate but slower methods.
*   **When to Use:** Real-time rendering of multiple scattering in non-homogeneous media where performance is critical and approximate results are acceptable.
*   **When Not to Use:** When high accuracy is required (e.g., offline rendering), or when density functions are not analytically integrable.

## Background
*   **Origin:** Proposed by Elek et al. [2013] as a solution to reduce visual artifacts and computational cost in multiple-scattering light transport [S1, unofficial].
*   **History:** The method is part of a broader effort to enable real-time rendering of complex light transport effects in participating media. It builds on the concept of treating screen-space pixels as light sources and applying filtering to approximate multiple scattering [S1, unofficial].
*   **Usage:** Implemented in fragment shaders for GPU execution, tested on hardware like the GeForce GTX 760M [S1, unofficial]. It is used in contexts requiring real-time performance, such as video games or interactive applications, where traditional path-tracing is too slow.

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
