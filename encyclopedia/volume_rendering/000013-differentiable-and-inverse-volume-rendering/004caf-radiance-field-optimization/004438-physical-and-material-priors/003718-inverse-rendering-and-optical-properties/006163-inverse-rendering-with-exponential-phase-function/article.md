# Inverse Rendering with Exponential Phase Function

Also called: Inverse Rendering with Proposed Phase Function, Physics-based Differential Renderer for Slab Estimation

## Overview
This method estimates the scattering parameters (extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and phase function parameters $\Phi_p$) of a homogeneous translucent slab from captured images, using a physics-based differentiable renderer [S1, unofficial]. It addresses the problem of fitting complex scattering profiles where standard phase functions fail, specifically by employing a three-degree exponential phase function [S1, unofficial]. It sets itself apart from siblings like [Volumetric Inverse Rendering with Differentiable Path Tracing](1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md) by focusing on analytical parameter estimation for specific slab geometries rather than general scene reconstruction, and from [NeRV (Neural Reflectance and Visibility Fields)](10246-nerv-neural-reflectance-and-visibility-fields.md) by relying on explicit physical priors rather than neural field representations [G].

## How it works
**Inputs:**
*   A set of $K$ captured images $I_{l_i}$ of a homogeneous translucent slab with known geometry and thickness, illuminated by a known beam light source at directions $l_i$ for $i=1 \dots K$ [S1, unofficial].
*   The geometry of the slab and the light source positions [S1, unofficial].
*   Initial guesses for $\sigma_t$, $\sigma_s$, and $\Phi_p$ [G].

**Outputs:**
*   Estimated parameters $\sigma_t^*, \sigma_s^*, \Phi_p^*$ [S1, unofficial].

**Data Structures:**
*   **Phase Function Parameters ($\Phi_p$):** The method uses a three-degree proposed phase function with base polynomial $Q_i = \mu_i$ (where $\mu$ is the cosine of the scattering angle), resulting in $\Phi_p = (b_1, b_2, b_3)$ [S1, unofficial].
*   **Intensity Profile:** To reduce computational cost, only a horizontal line of pixels on the symmetric line of the scattering spot is extracted from each captured image and synthesized [S1, unofficial].

**Optimization Objective:**
The parameters are solved via a least-squares analysis-by-synthesis approach. To handle the large dynamic range of scattering intensities (where peaks are much brighter than tails) and avoid overfitting high intensities or amplifying noise in dark pixels, a log-intensity transformation with a bias $\delta$ is used:

$$
\sigma_t^*, \sigma_s^*, \Phi_p^* = \arg \min_{\sigma_t, \sigma_s, \Phi_p} \sum_{i=1}^{K} \left( \log(I_{l_i} + \delta) - \log(I(\sigma_t, \sigma_s, \Phi_p, l_i) + \delta) \right)^2
$$

where $I_{l_i}$ is the captured image intensity, $I(\sigma_t, \sigma_s, \Phi_p, l_i)$ is the synthesized image intensity, and $\delta$ is a positive bias constant randomly alternated during optimization [S1, unofficial].

