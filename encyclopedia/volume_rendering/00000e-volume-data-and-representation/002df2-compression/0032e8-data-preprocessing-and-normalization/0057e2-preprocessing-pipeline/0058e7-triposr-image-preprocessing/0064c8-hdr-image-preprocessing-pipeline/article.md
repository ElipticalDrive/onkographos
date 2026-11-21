# HDR Image Preprocessing Pipeline

Also called: Bracketed Exposure HDR Construction, CRF Recovery and Normalization

## Overview
This pipeline converts a series of bracketed Low Dynamic Range (LDR) exposures into a normalized 32-bit High Dynamic Range (HDR) radiance image suitable for training Neural Radiance Fields (NeRF) and diffusion-based inverse rendering models. It solves the problem of capturing and representing scene illumination beyond the limited range of standard 8-bit sensors by recovering the camera's nonlinear tone-mapping function and linearizing the data. It differs from standard LDR preprocessing by explicitly reconstructing linear radiance values and preserving relative intensity differences through uniform scaling rather than quantization, ensuring numerical stability during deep learning training [S1, S2].

## How it works
**Inputs:** A set of bracketed LDR image series captured from fixed camera poses. Each series consists of $n$ images (typically 7) with varying exposure durations.
**Outputs:** A single 32-bit HDR image per series, stored as an EXR file, with radiance values linearly scaled to the range $[0, 255]$.

**Data Structures:**
*   **LDR Input:** $n$ 8-bit JPEG images, dimensions $W \times H$.
*   **Aligned Series:** $n$ images of same dimensions, registered to a common coordinate frame.
*   **CRF Model:** A function mapping LDR pixel values to linear radiance.
*   **HDR Image:** 32-bit floating-point image, dimensions $W/4 \times H/4$.

**Steps:**

1.  **Capture:** Acquire a bracketed series of $n$ LDR images from a fixed camera pose.
    *   Use a tripod to minimize pose perturbations [S1].
    *   Keep aperture fixed; vary only exposure duration (shutter speed) [S1].
    *   Maintain a constant relative exposure difference of 1–2 stops between consecutive images [S1].
    *   Ensure the dynamic range covers the scene's limits, such that every pixel is "well-exposed" in at least one image [S1].

2.  **Alignment:** Align the $n$ images in the series to compensate for minor camera pose perturbations. This step is critical to prevent degradation in the final HDR reconstruction [S1].

3.  **Representative Selection:** Select one "representative" bracketed series from the dataset. This series should roughly cover the upper and lower limits of the overall dynamic range of the lighting conditions [S1].

4.  **CRF Recovery:** Recover the Camera Response Function (CRF), which is the camera's nonlinear tone-mapping function, from the representative series.
    *   Use the method described in [17] (referenced in [S1]).
    *   The CRF maps LDR pixel values to linear radiance. While the linear system could theoretically incorporate pixels from all series, solving it on one representative series is sufficient for consistent exposure levels across the dataset [S1].

5.  **CRF Application:** Apply the recovered CRF uniformly to all bracketed series in the dataset. This ensures consistent exposure levels and linear radiance values across all images [S1].

6.  **Downsampling:** Downsample each resulting 32-bit HDR image by a factor of 4x to improve computational efficiency [S1].

