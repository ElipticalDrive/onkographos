# Neural Photon Field Rendering

Also called: Neural Path Tracer, Neural Photon Field Rendering Pipeline, PFF, Photon Field Network, Wavefront Path Tracing with Neural Fields

## Overview
Neural Photon Field Rendering is a GPU-based path tracing method that estimates direct illumination via standard ray-marching techniques and approximates indirect global illumination by querying a pre-trained neural field [S1]. Unlike traditional path tracers that recursively trace secondary rays for global illumination, this method computes only the first volume interaction and uses the neural network to predict the incoming radiance $L_i$ from the rest of the scene [S1].

## How it works
The method operates as a wavefront path tracer [S1]. It replaces the expensive secondary ray tracing required for global illumination with a neural network inference step, significantly reducing stochastic noise and computational cost at low sample counts [S1].

### Inputs and Outputs
*   **Inputs:** Camera parameters, volume density field, phase function coefficients ($g$), pre-trained neural photon field network, number of samples per pixel ($SPP$) [S1].
*   **Outputs:** Final radiance $L$ per pixel [S1].

### Data Structures
1.  **Rays Buffer:** Stores ray origins, directions, and intersection status for rays that intersect the volume [S1].
2.  **Samples Buffer:** A fixed-size buffer of size $|samples|_{max} = SPP \times |rays|$ [S1]. Each entry stores the 3D position, the phase coefficient $g$ used for that sample, and validity flags [S1]. Invalid interactions are left blank [S1].
3.  **Neural Network Input:** A 6-dimensional vector consisting of the 3D position $(x, y, z)$ and the phase coefficient $g$ [S1]. The position is encoded using hashgrid encoding [S1].

### Parameters
*   **SPP:** Samples per pixel. Typically 1 for performance measurement, but supports frame accumulation or denoising [S1].
*   **Maximum Path Length:** Set to 16 for all rendering evaluations [S1].
*   **Phase Coefficients ($G$):** The network is trained on a discrete subset $G \subset [-1, 1]$ [S1]. It generalizes to unseen $g \notin G$ [S1].
*   **Photon Map Size:** The underlying data is generated from photon maps consisting of 12M total photons, distributed as 4M photons per phase [S1].

