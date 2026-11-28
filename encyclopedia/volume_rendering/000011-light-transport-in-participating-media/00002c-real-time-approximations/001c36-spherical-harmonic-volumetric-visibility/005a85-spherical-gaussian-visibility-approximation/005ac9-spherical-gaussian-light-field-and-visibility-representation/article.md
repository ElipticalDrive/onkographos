# Spherical Gaussian Light-Field and Visibility Representation

Also called: SG-based light-field, illumination-weighted environment visibility approximation

## Overview
A rendering technique that represents light fields and illumination-weighted environment visibility as sets of Spherical Gaussians (SGs), optimized via neural network backpropagation and stored in 2D textures for hardware-accelerated lookups [S1, unofficial]. It sets itself apart from its parent [Spherical Gaussian Visibility Approximation](16679-spherical-gaussian-visibility-approximation.md) by treating SG parameters (axis and sharpness) as learnable degrees of freedom rather than fixed analytical projections, allowing for fewer lobes and higher quality in static scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-convolved environment map (without visibility) and a pre-convolved environment map (with visibility) [S1, unofficial].
*   **Output:** A set of SG parameters (axis $\xi$, sharpness $\lambda$, amplitude $\mu$) for each texel in a 2D texture, representing the light field and visibility approximation [S1, unofficial].

**Data Structures:**
*   **SG Parameters:** Stored in 2D textures [S1, unofficial].
    *   **Layout:** Half-float RGB images [S1, unofficial].
    *   **Resolution:** 128x128 per direction [S1, unofficial].
    *   **Fields:** The texture stores the parameters required to evaluate the SGs [S1, unofficial].

**Steps:**
1.  **Compute Illumination-Weighted Environment Visibility:** For each texel, compute the visibility field by taking the ratio of the pre-convolved environment map with visibility to the pre-convolved environment map without visibility [S1, unofficial].
    $$ V_{weighted} = \frac{E_{with\_vis}}{E_{without\_vis}} $$
    Where $E$ represents the pre-convolved environment map [S1, unofficial].

2.  **Optimize SG Parameters:** Treat the SG parameters (axis $\xi$ and sharpness $\lambda$) as extra degrees of freedom [S1, unofficial]. Use a convolutional neural network (CNN) to estimate these parameters [S1, unofficial]. The network is trained by backpropagating the error between the predicted light-field image (evaluated from the SGs) and the target light-field image [S1, unofficial]. This implicit ordering ensures adjacent light-fields are similar [S1, unofficial].

3.  **Store Parameters:** Store the optimized SG parameters in 2D textures (typically 128x128 half-float RGB images) to allow for hardware texture lookups and filtering [S1, unofficial].

4.  **Estimate Reflected Light:** To estimate reflected light for a given direction:
    a. Evaluate the stored SGs in the direction of interest [S1, unofficial].
    b. Multiply the result by the pre-convolved environment map [S1, unofficial].
    $$ L_{reflected} \approx \left( \sum_{k} G(\omega; \xi_k, \lambda_k, \mu_k) \right) \times E_{without\_vis} $$

**Parameters:**
*   **Texture Resolution:** 128x128 half-float RGB images per direction [S1, unofficial].
*   **SG Lobes:** Fewer lobes are used compared to previous methods [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Static Scenes:** The method is optimized for static scenes [S1, unofficial]. Dynamic scene handling is not given in the sources.
*   **Precision:** Uses half-float precision [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_reflected_light(direction, sg_texture, env_map, num_lobes):
    """
    Estimates reflected light using stored SG parameters and environment map.
    
    Args:
        direction (vec3): The view direction.
        sg_texture (Texture2D): 128x128 half-float RGB texture storing SG parameters.
        env_map (Texture2D): Pre-convolved environment map without visibility.
        num_lobes (int): Number of SG lobes per texel.
        
    Returns:
        vec3: Estimated reflected light color.
    """
    # 1. Sample SG parameters from texture
    # Note: The exact mapping of RGB channels to SG parameters (axis, sharpness, amplitude)
    # is implementation-specific and not detailed in the source.
    sg_params = sg_texture.sample(direction)
    
    # 2. Evaluate SGs
    # G(nu; xi, lambda, mu) = mu * exp(lambda * (nu . xi - 1))
    # Sum over all lobes k
    sg_sum = vec3(0.0)
    for k in range(num_lobes):
        xi = sg_params[k].axis
        lambda_val = sg_params[k].sharpness
        mu = sg_params[k].amplitude
        dot_product = dot(direction, xi)
        sg_sum += mu * exp(lambda_val * (dot_product - 1.0))
        
    # 3. Multiply by pre-convolved environment map
    env_color = env_map.sample(direction)
    reflected_light = sg_sum * env_color
    
    return reflected_light
```

## Subtopics

### [Spherical Gaussian Environment Map Representation](18723-spherical-gaussian-environment-map-representation.md)
Represents distant environment illumination as a mixture of SG directional lobes, replacing traditional HDR environment maps or cubemaps [G]. It enables efficient, authorable, and analytically differentiable image-based lighting by allowing diffuse irradiance and specular prefiltered radiance to be computed in closed form [G]. Choose it when live derivation of lighting parameters without cubemap re-convolution is required [G].

## Operating characteristics
*   **Time Complexity:** Real-time performance, achieving full HD global-illumination rendering in 1-2ms [S1, unofficial].
*   **Hardware:** Tested on an RTX 2080 graphics card [S1, unofficial].
*   **Memory:** Uses 2D textures (128x128 half-float RGB images) for storage [S1, unofficial].
*   **Strengths:** Uses fewer SG lobes than previous methods while significantly improving rendering quality and performance for static scenes [S1, unofficial].
*   **Limitations:** Optimized for static scenes [S1, unofficial].

## Background
The method builds on the concept of using Spherical Gaussians for efficient approximation of the rendering equation [G]. It was introduced in the context of sparse voxel DAGs and memory-efficient compression of surface representations [S1, unofficial]. The approach leverages neural networks to optimize SG parameters, allowing for hardware-accelerated lookups [S1, unofficial].

## Sources
[S1] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