7.  **Normalization:** Normalize all radiance values by linearly scaling them to the range $[0, 255]$.
    *   This scaling is uniform, preserving relative differences between values [S1].
    *   Values are not quantized to integers; they remain 32-bit floats to avoid quantization artifacts [S1].
    *   Store the final images as EXR files [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_hdr_pipeline(ldr_series_list, crf_recovery_method):
    """
    Preprocess bracketed LDR exposures into normalized HDR images.
    
    Args:
        ldr_series_list: List of lists, where each inner list contains n LDR images (8-bit JPEGs)
                         from a single bracketed capture.
        crf_recovery_method: Function to recover CRF from a representative series.
        
    Returns:
        List of normalized 32-bit HDR images (EXR format data).
    """
    aligned_series_list = []
    
    # Step 1 & 2: Capture and Align
    # Note: Capture is assumed done externally. Alignment is performed here.
    for series in ldr_series_list:
        # Align images in the series to a common pose
        # Mitigates slight camera pose perturbations [S1]
        aligned_series = align_images(series)
        aligned_series_list.append(aligned_series)
        
    # Step 3: Select Representative Series
    # Choose one series that covers the full dynamic range
    representative_series = select_representative_series(aligned_series_list)
    
    # Step 4: Recover CRF
    # CRF maps LDR values to linear radiance [S1]
    crf = crf_recovery_method(representative_series)
    
    # Step 5: Apply CRF and Downsample
    hdr_images = []
    for aligned_series in aligned_series_list:
        # Apply CRF to convert LDR to linear radiance
        linear_radiance_images = apply_crf(aligned_series, crf)
        
        # Step 6: Downsample by factor of 4
        # Improves computational efficiency [S1]
        downscaled_image = downsample(linear_radiance_images[0], factor=4)
        
        # Step 7: Normalize to [0, 255]
        # Linear scaling preserving relative differences, no quantization [S1]
        min_val = downscaled_image.min()
        max_val = downscaled_image.max()
        normalized_image = (downscaled_image - min_val) / (max_val - min_val) * 255.0
        
        hdr_images.append(normalized_image)
        
    return hdr_images

def align_images(images):
    """
    Align a series of images to compensate for camera pose perturbations.
    Uses feature matching and homography estimation.
    """
    # Implementation details for alignment are not specified in sources [S1]
    # Standard computer vision techniques (e.g., OpenCV) are implied [S1]
    reference_image = images[0]
    aligned = [reference_image]
    for i in range(1, len(images)):
        # Estimate transformation from reference to current image
        transform = estimate_transform(reference_image, images[i])
        # Warp current image to reference frame
        aligned_image = warp_image(images[i], transform)
        aligned.append(aligned_image)
    return aligned

def apply_crf(images, crf):
    """
    Apply the Camera Response Function to convert LDR values to linear radiance.
    """
    # CRF is a function f: LDR -> Radiance
    # Applied uniformly to all images in the series [S1]
    linear_images = []
    for img in images:
        # Map each pixel value through the CRF
        linear_img = crf(img)
        linear_images.append(linear_img)
    return linear_images

def downsample(image, factor):
    """
    Downsample an image by a given factor.
    """
    # Factor of 4 is used for efficiency [S1]
    new_width = image.width // factor
    new_height = image.height // factor
    # Use appropriate interpolation (e.g., bilinear or bicubic)
    return resize(image, new_width, new_height)
```

**Parameters:**
*   **Bracket Size ($n$):** Number of LDR exposures per HDR image. Typical value: 7 [S1].
*   **Exposure Step:** Relative difference in exposure between consecutive images. Typical value: 1–2 stops [S1].

**Edge Cases and Pitfalls:**
*   **Camera Pose Perturbations:** Slight movements during capture can degrade HDR quality. Mitigated by using a tripod and performing image alignment [S1].
*   **Dynamic Range Coverage:** If the bracketed series does not cover the full dynamic range, some pixels may be clipped in all exposures. Ensure the series covers upper and lower limits [S1].
*   **Quantization:** Normalization must not quantize values to integers. Values must remain 32-bit floats to preserve relative differences [S1].

## Subtopics

### HDR Compression for Stable Training
A preprocessing step used in diffusion-based inverse rendering to ensure stable training of DDPM models. It compresses HDR content using a global invertible mapping before training and applies the inverse mapping before path tracing. For outdoor data, the mapping is $f(x) = 0.5 x^{1/2.4}$; for indoor data, it is $f(x) = 0.9 x^{1/6}$ [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by image alignment and CRF recovery. CRF recovery involves solving a linear system, which can be computationally intensive if all pixels are used, but is optimized by using a single representative series [S1].
*   **Memory Complexity:** Requires storing $n$ LDR images and the resulting 32-bit HDR images. Downsampling by 4x reduces memory usage by 16x for the HDR images [S1].
*   **Performance:** The pipeline is designed for computational efficiency, with downsampling and single-series CRF recovery reducing training time [S1]. Training HDR NeRF models with this data is comparable to LDR training, with Instant-NGP supporting 32-bit HDR inputs [S1].
*   **Strengths:** Preserves high dynamic range information, ensures numerical stability during training, and is compatible with standard NeRF implementations [S1].
*   **Limitations:** Requires careful capture setup (tripod, fixed aperture) and alignment. CRF recovery assumes a consistent camera response across all images [S1].
*   **When to Use:** When training HDR NeRF models or comparing against LDR counterparts. Also used in diffusion-based inverse rendering for stable training [S1, S2].
*   **When Not to Use:** When computational resources are extremely limited and HDR is not required, or when camera pose cannot be stabilized during capture.

## Background
The HDR Image Preprocessing Pipeline was introduced in the context of training HDR NeRF models. It builds on the standard NeRF formulation by extending it to handle high dynamic range data. The method for recovering the CRF is based on the work of [17] (referenced in [S1]). The pipeline has been used in experiments with the Instant-NGP implementation, which supports 32-bit HDR images in EXR format [S1]. It is also used in diffusion-based inverse rendering for stable training of DDPM models [S2].

## Sources
[S1] Dynamic Mesh-Aware Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00042, fetched 2026-10-26
[S2] Diffusion Posterior Illumination for Ambiguity-Aware Inverse Rendering, https://doi.org/10.1145/3618357, fetched 2026-10-26