### Steps
1.  **Ray Generation:** Spawn view rays based on camera parameters. Perform basic intersection testing against the volume bounding box. Store intersecting rays in the Rays Buffer [S1].
2.  **Volume Sampling:** For each ray in the buffer, use delta tracking to estimate the depth of the first interaction with the volume. Generate $SPP$ samples per ray. Store the interaction positions and the corresponding phase coefficient $g$ in the Samples Buffer. Invalid interactions are marked as blank [S1].
3.  **Photon Field Prediction:** Batch all valid samples from the Samples Buffer into a single neural network inference step. The network takes the 6D input (position + $g$) and outputs the predicted incoming radiance $L_i^{pred}$ [S1]. This is decoded using the network's output layer (Equation 7 in [S1]) to obtain the final radiance value [S1].
4.  **Compaction:** Compact the Samples Buffer by discarding samples where delta tracking did not generate a valid interaction, removing 'holes' to optimize subsequent memory access [S1].
5.  **Direct Illumination:** Use Next-Event Estimation (NEE) and delta tracking to sample light sources directly from the generated sample points. This computes the direct illumination term $L_d$ [S1].
6.  **Composition:** Compute the scattering probability $\sigma_s$ at each sample point. Determine Multiple Importance Sampling (MIS) weights $w_d$ (for NEE) and $w_i$ (for the neural network prediction) [S1]. Compose the total illumination using the formula below [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_photon_field_rendering(camera, neural_field, spp, volume_bounds, lights):
    # 1. Ray Generation
    rays = []
    for pixel in camera.raster:
        ray = camera.spawn_view_ray(pixel)
        if ray.intersects(volume_bounds):
            rays.append((pixel, ray))

    pixel_radiance = {}
    for pixel, ray in rays:
        samples = []
        # 2. Volume Sampling
        for _ in range(spp):
            depth = delta_tracking(ray, volume)
            if depth is not None:
                pos = ray.origin + ray.direction * depth
                samples.append({'pos': pos, 'g': ray.phase_coefficient, 'valid': True})
            else:
                samples.append({'pos': None, 'g': None, 'valid': False})

        # 4. Compaction (filter invalid samples)
        valid_samples = [s for s in samples if s['valid']]
        if not valid_samples:
            pixel_radiance[pixel] = 0.0
            continue

        # 3. Photon Field Prediction
        inputs = []
        for s in valid_samples:
            pos_enc = hashgrid_encode(s['pos'])
            inputs.append(np.concatenate([pos_enc, [s['g']]]))
        raw_output = neural_field.forward(np.array(inputs))
        li_pred = np.exp(raw_output)  # [G] Equation 7 decoding operation

        # 5. Direct Illumination
        ld = []
        for s in valid_samples:
            ld.append(next_event_estimation(s['pos'], volume, lights))

        # 6. Composition
        sigma_s = [compute_scattering_probability(s) for s in valid_samples]
        w_d, w_i = compute_mis_weights(valid_samples, ld, li_pred)

        total = 0.0
        for k in range(len(valid_samples)):
            total += w_d[k] * ld[k] + w_i[k] * sigma_s[k] * li_pred[k]
        pixel_radiance[pixel] = total / spp

    return pixel_radiance

# Helper definitions
def delta_tracking(ray, volume):
    # [G] Standard delta tracking algorithm to find first interaction depth
    pass

def hashgrid_encode(pos):
    # [G] Standard hashgrid positional encoding
    pass

def next_event_estimation(pos, volume, lights):
    # [G] Standard NEE sampling and shadow ray testing
    pass

def compute_scattering_probability(sample):
    # [G] sigma_s = sigma_t - sigma_a at sample position
    pass

def compute_mis_weights(samples, ld, li_pred):
    # [G] Power heuristic MIS weights for direct and indirect terms
    pass
```

### Edge Cases and Pitfalls
*   **High Transparency/Gradients:** Highly transparent volumes and regions with high density gradients (e.g., air-surface boundaries) suffer from insufficient photon density in the training data, causing the averaged radiance to underestimate true illumination [S1].
*   **Unseen Phase Coefficients:** Evaluating the network on phase coefficients $g \notin G$ introduces error compounded by both prediction bias and the inherent stochasticity of the photon tracing used to train the field [S1].
*   **Delta Tracking Failure:** If delta tracking fails to find a valid interaction within the volume bounds, the sample is marked invalid and discarded during compaction [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The rendering time is dominated by the neural network inference and direct illumination sampling, avoiding the exponential cost of secondary ray tracing for global illumination [S1].
*   **Memory Complexity:** Requires storing the neural network weights and the samples buffer ($SPP \times |rays|$) [S1]. The pre-computed photon maps are not stored during rendering but are used for training [S1].
*   **Performance:** Achieves an average speedup of 4–5× over a conventional path tracer [S1]. The speedup for generating the indirect illumination term ($L_i$) reaches up to 13× [S1]. Frame time speedup strongly correlates with the screen fill ratio of the rendered volume [S1].
*   **Image Quality:** Exhibits significantly less stochastic noise than path tracers at low SPP (e.g., 1–4 SPP) [S1]. Noise profiles improve faster as SPP increases [S1]. The method is suitable for real-time applications or denoising pipelines [S1].
*   **Limitations:** Quality is limited by the quality of the underlying photon maps (e.g., 12M photons) [S1]. Low opacity volumes and boundary gradients may suffer from underestimation [S1]. Multi-phase photon map tracing times on CPU range from 4.2s (1M photons) to 282s (12M photons) depending on dataset size and density [S1].

## Background
The method was introduced in "Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination" [S1]. It builds on the concept of photon mapping and neural field approximation [S1]. The authors propose using a neural network to approximate the radiance field learned from photon maps, enabling real-time rendering of global illumination in participating media [S1]. The approach is designed to be implemented as a GPU path tracer, leveraging wavefront parallelism [S1].

## Sources
[S1] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
