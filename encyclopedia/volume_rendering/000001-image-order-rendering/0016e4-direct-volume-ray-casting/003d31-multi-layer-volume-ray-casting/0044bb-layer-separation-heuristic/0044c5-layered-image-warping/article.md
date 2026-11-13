# Layered Image Warping

Also called: Feature Peeling-based Layered Warping

## Overview
Layered Image Warping is a technique for transparent volume rendering that overcomes the limitations of single-image warping by representing the inner structure of an object as multiple partial images, called layers [S1]. It solves the problem of preserving depth cues and handling view-dependent lighting in transparent volumes, which cannot be shaded correctly using a single map of normal vectors [S1]. Unlike its sibling methods that may use uniform or heuristic-based splitting, this method groups volume elements by distance from the camera and similar opacity to ensure consistent shading [S1].

## How it works
**Inputs:** A volume dataset, a ray path through the volume, and a smoothing kernel for opacity.
**Outputs:** A set of layer images, each associated with a depth value for warping.

**Data Structures:**
*   **Layer:** A 2D image buffer storing color and depth values for a specific subset of the volume.
*   **Ray Profile:** A 1D array of opacity values sampled along the ray, smoothed to detect features.

**Algorithm Steps:**

1.  **Smoothing:** Smooth the opacity values along the ray to reduce noise. Let $o_i$ be the opacity at sample $i$. The smoothed opacity $\hat{o}_i$ is computed using a smoothing kernel [S1].
2.  **Feature Detection:** Detect local minima and maxima of the smoothed opacity $\hat{o}_i$ along the ray [S1].
3.  **Layer Separation (Feature Peeling):** Use the detected extrema to separate the ray into segments (layers) [S1].
    *   **Noise Suppression:** Suppress layers that have a total opacity of less than 10%. This threshold prevents the creation of layers from noise [S1].
    *   **Feature Preservation:** Avoid separating connected, highly opaque regions, as they likely belong to the same feature [S1].
4.  **Depth Value Selection:** For each pixel in a layer image, determine a depth value representing the ray interval. Two methods are evaluated to locate the "main effect" of light attenuation within the layer's interval:
    *   **Method A (Max Opacity):** Select the point with the maximum opacity after classification within the layer's segment [S1].
    *   **Method B (Max Color Impact):** Select the point with the maximum impact on color, calculated as the voxel's opacity multiplied by the remaining opacity along the ray segment [S1]. Method B is preferred for slight advantages [S1].
5.  **Rendering:** Render each layer independently. The depth value associated with each pixel allows for correct warping and compositing of the layers [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def layered_image_warping(ray_samples, smoothing_kernel, min_opacity_threshold=0.1):
    """
    ray_samples: list of tuples (depth, opacity, color) along the ray.
    smoothing_kernel: function to smooth opacity values.
    min_opacity_threshold: minimum opacity for a layer to be valid (default 0.1).
    """
    # Step 1: Smooth opacity
    opacities = [s[1] for s in ray_samples]
    smoothed_opacities = smoothing_kernel(opacities)
    
    # Step 2: Detect local minima and maxima
    extrema_indices = find_local_extrema(smoothed_opacities)
    
    # Step 3: Separate layers
    layers = []
    current_layer_start = 0
    
    for idx in extrema_indices:
        # Define potential layer segment
        layer_segment = ray_samples[current_layer_start:idx+1]
        
        # Calculate total opacity of the segment
        total_opacity = sum(s[1] for s in layer_segment)
        
        # Check noise suppression
        if total_opacity >= min_opacity_threshold:
            # Check feature preservation (avoid splitting highly opaque connected regions)
            # This is a heuristic check; if the segment is highly opaque and connected,
            # it might be merged with the next segment. For simplicity, we assume
            # the extrema detection already handles major features, and we just
            # filter by opacity.
            
            # Step 4: Depth Value Selection (Method B: Max Color Impact)
            # Calculate remaining opacity for each sample in the segment
            # Remaining opacity is the product of opacities from the start of the ray
            # up to the current sample.
            remaining_opacities = []
            cumulative_opacity = 1.0
            for s in ray_samples:
                remaining_opacities.append(cumulative_opacity)
                cumulative_opacity *= (1.0 - s[1])
            
            max_impact = -1.0
            best_depth = None
            
            for i, s in enumerate(layer_segment):
                # Impact = voxel opacity * remaining opacity
                # Note: remaining opacity should be calculated relative to the segment start
                # or global ray start? The source says "remaining opacity along the ray segment".
                # Assuming it means the attenuation *before* this voxel in the segment.
                # Let's use global remaining opacity for consistency with "light attenuation".
                impact = s[1] * remaining_opacities[ray_samples.index(s)]
                
                if impact > max_impact:
                    max_impact = impact
                    best_depth = s[0]
            
            if best_depth is None:
                best_depth = layer_segment[0][0] # Fallback
            
            layers.append({
                'segment': layer_segment,
                'depth': best_depth,
                'color': compute_layer_color(layer_segment)
            })
        
        current_layer_start = idx + 1
    
    # Handle the last segment
    if current_layer_start < len(ray_samples):
        layer_segment = ray_samples[current_layer_start:]
        total_opacity = sum(s[1] for s in layer_segment)
        if total_opacity >= min_opacity_threshold:
            # Same depth selection logic
            max_impact = -1.0
            best_depth = None
            for s in layer_segment:
                impact = s[1] * remaining_opacities[ray_samples.index(s)]
                if impact > max_impact:
                    max_impact = impact
                    best_depth = s[0]
            if best_depth is None:
                best_depth = layer_segment[0][0]
            layers.append({
                'segment': layer_segment,
                'depth': best_depth,
                'color': compute_layer_color(layer_segment)
            })
            
    return layers

def compute_layer_color(segment):
    # Standard volume rendering integration for the segment
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    for s in segment:
        c, o = s[2], s[1]
        color = color + c * o * (1.0 - alpha)
        alpha = alpha + o * (1.0 - alpha)
    return color
```

**Parameters:**
*   **Minimum Opacity Threshold:** 10% (0.1). Layers with less than this opacity are suppressed to reduce noise [S1].

**Edge Cases and Pitfalls:**
*   **Noise:** Without smoothing and the 10% threshold, noise can create spurious layers [S1].
*   **Connected Opaque Regions:** Highly opaque connected regions should not be split, as they belong to the same feature [S1].
*   **Depth Selection:** Using entrance or exit depth can fail to capture the main effect of light attenuation, which may occur in the middle of the ray interval [S1].

## Operating characteristics
*   **Performance:** On a volumetric foot dataset (VIX [S1]), increasing the number of layers from 2 to 9 increases the average absolute pixel error (visual quality degradation) from ~0.2 to ~0.6, but also increases the framerate from ~6 fps to ~9 fps [S1].
*   **Strengths:** Provides efficient cues about the inner structure of an object and handles view-dependent lighting better than single-image warping [S1].
*   **Limitations:** Increasing layers degrades visual quality (higher pixel error) [S1]. Requires deferred shading for correct view-dependent lighting [S1].

## Background
*   **Origin:** The method is based on the idea of feature peeling [MMG07] and was introduced in the context of Multi-Frame Rate Volume Rendering [S1].
*   **History:** It was developed to address the limitations of single image warping for transparent volumes [S1].
*   **Usage:** Used in multi-frame rate volume rendering to improve performance while maintaining visual quality [S1].

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
