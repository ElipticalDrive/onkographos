# Volume Raycasting Pipeline

Also called: Direct Volume Raycasting Pipeline, Image-Order Volume Rendering Pipeline, Raycasting Pipeline

## Overview
The Volume Raycasting Pipeline is the standard procedural framework for direct volume rendering, calculating the volume rendering integral for every viewing ray originating at the image plane [S1, unofficial]. It advances rays through the volume with regular increments, performing reconstruction, classification, shading, and compositing at each sample location [S1, unofficial]. It is distinguished from its siblings by its explicit separation of these four stages and the critical decision of whether classification and shading occur before or after reconstruction (pre- vs. post-), where post-classification and post-shading are preferred to avoid severe artifacts caused by non-linear reconstruction [S1, unofficial].

## How it works
The method evaluates the volume rendering integral by advancing rays through the volume with regular increments (the object sample distance) [S1, unofficial]. At each resample location along the ray, the pipeline executes four sequential steps: Reconstruction, Classification, Shading, and Compositing [S1, unofficial]. The pipeline supports four variants based on the ordering of Classification and Shading relative to Reconstruction:
1.  Pre-classification, Pre-shading
2.  Pre-classification, Post-shading
3.  Post-classification, Pre-shading
4.  Post-classification, Post-shading

The recommended configuration uses **Post-classification** and **Post-shading** [S1, unofficial].

### Inputs and Outputs
*   **Input:** A discrete scalar dataset $D$ defined on a 3D grid; a viewing ray $R(t)$ defined by an origin and direction; a transfer function $T(v)$ mapping scalar values to color and opacity; an illumination model $L(g)$ mapping gradients to shading parameters.
*   **Output:** A final color value $C_{final}$ for each pixel.

### Data Structures
*   **Dataset $D$:** A 3D array of scalar values. Layout is implementation-dependent but typically stored in row-major or column-major order.
*   **Ray $R(t)$:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, parameterized by distance $t$.
*   **Sample Point:** A position $\mathbf{p}$ along the ray where data is sampled.

### Numbered Steps
1.  **Initialize Ray:** For each pixel, generate a viewing ray $R(t)$ originating from the image plane and passing through the volume.
2.  **Advance Ray:** Move along the ray with a regular increment $\Delta t$ (object sample distance). Let the current position be $\mathbf{p}$.
3.  **Reconstruction:** Construct a continuous function from the discrete dataset $D$ at position $\mathbf{p}$.
    *   If **Pre-classification/Pre-shading**: Sample the discrete data $D$ at $\mathbf{p}$ to get value $v$.
    *   If **Post-classification/Post-shading**: Apply a reconstruction filter to the original sample values around $\mathbf{p}$ to estimate the continuous function value $v_{rec}$ and gradient $g_{rec}$ at $\mathbf{p}$ [S1, unofficial].
4.  **Classification:**
    *   If **Pre-classification**: Assign color $c$ and opacity $\alpha$ to the sampled value $v$ using transfer function $T$: $(c, \alpha) = T(v)$.
    *   If **Post-classification**: Classify the reconstructed value $v_{rec}$: $(c, \alpha) = T(v_{rec})$ [S1, unofficial].
5.  **Shading:**
    *   If **Pre-shading**: Evaluate the illumination model $L$ at the grid points only (or use pre-computed shading values) [S1, unofficial].
    *   If **Post-shading**: Reconstruct the gradient $g_{rec}$ at $\mathbf{p}$ and evaluate the illumination model $L(g_{rec})$ to determine shading parameters (e.g., diffuse color, specular highlights) [S1, unofficial].
6.  **Compositing:** Determine the contribution of the classified and shaded sample to the final image. This typically involves front-to-back or back-to-front alpha compositing, blending the current sample's color and opacity with the accumulated result from previous samples along the ray.
7.  **Repeat:** Continue steps 2-6 until the ray exits the volume or a termination condition is met.
8.  **Finalize:** Output the accumulated color for the pixel.

### Formulas
*   **Volume Rendering Integral:** The goal is to approximate $\int_{t_0}^{t_1} c(t) e^{-\int_{t_0}^{t} \alpha(\tau) d\tau} dt$ [S1, unofficial].
*   **Pre-classification:** $(c, \alpha) = T(D(\mathbf{p}))$ [S1, unofficial].
*   **Post-classification:** $(c, \alpha) = T(\text{Reconstruct}(D, \mathbf{p}))$ [S1, unofficial].
*   **Pre-shading:** $L$ evaluated at grid points [S1, unofficial].
*   **Post-shading:** $L(\text{ReconstructGradient}(D, \mathbf{p}))$ [S1, unofficial].

