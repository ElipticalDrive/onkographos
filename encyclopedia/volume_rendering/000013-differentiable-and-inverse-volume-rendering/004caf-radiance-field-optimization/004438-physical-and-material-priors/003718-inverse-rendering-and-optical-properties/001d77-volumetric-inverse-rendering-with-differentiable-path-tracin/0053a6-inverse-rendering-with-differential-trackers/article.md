# Inverse Rendering with Differential Trackers

Also called: Differential ratio tracking, Unbiased inverse volume rendering

## Overview
Inverse Rendering with Differential Trackers is an unbiased method for estimating gradients with respect to the parameters of scattering volumes (density $\sigma_t$, albedo $\alpha$) to minimize a re-rendering objective function in inverse rendering [S1]. It sets itself apart from standard free-flight sampling by using differential ratio tracking with a majorant of $1.01 \sigma_t$, which avoids the need for a separate medium control density hyperparameter and reduces computation cost [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference images, initial values for medium properties (density $\sigma_t$, albedo $\alpha$, emission $L_e$), camera parameters, illumination conditions, and an isotropic phase function [S1].
*   **Outputs:** Optimized medium properties ($\sigma_t, \alpha$) that minimize the difference between rendered and reference images [S1].

**Data Structures:**
*   **Medium Properties:** Stored on a regular grid with fields for extinction coefficient $\sigma_t$, scattering albedo $\alpha$, and emission $L_e$ [S1, G].
*   **Path State:** Stores the current position, direction, throughput, and accumulated optical depth along the path [G].

**Algorithm Steps:**
1.  **Initialization:** Use a coarse-to-fine approach for initialization unless specified otherwise [S1].
2.  **Stage 1 (Emissive Fit):** Optimize only for emission $L_e$ to fit a nonphysical, emissive volume [S1].
3.  **Stage 2 (Albedo Fit):** Set medium emission $L_e$ to zero and optimize for scattering albedo $\alpha$ such that all color originates from lighting and the reconstructed albedo [S1].
4.  **Gradient Estimation:** Use differential ratio tracking with a majorant of $1.01 \sigma_t$ to compute gradients [S1]. This involves tracing one additional recursive path for each training path [S1].
5.  **Optimization:** Use gradient-based optimization (Adam or SGD with momentum) to update medium properties [S1].

**Formulas:**
*   **Majorant:** $M = 1.01 \sigma_t$ [S1].
*   **Objective:** Minimize a re-rendering objective function over medium parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_rendering_with_differential_trackers(reference_images, initial_sigma_t, initial_alpha, num_iterations=1000, learning_rate=0.01):
    sigma_t = initial_sigma_t.copy()
    alpha = initial_alpha.copy()
    L_e = initial_alpha * sigma_t  # Initial emissive guess [G]
    
    # Stage 1: Fit emissive volume
    for i in range(num_iterations // 2):
        rendered_image = differentiable_volume_render(sigma_t, alpha, L_e)
        loss = mean_squared_error(rendered_image, reference_images)
        grad_L_e = compute_gradient_differential_tracking(sigma_t, alpha, L_e, loss)
        L_e = L_e - learning_rate * grad_L_e
        
    # Stage 2: Optimize albedo
    L_e = 0.0  # Set emission to zero [S1]
    for i in range(num_iterations // 2, num_iterations):
        rendered_image = differentiable_volume_render(sigma_t, alpha, L_e)
        loss = mean_squared_error(rendered_image, reference_images)
        grad_alpha = compute_gradient_differential_tracking(sigma_t, alpha, L_e, loss)
        alpha = alpha - learning_rate * grad_alpha
        
    return sigma_t, alpha

def differentiable_volume_render(sigma_t, alpha, L_e):
    # Standard differentiable volume rendering integration [G]
    image = zeros_like(reference_images)
    for each pixel:
        ray = cast_ray(pixel)
        throughput = 1.0
        radiance = 0.0
        for step in range(64):  # Maximum path length [S1]
            sigma = sample_grid(sigma_t, ray.position)
            alpha_val = sample_grid(alpha, ray.position)
            L = sample_grid(L_e, ray.position)
            s = sample_free_flight(sigma)
            T = exp(-sigma * s)
            radiance += throughput * T * (L + sigma * alpha_val * phase_function(ray.direction))
            throughput *= T
            ray.position += ray.direction * s
        image[pixel] = radiance
    return image

def compute_gradient_differential_tracking(sigma_t, alpha, L_e, loss):
    # Trace primary path with actual density
    primary_path = trace_path(sigma_t, alpha, L_e, max_length=64)
    # Trace recursive path with majorant density for gradient estimation
    majorant_sigma = 1.01 * sigma_t
    recursive_path = trace_path(majorant_sigma, alpha, L_e, max_length=64)
    # Compute gradient using path replay backpropagation ratio
    gradient = path_replay_backpropagation(primary_path, recursive_path, loss)
    return gradient

def trace_path(sigma, alpha, L_e, max_length):
    # Standard differentiable path tracing loop [G]
    path = []
    throughput = 1.0
    for _ in range(max_length):
        sigma_t_val = sample_grid(sigma, current_position)
        s = sample_free_flight(sigma_t_val)
        T = exp(-sigma_t_val * s)
        path.append({'s': s, 'T': T, 'throughput': throughput, 'sigma': sigma_t_val})
        throughput *= T
        if throughput < 1e-4: break
    return path

def path_replay_backpropagation(primary_path, recursive_path, loss):
    # PRB gradient computation using ratio of PDFs [G]
    gradient = zeros_like(sigma)
    for step in range(len(primary_path)):
        p = primary_path[step]
        r = recursive_path[step]
        ratio = p['sigma'] / r['sigma']  # Differential ratio tracking weight
        gradient += loss * ratio * p['throughput']
    return gradient
```

**Parameters:**
*   **Maximum path length:** 64 vertices [S1].
*   **Majorant factor:** 1.01 [S1].
*   **Optimizer:** Adam or SGD with momentum [S1].

**Edge Cases and Pitfalls:**
*   **High gradient variance:** Free-flight sampling gradients are prone to large outliers, causing persistent thin 'haze' artifacts around reconstructed volumes when using Adam optimizer [S1].
*   **Defensive sampling:** Eliminates large outliers but does not sufficiently reduce overall variance, exposing similar haze artifacts [S1].
*   **Optimizer choice:** SGD fails to converge due to gradient outliers, filling empty regions with artifacts [S1]. Adam suppresses these outliers via adaptive step sizes, allowing convergence despite high variance [S1].

## Operating characteristics
*   **Time Complexity:** Linear time complexity in path length [S1].
*   **Memory Complexity:** Constant memory complexity [S1].
*   **Performance:** The recursive path amounts to an overhead of ~20% compared to pure free-flight sampling in the path replay backpropagation framework [S1].
*   **Strengths:** Unbiased gradient estimation, no need for medium control density hyperparameter, efficient parallel optimization of millions of parameters [S1].
*   **Limitations:** High gradient variance can lead to artifacts if not handled by appropriate optimizer (Adam) or initialization scheme [S1].
*   **When to use:** When unbiased gradient estimation is required for inverse rendering of participating media, and computational resources allow for the ~20% overhead of recursive path tracing [S1].
*   **When not to use:** When real-world, imperfect data is used without domain-specific assumptions, as the method focuses on synthetic scenes with known parameters [S1].

## Background
*   **Origin:** Based on Path Replay Backpropagation framework by Vicini et al. [2021b] [S1].
*   **History:** Developed to address severe issues in existing gradient estimation methods, which led to biased and high-variance gradients [S1].
*   **Usage:** Implemented in Mitsuba 2 [Nimier-David et al. 2019] [S1].

## Sources
[S1] Unbiased inverse volume rendering with differential trackers, https://doi.org/10.1145/3528223.3530073, fetched 2026-11-01
