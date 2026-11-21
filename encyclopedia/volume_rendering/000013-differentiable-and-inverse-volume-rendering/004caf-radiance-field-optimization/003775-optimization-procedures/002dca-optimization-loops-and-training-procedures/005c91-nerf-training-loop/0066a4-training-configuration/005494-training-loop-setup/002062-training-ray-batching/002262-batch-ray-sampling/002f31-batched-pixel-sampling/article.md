# Batched Pixel Sampling


## Overview
Batched Pixel Sampling is a technique for selecting a subset of pixels and their associated sensors for rendering in differentiable inverse volume rendering, generating separate samplers for pixel selection, primal subpixel sampling, and adjoint subpixel sampling [S1, unofficial]. It serves as a form of pixel-level mini-batching that accelerates inverse rendering by dynamically adjusting pixel sampling probabilities based on gradient variance and contribution to the loss function, rather than uniformly reducing the sampling budget [S2, unofficial]. This approach outperforms uniform mini-batching by avoiding the high gradient variance that degrades performance when naively reducing the sampling budget [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch size, a seed, the number of samples per pixel (SPP) for the primal pass, the number of samples for the adjoint pass (often derived from SPP), and the number of available sensors.
*   **Outputs:** An array of sensor indices, an array of pixel coordinates, and a list of three independent samplers (one for pixel selection, one for primal subpixel sampling, one for adjoint subpixel sampling).

**Data Structures:**
*   `batch_samplers`: A list of three sampler objects. Each sampler is of type `independent` [S1, unofficial].
*   `sensor_idx`: An array of unsigned 32-bit integers, size `batch_size`, storing the selected sensor index for each ray [S1, unofficial].
*   `pixels`: An array of 2D unsigned integer points, size `batch_size`, storing the pixel coordinates [S1, unofficial].

**Parameters:**
*   `batch_size`: The number of pixels to sample in the current batch.
*   `spp`: Samples per pixel for the primal pass.
*   `spp_grad`: Samples per pixel for the adjoint pass.
*   `seed`: The random seed for the sampling process.
*   `film_size`: The dimensions of the film/sensor in pixels.
*   `n_sensors`: The total number of sensors available.

**Steps:**
1.  **Determine Sensor Count:** Calculate the number of sensors, `n_sensors`, from the sensor data structure [S1, unofficial].
2.  **Create Samplers:** Initialize three independent samplers with sizes `batch_size`, `batch_size * spp`, and `batch_size * spp_grad` respectively [S1, unofficial].
3.  **Seed Samplers:** Seed each sampler using a derived sub-seed. The sub-seed is calculated as `mi.sample_tea_32(seed, 17 * i + 5)[0]`, where `i` is the index of the sampler (0, 1, or 2), and `seed` is the input seed. The wavefront size is also passed to the sampler initialization [S1, unofficial].
4.  **Select Sensors:** For each ray in the batch, select a sensor index by sampling from the first sampler (pixel selection sampler) and mapping it to the sensor range: `sensor_idx = mi.UInt32(n_sensors * batch_samplers[0].next_1d())` [S1, unofficial].
5.  **Sample Pixels:** For each ray in the batch, sample individual pixel coordinates by sampling a 2D float value from the first sampler and scaling it by the film size: `pixels = mi.Point2u(mi.Point2f(film_size) * batch_samplers[0].next_2d())` [S1, unofficial].
6.  **Return Results:** Return the `sensor_idx`, `pixels`, and the `batch_samplers` list [S1, unofficial].

**Adaptive Weighting (Contextual Usage):**
While the core batching mechanism selects pixels uniformly or via a base sampler, the *selection* of which pixels to prioritize in the batch can be driven by adaptive weights [S2, unofficial][S3].
*   Compute the adjoint radiance $\frac{\partial I}{\partial L}$ from the primal rendering pass [S3].
*   Calculate per-pixel weights proportional to the absolute value of the adjoint radiance: $w_i \propto |\frac{\partial I}{\partial L}_i|$ [S3].
*   Use these weights to determine the probability of selecting each pixel for the batch [S2, unofficial][S3].
*   Note: The adjoint radiance can have negative values. Sampling proportional to $|\frac{\partial I}{\partial L}|$ introduces sign variance. Positivization (sampling positive and negative components separately) can remove this variance but was found to have little effect on overall estimation variance in practice, so the simpler absolute value method is used [S3].

**Sample Budget Allocation:**
*   The sample budget is split between the primal rendering pass (computing the image) and the adjoint rendering pass (computing the loss gradient) [S3].
*   The optimal ratio found is 3:1 (75% primal, 25% adjoint) [S3].

**Pitfalls and Edge Cases:**
*   **Gradient Variance:** Naively reducing the sampling budget by uniformly sampling pixels results in high gradient variance, degrading performance [S2, unofficial]. Adaptive weighting mitigates this [S2, unofficial].
*   **Sign Variance:** Using $|\frac{\partial I}{\partial L}|$ for sampling weights introduces sign variance. While positivization is an option, it adds complexity with minimal benefit [S3].
*   **Film Reconstruction Filter:** The method assumes a box reconstruction filter, as proximity of rays in wavefront order gives no information about true proximity in sensor space [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_batch_pixels(batch_size, spp, spp_grad, seed, film_size, n_sensors):
    """
    Samples a batch of pixels and sensors for rendering.
    
    Args:
        batch_size (int): Number of pixels to sample.
        spp (int): Samples per pixel for primal pass.
        spp_grad (int): Samples per pixel for adjoint pass.
        seed (int): Random seed.
        film_size (tuple): (width, height) of the film in pixels.
        n_sensors (int): Number of available sensors.
        
    Returns:
        sensor_idx (list): List of sensor indices for each ray.
        pixels (list): List of pixel coordinates for each ray.
        batch_samplers (list): List of three independent samplers.
    """
    # Step 2: Create three independent samplers
    # Sizes: batch_size, batch_size * spp, batch_size * spp_grad
    batch_samplers = []
    for i in range(3):
        sampler_size = batch_size * (spp if i == 1 else (spp_grad if i == 2 else 1))
        # Assuming an independent sampler constructor exists
        batch_samplers.append(IndependentSampler(size=sampler_size))
    
    # Step 3: Seed each sampler
    for i, sampler in enumerate(batch_samplers):
        # Derive sub-seed using TEA hash
        sub_seed = tea_32_hash(seed, 17 * i + 5)
        sampler.seed(sub_seed)
        # Wavefront size is typically batch_size or a multiple thereof
        sampler.set_wavefront_size(batch_size)
    
    # Step 4: Select sensor for each ray
    sensor_idx = []
    for i in range(batch_size):
        # Sample a 1D float from the first sampler
        u = batch_samplers[0].next_1d()
        # Map to sensor index
        idx = int(n_sensors * u)
        sensor_idx.append(idx)
    
    # Step 5: Sample individual pixels
    pixels = []
    for i in range(batch_size):
        # Sample a 2D float from the first sampler
        u2 = batch_samplers[0].next_2d()
        # Scale by film size and convert to integer coordinates
        px = int(u2.x * film_size[0])
        py = int(u2.y * film_size[1])
        pixels.append((px, py))
    
    # Step 6: Return results
    return sensor_idx, pixels, batch_samplers

def tea_32_hash(seed, offset):
    """
    Simplified TEA-like hash for generating sub-seeds.
    This is a placeholder for the actual mi.sample_tea_32 implementation.
    """
    # Simple hash function for demonstration
    h = seed + offset
    h = (h ^ 0x5bd1e995) & 0xFFFFFFFF
    h = (h * 0x5bd1e995) & 0xFFFFFFFF
    h = (h ^ (h >> 15)) & 0xFFFFFFFF
    return h
```

## Operating characteristics
*   **Time Complexity:** The sampling step itself is $O(N)$ where $N$ is the batch size. The adaptive weighting calculation requires a full primal render pass, which is $O(M \cdot S)$ where $M$ is the total number of pixels and $S$ is the number of samples per pixel.
*   **Memory Complexity:** $O(N)$ for storing the batched rays and samplers. The adaptive weights require $O(M)$ storage if computed for all pixels.
*   **Measured Performance:** Adaptive sampling combined with differential sampling reduces loss gradient variance by 1–2 orders of magnitude compared to uniform sampling or BRDF-only sampling [S3]. It outperforms uniform mini-batching by allowing faster convergence at equal time [S2, unofficial].
*   **Strengths:** Reduces computational cost by rendering only a subset of pixels per iteration while maintaining low gradient variance [S2, unofficial]. The 3:1 primal-to-adjoint sample budget ratio is optimal for variance reduction [S3].
*   **Limitations:** Requires a primal pass to compute adjoint radiance for adaptive weighting, adding overhead. The method assumes a box reconstruction filter [S1, unofficial].
*   **When to Use:** When performing inverse rendering with high-resolution images where rendering all pixels per iteration is too costly, and uniform subsampling leads to high gradient variance.
*   **When Not to Use:** When the scene is small enough that full rendering is feasible, or when the overhead of computing adaptive weights outweighs the benefits of subsampling.

## Background
Batched Pixel Sampling is part of the broader field of differentiable rendering and inverse rendering. It addresses the computational challenges of differentiable rendering, particularly in inverse rendering from high-resolution/multiview images [S2, unofficial]. The technique was introduced as part of an image-space adaptive sampling framework for fast inverse rendering [S2, unofficial]. The specific implementation details regarding samplers and seeds are found in the unbiased inverse volume rendering codebase [S1, unofficial]. The theoretical basis for using adjoint radiance for importance sampling in differential rendering is discussed in path sampling methods for differentiable rendering [S3].

## Sources
[S1] Unbiased Inverse Volume Rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Variance-aware Differentiable Rendering, https://core.ac.uk/works/337282320, fetched 2026-11-02 (unofficial)
[S3] Path Sampling Methods for Differentiable Rendering, https://doi.org/10.2312/sr.20241148, fetched 2026-10-29
