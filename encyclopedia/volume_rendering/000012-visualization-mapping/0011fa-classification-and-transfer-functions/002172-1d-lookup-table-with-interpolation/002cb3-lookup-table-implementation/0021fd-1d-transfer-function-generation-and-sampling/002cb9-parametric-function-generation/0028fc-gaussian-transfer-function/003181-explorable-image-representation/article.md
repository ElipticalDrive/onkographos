# Explorable Image Representation

Also called: Image-based volume exploration, Multi-layer explorable images

## Overview
Explorable Image Representation is a method for interactive volume visualization that pre-renders scalar data into a set of multi-layered 2D images, allowing users to modify layer transparency and color without re-rendering the 3D scene [S1, unofficial]. It differs from standard ray-casting by decoupling interaction from the 3D data, enabling image-based exploration on devices with limited computational resources [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume data; user-defined ranges of interest; number of layers to explore.
*   **Output:** A set of 2D explorable images; metadata describing layer order, approximate color, and opacity for each layer [S1, unofficial].

**Data Structures:**
*   **Explorable Image:** A 2D raster image (RGBA) containing superimposed layers.
*   **Layer Metadata:** A list of objects, each containing:
    *   `layer_id`: Identifier for the material/region.
    *   `approx_color`: RGB estimate used for separation during generation.
    *   `approx_opacity`: Initial opacity estimate.
    *   `bounds`: Information defining the spatial extent of the layer within the volume [S1, unofficial].

**Algorithm Steps:**

1.  **Transfer Function Generation:**
    *   Compute peaks in the transfer function to identify material boundaries (iso-values) [S1, unofficial].
    *   Select peak centers to be as equally spaced as possible within the user-defined range [S1, unofficial].
    *   Construct a transfer function consisting of Gaussian curves centered on these peaks. The width, height, and number of peaks determine the granularity of the layers [S1, unofficial].
    *   *Note:* The initial transfer function is not optimized for final visualization because it serves as a basis for separable layer extraction [S1, unofficial].

2.  **Layer Separation and Image Generation:**
    *   Assume a rough estimate of each layer’s color to ensure separability during rendering [S1, unofficial].
    *   Render the volume using the generated transfer function.
    *   Apply boundary blurring control if specified by the user; otherwise, use a default value [S1, unofficial].

3.  **Layer Combination Scheme:**
    *   Generate a minimal set of images that allows reconstruction of any layer combination via compositing [S1, unofficial].
    *   **Image 1:** Contains the innermost and outermost layers.
    *   **Image 2:** Contains the innermost, outermost, and the middle layer.
    *   **Subsequent Images:** Follow the pattern where each new image adds the next intermediate layer, bounded by the previously established inner and outer layers. This allows data in between to be constructed from bounding layer information [S1, unofficial].

4.  **Interaction:**
    *   Users modify transparency and color of individual layers by applying image-based operations (e.g., alpha blending, color multiplication) to the pre-rendered explorable images [S1, unofficial].
    *   No 3D data access or additional rendering passes are required during interaction [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_explorable_images(volume_data, user_ranges, num_layers):
    # Step 1: Generate Transfer Function
    # Compute peaks in the transfer function to find material boundaries
    peaks = compute_transfer_function_peaks(volume_data, user_ranges)
    
    # Select peaks to be equally spaced
    selected_peaks = select_equally_spaced_peaks(peaks, num_layers)
    
    # Create Gaussian transfer function centered on selected peaks
    transfer_function = create_gaussian_transfer_function(selected_peaks)
    
    # Step 2: Render and Separate Layers
    # Assume rough color estimates for separation
    layer_colors = estimate_layer_colors(volume_data, selected_peaks)
    
    # Render volume with boundary blurring (default if not specified)
    blurred_volume = apply_boundary_blurring(volume_data, default_blur)
    
    # Render using the transfer function to get initial composite
    # Note: The method implies extracting layers from the rendered result
    # or rendering layers separately and combining them according to the scheme.
    # Based on the text, we generate a set of multi-layered images.
    
    images = []
    metadata = []
    
    # Step 3: Layer Combination Scheme
    # Determine layer indices from innermost to outermost
    layer_indices = get_layer_indices(selected_peaks)
    
    # Generate Image 1: Innermost and Outermost
    if len(layer_indices) >= 2:
        inner_idx = layer_indices[0]
        outer_idx = layer_indices[-1]
        
        # Render or extract these specific layers
        img_1 = render_layers(blurred_volume, [inner_idx, outer_idx], layer_colors)
        images.append(img_1)
        
        # Store metadata
        metadata.append({
            'layers': [inner_idx, outer_idx],
            'colors': [layer_colors[inner_idx], layer_colors[outer_idx]],
            'bounds': get_layer_bounds(volume_data, [inner_idx, outer_idx])
        })
        
        # Generate subsequent images adding intermediate layers
        for i in range(1, len(layer_indices) - 1):
            mid_idx = layer_indices[i]
            # Current image contains inner, outer, and mid
            current_layers = [inner_idx, mid_idx, outer_idx]
            img = render_layers(blurred_volume, current_layers, layer_colors)
            images.append(img)
            
            metadata.append({
                'layers': current_layers,
                'colors': [layer_colors[l] for l in current_layers],
                'bounds': get_layer_bounds(volume_data, current_layers)
            })
            
    return images, metadata

def interact_with_explorable_image(image, metadata, user_color_changes, user_opacity_changes):
    # Image-based interaction
    # Apply user-defined color and opacity changes to the pre-rendered image
    # using the stored metadata to identify layers
    result_image = image
    
    for layer_id in metadata['layers']:
        if layer_id in user_color_changes:
            result_image = apply_color_change(result_image, layer_id, user_color_changes[layer_id])
        if layer_id in user_opacity_changes:
            result_image = apply_opacity_change(result_image, layer_id, user_opacity_changes[layer_id])
            
    return result_image
```

**Parameters:**
*   **Boundary Blurring:** Controls the amount of blurring at material boundaries. Default value is used if not specified [S1, unofficial].
*   **Number of Layers:** Controls the number of generated input images. Users can select a range from a single image (entire data) to individual images for each layer [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Storage/Bandwidth Trade-off:** There is a trade-off between storage space/bandwidth requirements and the ability to reproduce all possible interactions between layers [S1, unofficial].
*   **Initial Optimization:** The initial transfer function is not optimized for visualization because it allows users to change color and opacity of individual layers [S1, unofficial].
*   **Color Approximation:** The method assumes a rough estimate of each layer’s color for separating input images. If colors are too similar, separation may fail [S1, unofficial].

## Subtopics

### Gaussian Transfer Function
Uses Gaussian bumps to define opacity and color based on scalar value position. Provides the basis for the transfer function generation in Explorable Image Representation. [Gaussian Transfer Function](4408-gaussian-transfer-function.md)

### Parametric Function Generation
Generates transfer functions using parametric curves (like Gaussians). Explorable Image Representation uses this to create the initial multi-layered transfer function. [Parametric Function Generation](4407-parametric-function-generation.md)

## Operating characteristics
*   **Time Complexity:** Pre-rendering time depends on the volume size and the number of layers. Interaction time is constant relative to the volume size, as it operates on 2D images [S1, unofficial].
*   **Memory Complexity:** Storage requirements increase with the number of generated explorable images. The method trades off storage/bandwidth for interactive flexibility [S1, unofficial].
*   **Strengths:** Enables image-based interaction without 3D data access or re-rendering passes. Suitable for mobile devices or client-server architectures with limited bandwidth [S1, unofficial].
*   **Limitations:** Initial transfer function is not optimized for final visualization. Relies on color approximation for layer separation. Trade-off between storage and interaction fidelity [S1, unofficial].
*   **When to Use:** When interactive exploration of layer properties is required on devices with limited rendering capabilities, or when bandwidth is constrained.
*   **When Not to Use:** When high-fidelity, optimized visualization is the primary goal, or when the number of layers is very large, leading to excessive storage requirements [S1, unofficial].

## Background
The Explorable Image Representation method was proposed by Tikhonova et al. in the context of volumetric medical images visualization on mobile devices [S1, unofficial]. It addresses the challenge of interactive volume exploration on resource-constrained devices by pre-computing explorable images. The method is part of a broader effort to optimize volume rendering for mobile and web-based applications [S1, unofficial].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
