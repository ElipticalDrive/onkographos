# Google Immersive View NeRF

Also called: Indoor NeRF Reconstruction, NeRF Reconstruction for Immersive View

## Overview
A method for 3D reconstruction and novel view synthesis that distills a dense collection of indoor photos into a neural field to render photorealistic views from viewpoints not present in the original collection, specifically optimized for the scale, lighting variability, and privacy constraints of Google Maps Immersive View [S1, unofficial]. It distinguishes itself from standard NeRF implementations by integrating low-dimensional generative latent optimization (GLO) vectors and metadata-derived exposure conditioning into the radiance network, thereby resolving cloudy geometry artifacts caused by lighting changes without sacrificing interpretability [S1, unofficial]. Unlike [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md) which focuses on sampling strategies, this method focuses on appearance modeling and capture constraints; it builds on [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md) as its architectural base.

## How it works
### Inputs and Outputs
**Inputs:**
1.  **Capture Data:** A dense collection of DSLR photographs of an indoor venue, captured within a short timeframe (typically <1 hour) [S1, unofficial].
2.  **Metadata:** Photo metadata including ISO, aperture, and white balance settings [S1, unofficial].
3.  **Camera Parameters:** Extrinsic (position, orientation) and intrinsic (focal length) parameters solved via Structure-from-Motion (SfM) [S1, unofficial].

**Outputs:**
1.  **NeRF Model:** A trained neural radiance field specific to the location [S1, unofficial].
2.  **Immersive Video:** 360° rendered video along a smoothed flight path [S1, unofficial].

### Data Structures
The method utilizes the standard Mip-NeRF 360 architecture [S1, unofficial]. The key structural modification is in the **Radiance Network** inputs:
*   **Positional/Orientation Encodings:** Standard positional and angular encodings for the 3D point and view direction [S1, unofficial].
*   **GLO Vectors:** A set of learned real-valued latent vectors, one per input image, appended to the radiance network inputs [S1, unofficial].
*   **Exposure Conditioning:** Scalar or vector values derived directly from photo metadata (ISO, etc.), appended to the radiance network inputs [S1, unofficial].

### Numbered Steps

1.  **Capture:** Photograph the venue using DSLR cameras. Ensure every surface is observed from multiple directions. Keep camera properties (white balance, aperture) fixed. Freeze the scene (no movement, stable lighting) [S1, unofficial].
2.  **Privacy Processing:** Automatically scan uploaded photos and blur personally identifiable content (faces, license plates) [S1, unofficial].
3.  **Structure-from-Motion (SfM):** Run an SfM pipeline to solve for each photo's camera parameters (position, orientation, focal length). Associate each pixel with a 3D point and direction [S1, unofficial].
4.  **Scale Calibration:** Capture pairs of images exactly 3 meters apart during the session. Use this known distance to scale the generated 3D model for consistent video velocity [S1, unofficial].
5.  **Model Initialization:** Initialize a new NeRF model from scratch for the location, using Mip-NeRF 360 as the base architecture [S1, unofficial].
6.  **Latent Vector Assignment:** Assign a unique low-dimensional generative latent optimization (GLO) vector to each input image. These vectors are learned during training [S1, unofficial].
7.  **Exposure Conditioning Extraction:** Extract exposure values from the photo metadata for each image [S1, unofficial].
8.  **Radiance Network Input Construction:** For each ray sampled during training, construct the input vector for the radiance network comprising:
    *   Standard positional/oriental encodings.
    *   The GLO vector corresponding to the source image of the ray.
    *   The exposure conditioning value corresponding to the source image [S1, unofficial].
9.  **Training:** Train the model from scratch using the Mip-NeRF 360 optimization loop, minimizing rendering loss against the input photos [S1, unofficial].
10. **Video Rendering:** Define a flight path by interpolating keyframe positions computed during SfM. Render 360° videos by ray casting through a virtual spherical camera along this path [S1, unofficial].

### Formulas
The sources do not provide explicit mathematical formulas for the GLO vector integration or exposure conditioning. The radiance network function $\mathcal{F}$ can be conceptually described as:

$$
(\sigma, \mathbf{c}) = \mathcal{F}(\mathbf{x}, \mathbf{d}, \mathbf{z}_i, \mathbf{e}_i)
$$

Where:
*   $\mathbf{x}$ is the 3D position.
*   $\mathbf{d}$ is the view direction.
*   $\mathbf{z}_i$ is the GLO vector for image $i$.
*   $\mathbf{e}_i$ is the exposure conditioning for image $i$.
*   $\sigma$ is the density.
*   $\mathbf{c}$ is the color.

*Note: Exact architectural details of the Mip-NeRF 360 base are assumed from the parent article [3426-mip-nerf-360-optimization-setup.md].*

### Parameters
*   **Capture Timeframe:** < 1 hour [S1, unofficial].
*   **Calibration Distance:** 3 meters between image pairs [S1, unofficial].
*   **Hardware:** TPUs or GPUs [S1, unofficial].