**Algorithm Steps:**
1.  **Normalization:** Normalize the captured images such that the average intensity of the entire image set is 1, ignoring light source intensity differences between real and synthesized scenes [S1, unofficial].
2.  **Initialization:** Initialize $\sigma_t$, $\sigma_s$, and $\Phi_p = (b_1, b_2, b_3)$. Set the initial sample-per-pixel count to 128 [S1, unofficial].
3.  **Bias Selection:** Randomly select a positive bias value $\delta$ for the current optimization step [S1, unofficial].
4.  **Rendering:** Render the synthesized image $I(\sigma_t, \sigma_s, \Phi_p, l_i)$ for the current light direction $l_i$ using the physics-based differentiable renderer. Extract the horizontal line of pixels on the symmetric line of the scattering spot [S1, unofficial].
5.  **Loss Calculation:** Compute the loss using the log-intensity formula with the selected $\delta$ [S1, unofficial].
6.  **Gradient Update:** Compute the gradient $\nabla_\theta \mathcal{L}$ with respect to the parameters $\theta = (\sigma_t, \sigma_s, \Phi_p)$ using backpropagation through the differentiable renderer. Update parameters using a least-squares optimizer [S1, unofficial].
7.  **Adaptive Sampling:** Monitor the loss. If the loss gets smaller and stable, gradually increase the sample-per-pixel count [S1, unofficial].
8.  **Iteration:** Repeat steps 3–7 until convergence [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_rendering_exponential_phase(captured_images, light_directions, geometry, initial_params, max_iterations=1000):
    # 1. Normalize captured images so average intensity of the whole set is 1
    avg_intensity = sum(numpy.mean(img) for img in captured_images) / len(captured_images)
    normalized_images = [img / avg_intensity for img in captured_images]
    
    # Initialize parameters
    sigma_t, sigma_s, b1, b2, b3 = initial_params
    spp = 128
    prev_loss = float('inf')
    
    for iteration in range(max_iterations):
        # 3. Randomly select a positive bias delta
        delta = random.uniform(0.01, 0.1)
        
        total_loss = 0.0
        gradients = [0.0] * 5
        
        for i in range(len(light_directions)):
            l_i = light_directions[i]
            I_target = normalized_images[i]
            
            # 4. Render synthesized image and extract horizontal line of pixels
            # Physics-based differentiable renderer synthesizes image and computes gradients [S1, unofficial]
            I_synthesized = differentiable_renderer.render_line(
                sigma_t, sigma_s, b1, b2, b3, l_i, geometry, spp
            )
            
            # 5. Compute loss with log-intensity and bias
            log_target = numpy.log(I_target + delta)
            log_synthesized = numpy.log(I_synthesized + delta)
            loss = numpy.sum((log_target - log_synthesized) ** 2)
            total_loss += loss
            
            # 6. Compute gradient via backpropagation through the differentiable renderer
            # and update parameters using a least-squares optimizer
            grad = differentiable_renderer.compute_gradient(log_target, log_synthesized)
            gradients = [g + g_p for g, g_p in zip(gradients, grad)]
        
        # Update parameters
        params = least_squares_optimizer.step([sigma_t, sigma_s, b1, b2, b3], gradients)
        sigma_t, sigma_s, b1, b2, b3 = params
        
        # 7. Adaptive sampling: increase spp when loss gets smaller and stable
        if total_loss < prev_loss * 0.99:
            spp += 32
            prev_loss = total_loss
            
    return sigma_t, sigma_s, b1, b2, b3
```

**Edge Cases and Pitfalls:**
*   **Optically Thin Media:** Optimization fails for optically thin media with large particles (strong forward scattering) because the dynamic ranges between front-lighting and back-lighting images are too different [S1, unofficial].
*   **Optically Thick Media:** Optimization also fails for optically thicker media with stronger $\sigma_t$ [S1, unofficial].
*   **Noise Amplification:** Using log-intensity alone amplifies noise in dark pixels with low signal-to-noise ratio [S1, unofficial].
*   **Overfitting Peaks:** Standard least squares overfits extremely high intensities (peaks) because peak and tail are equally weighted in the sum of squares, despite having different magnitudes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is computationally expensive due to the iterative rendering process. Computational cost is reduced by rendering only a single horizontal line of pixels per image and starting with a low sample count (128 spp), gradually increasing it [S1, unofficial].
*   **Memory Complexity:** Low, as it only stores the intensity profiles of the selected pixel lines and the parameter vectors [G].
*   **Performance:** The proposed three-degree exponential phase function produces better fitting performance than the Two-term Henyey-Greenstein (HG) phase function. For Two-term HG, most errors come from fitting the peak intensities of the data profiles [S1, unofficial].
*   **Strengths:** Robust fitting for complex scattering profiles; handles dynamic range issues via log-intensity and bias; efficient via line-based rendering [S1, unofficial].
*   **Limitations:** Limited to homogeneous slabs with known geometry; fails for extreme optical thicknesses or strong forward scattering; sensitive to noise in dark regions [S1, unofficial].
*   **When to use:** When estimating scattering parameters for translucent materials where standard phase functions (like HG) fail to capture the scattering profile, and the geometry is simple (slab) [S1, unofficial].
*   **When not to use:** For complex scenes, non-homogeneous media, or when the optical thickness is extremely high or low [S1, unofficial].

## Background
This method was proposed by Thanh-Trung Ngo and Hajime Nagahara in 2022 [S1, unofficial]. It is implemented on top of a physics-based differentiable renderer provided by Che et al., which is based on Mitsuba 0.5 [S1, unofficial]. The experimental setup was inspired by previous work on slab estimation [S1, unofficial]. The method was validated through simulation experiments using Mie scattering phase functions (poly-dispersion datasets) with particle diameters of 0.5, 0.3, 0.2, and 0.01 micrometers, $\sigma_t = 2, 6$, fixed volume albedo at 0.9, and slab thickness of 1 mm, rendered in ten light source directions (five front, five back) [S1, unofficial]. Real-world experiments used an Apogee ALTA U260 CCD camera (512x512 resolution), Computar TEC-M55 telecentric lens, Thorlabs MCWHF2 white LED, Thorlabs fiberport collimator on Sigma Koki SGSP-120YAW rotation stage, three color filters (red, green, blue), and materials including 1.2 mm yellow wax, 1.2 mm white rubber eraser, 1.2 mm dry yellow clay, and 2.5 mm cheese slab [S1, unofficial].

## Sources
[S1] A General Scattering Phase Function for Inverse Rendering, https://doi.org/10.48550/arxiv.2209.13875, fetched 2026-10-26 (unofficial)