### Parameters
*   **Object sample distance ($\Delta t$):** The regular increment for advancing rays. Not given in the sources.
*   **Reconstruction filter:** The specific filter used to construct the continuous function. Not given in the sources.
*   **Transfer function $T(v)$:** Maps scalar values to color and opacity. Not given in the sources.
*   **Illumination model $L(g)$:** Evaluates lighting based on gradients. Not given in the sources.

### Edge Cases and Pitfalls
*   **Non-commutativity:** Pre-classification and pre-shading cause severe artifacts because reconstruction is usually non-linear and does not commute with transfer functions or lighting models [S1, unofficial].
*   **Precision:** Using floating-point precision throughout the pipeline is recommended to minimize artifacts [S1, unofficial].
*   **Boundary Conditions:** Ray traversal must handle entry and exit from the volume correctly. Not given in the sources.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ray_casting_pipeline(dataset, ray, transfer_function, illumination_model, reconstruction_filter):
    """
    Executes the Volume Raycasting Pipeline for a single ray.
    
    Args:
        dataset: 3D array of scalar values.
        ray: Object with origin, direction, and method to advance.
        transfer_function: Function mapping scalar value to (color, opacity).
        illumination_model: Function mapping gradient to shading parameters.
        reconstruction_filter: Function to reconstruct continuous value and gradient from dataset.
        
    Returns:
        Final color for the pixel.
    """
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    transmittance = 1.0
    
    # Advance ray through volume with regular increments
    # Note: Exact traversal logic (e.g., DDA) is implementation-dependent [G]
    for sample_position in ray.traverse_through_volume():
        
        # Step 1: Reconstruction
        # Post-classification and post-shading are used [S1, unofficial]
        reconstructed_value, reconstructed_gradient = reconstruction_filter(dataset, sample_position)
        
        # Step 2: Classification
        color, opacity = transfer_function(reconstructed_value)
        
        # Step 3: Shading
        shading_params = illumination_model(reconstructed_gradient)
        
        # Apply shading to color (simplified)
        shaded_color = apply_shading(color, shading_params)
        
        # Step 4: Compositing (Front-to-Back)
        # Contribution = shaded_color * opacity * transmittance
        contribution = shaded_color * opacity * transmittance
        accumulated_color += contribution
        
        # Update transmittance
        transmittance *= (1.0 - opacity)
        
        # Early termination if fully opaque
        if transmittance < 0.001:
            break
            
    return accumulated_color
```

## Subtopics

### Reconstruction
The process of constructing a continuous function from the dataset to allow sampling at arbitrary positions along the ray. It is fundamental to post-classification and post-shading [S1, unofficial].

### Classification
The process of assigning color and opacity to data values using a transfer function. Post-classification is preferred to avoid artifacts from non-linear reconstruction [S1, unofficial].

### Shading
The evaluation of the illumination model. Post-shading reconstructs gradients at every resample location for better visual quality [S1, unofficial].

### Compositing
The process of blending sample contributions into the final image. Typically uses front-to-back alpha compositing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels). Not given in the sources.
*   **Memory Complexity:** Depends on the dataset size and the storage of intermediate values per sample. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Flexible pipeline allowing different combinations of classification and shading orders; post-classification and post-shading provide high visual quality [S1, unofficial].
*   **Limitations:** Pre-classification and pre-shading cause severe artifacts [S1, unofficial]. Performance can be high due to per-sample reconstruction and shading calculations [S1, unofficial].
*   **When to use:** When high-quality volume rendering is required and computational resources allow for post-classification and post-shading [S1, unofficial].
*   **When not to use:** When performance is critical and visual artifacts from pre-classification/pre-shading are acceptable, or when the reconstruction filter is not available [S1, unofficial].

## Background
The Volume Raycasting Pipeline was described in a master's thesis by [Author not specified in source], titled "The Raycasting Pipeline" [S1, unofficial]. The work compares four variants of the pipeline based on the ordering of classification and shading relative to reconstruction, demonstrating that post-classification and post-shading yield the best results [S1, unofficial].

## Sources
[S1] The Raycasting Pipeline, https://users.cg.tuwien.ac.at/bruckner/homepage/content/mastersthesis/html/node29.html, fetched 2026-10-29 (unofficial)