### Edge Cases and Pitfalls
*   **Lighting Changes:** If lighting changes during capture, reconstruction quality suffers significantly [S1, unofficial].
*   **Cloudy Geometry:** Without GLO vectors or exposure conditioning, casual NeRF captures exhibit "cloudy geometry" artifacts when handling lighting variations [S1, unofficial].
*   **Scene Movement:** The scene must be frozen in time; any movement degrades reconstruction [S1, unofficial].
*   **Privacy:** Failure to blur personally identifiable content violates privacy requirements [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_immersive_nerf(photos, metadata, sfm_results):
    """
    Trains a NeRF model for Immersive View using GLO and Exposure conditioning.
    
    Args:
        photos: List of image arrays.
        metadata: List of metadata dicts containing 'iso', 'aperture', etc.
        sfm_results: Dict mapping image index to camera parameters (position, orientation, focal_length).
        
    Returns:
        trained_model: The trained NeRF model.
    """
    # 1. Privacy Processing
    blurred_photos = [blur_pii(photo) for photo in photos]
    
    # 2. Initialize Mip-NeRF 360 Base Model
    # Assumes base architecture from parent article
    model = MipNeRF360Base()
    
    # 3. Initialize GLO Vectors
    # One latent vector per image
    num_images = len(blurred_photos)
    glo_vectors = initialize_latent_vectors(num_images, dimension=16) # Dimension not specified, assumed small [G]
    
    # 4. Extract Exposure Conditioning
    exposure_values = [extract_exposure(meta) for meta in metadata]
    
    # 5. Scale Calibration
    # Use 3m pairs to scale the scene coordinates
    scale_factor = compute_scale_factor(sfm_results, calibration_distance_meters=3.0)
    sfm_results = scale_sfm_results(sfm_results, scale_factor)
    
    # 6. Training Loop
    for epoch in range(max_epochs):
        for i in range(num_images):
            image = blurred_photos[i]
            camera_params = sfm_results[i]
            glo_vec = glo_vectors[i]
            exposure_val = exposure_values[i]
            
            # Sample rays for this image
            rays = sample_rays_from_image(image, camera_params)
            
            # Render using Radiance Network with auxiliary inputs
            # The radiance network receives: position, direction, glo_vec, exposure_val
            rendered_colors = render_rays(model, rays, glo_vec, exposure_val)
            
            # Compute Loss
            loss = compute_rendering_loss(rendered_colors, image)
            
            # Backpropagate
            update_model(model, loss)
            update_latent_vectors(glo_vectors, loss) # GLO vectors are learned parameters [G]
            
    return model

def render_immersive_video(model, keyframe_indices, sfm_results, video_length_frames):
    """
    Renders a 360 video by interpolating keyframes.
    """
    # 1. Define Flight Path
    # Interpolate camera positions from SfM results of keyframes
    path = interpolate_path(sfm_results[keyframe_indices], video_length_frames)
    
    # 2. Render Frames
    frames = []
    for camera_pos in path:
        # Ray cast through virtual spherical camera
        rays = cast_spherical_rays(camera_pos)
        # Render using the trained model
        # Note: For video rendering, GLO and Exposure might be averaged or set to neutral if not tied to a specific source image
        # The source does not specify how GLO/Exposure are handled for novel view video rendering, 
        # only for training against source images. [G]
        color = render_rays(model, rays) 
        frames.append(color)
        
    return frames
```

## Subtopics

### Mip-NeRF 360 Optimization Setup
The base architecture used for the radiance field, providing anti-aliased cone tracing and positional encoding. [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)

### Anti-Aliased Interlevel Loss
A loss function technique often associated with mip-NeRF variants to reduce aliasing artifacts. [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md)

### Isotropic Gaussian Scale Contraction
A technique for handling scale invariance or contraction in neural fields, potentially relevant for the latent space or encoding. [Isotropic Gaussian Scale Contraction](5407-isotropic-gaussian-scale-contraction.md)

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive. The method uses Mip-NeRF 360, which is noted as "more computationally intensive" than Nvidia's Instant NGP [S1, unofficial].
*   **Memory Complexity:** Requires storing GLO vectors for each image and the full Mip-NeRF 360 network weights. Not given in the sources.
*   **Measured Performance:** Produces fewer artifacts and higher reconstruction quality than Instant NGP [S1, unofficial].
*   **Strengths:**
    *   Handles lighting changes and varying exposure without cloudy geometry artifacts [S1, unofficial].
    *   Provides interpretable exposure control (ISO/brightness) at inference time [S1, unofficial].
    *   Consistent quality across diverse indoor venues (dimly-lit bars to cafes) [S1, unofficial].
*   **Limitations:**
    *   Requires strict capture constraints (frozen scene, fixed camera properties) [S1, unofficial].
    *   High computational cost compared to faster alternatives like Instant NGP [S1, unofficial].
    *   GLO vectors are uninterpretable latent codes, though they solve the artifact problem [S1, unofficial].
*   **When to use:** When high-quality, artifact-free indoor reconstruction is required for user-facing products like Immersive View, and computational resources are available.
*   **When not to use:** For real-time applications requiring lower latency (use Instant NGP), or when capture constraints cannot be met (lighting changes, movement).

## Background
The method was developed by Google to power the "Immersive View" feature in Google Maps, which provides indoor views of restaurants, cafes, and other venues [S1, unofficial]. It was described in a blog post in June 2023 [S1, unofficial]. The work builds on Alphabet's recent advances in neural radiance fields, specifically incorporating techniques from "NeRF in the Wild" (GLO vectors) and "Block-NeRF" (exposure conditioning) [S1, unofficial]. It was chosen over Instant NGP for its superior reconstruction quality despite higher compute requirements [S1, unofficial].

## Sources
[S1] Reconstructing Indoor Spaces with NeRF, https://ai.googleblog.com/2023/06/reconstructing-indoor-spaces-with-nerf.html, fetched 2026-10-26 (unofficial)
