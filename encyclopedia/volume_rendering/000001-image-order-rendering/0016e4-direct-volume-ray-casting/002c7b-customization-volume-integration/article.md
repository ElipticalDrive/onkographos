# Customization Volume Integration

Also called: Customization Volume Integration in Ray Casting, Position-dependent Parameter Modification

## Overview
Customization Volume Integration is a method that integrates a pre-computed "customization volume" into the volume rendering pipeline to apply position-dependent modifications to visualization parameters, such as gamma correction of intensity values [S1]. It allows spatially varying adjustments to the data before classification, enabling localized control over rendering appearance without altering the underlying scalar field [S1]. Unlike global transfer function adjustments, this method decouples the modification logic from the ray-casting loop's core compositing, allowing for complex, spatially coherent parameter fields (e.g., Voronoi-based seeds) to drive the visualization [S1].

## How it works
**Inputs and Outputs:**
*   **InputVolume**: The primary scalar volume dataset to be visualized.
*   **CustomizationVolume**: A pre-computed volume containing position-dependent visualization parameters (e.g., gamma values).
*   **Output**: The final composited image with position-dependent parameter modifications applied.

**Data Structures:**
*   **InputVolume**: Typically a 3D grid of scalar values.
*   **CustomizationVolume**: A 3D grid of scalar values representing the parameter to be applied (e.g., float values for gamma). The resolution can vary (e.g., $32^3$ to $128^3$) [S1].
*   **getValue(volume, position)**: A helper function that retrieves the value from a volume at a given 3D position using linear filtering [S1].

**Numbered Steps:**
1.  **Pass Volumes**: The `CustomizationVolume` is passed to the shader performing the volume rendering (e.g., ray-casting) alongside the `InputVolume` [S1].
2.  **Ray Casting Loop**: For each sample position along the ray:
    a. Fetch the intensity value from the `InputVolume` at the current sampling position [S1].
    b. Fetch the visualization parameter (e.g., gamma) from the `CustomizationVolume` at the same sampling position using linear filtering [S1].
    c. Apply the fetched parameter to the intensity value. For example, if the parameter is gamma, compute $intensity_{modified} = pow(intensity, gamma)$ [S1]. Note: The intensity values are assumed to be in the range $[0, 1]$ for this operation [S1].
    d. Apply the transfer function to the modified intensity to obtain color and opacity [S1].
    e. Composite the result using the standard front-to-back accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_with_customization(InputVolume, CustomizationVolume, ray_begin, ray_end, transfer_function):
    """
    Performs volume rendering with position-dependent customization.
    
    Args:
        InputVolume: The primary scalar volume.
        CustomizationVolume: The pre-computed volume of parameters (e.g., gamma).
        ray_begin: Start t parameter of the ray segment.
        ray_end: End t parameter of the ray segment.
        transfer_function: Function mapping intensity to (color, opacity).
        
    Returns:
        Final color and opacity for the pixel.
    """
    # Standard initialization
    acc_color = vec3(0.0, 0.0, 0.0)
    acc_opacity = 0.0
    
    # Discretize the ray segment
    num_samples = 64 # Example value, depends on implementation
    delta_t = (ray_end - ray_begin) / num_samples
    
    for i in range(num_samples):
        t = ray_begin + i * delta_t
        position = ray_origin + t * ray_direction # Assuming ray_origin and ray_direction are available
        
        # 1. Fetch intensity from input volume
        intensity = get_value(InputVolume, position)
        
        # 2. Fetch customization parameter (e.g., gamma) from customization volume
        gamma = get_value(CustomizationVolume, position)
        
        # 3. Apply modification (e.g., gamma correction)
        # Note: intensity is assumed to be in [0, 1]
        intensity = pow(intensity, gamma)
        
        # 4. Apply transfer function
        color, opacity = transfer_function(intensity)
        
        # 5. Front-to-back compositing
        # alpha_term = opacity * (1 - acc_opacity)
        # acc_color += alpha_term * color
        # acc_opacity += alpha_term
        
        # Simplified iterative update (assuming opacity is already step-size corrected)
        acc_color = acc_color + (1.0 - acc_opacity) * color * opacity
        acc_opacity = acc_opacity + (1.0 - acc_opacity) * opacity
        
        # Early ray termination optimization (optional)
        if acc_opacity > 0.99:
            break
            
    return acc_color, acc_opacity

def get_value(volume, position):
    """
    Retrieves value from volume at position using linear filtering.
    This is a placeholder for the actual interpolation logic.
    """
    # Implementation depends on hardware (e.g., texture lookup with linear filtering)
    # [G] In a real implementation, this would involve coordinate transformation
    # and trilinear interpolation of the volume texture.
    pass
```

**Parameters:**
*   **CustomizationVolume Resolution**: Ranges from $32^3$ to $128^3$ [S1]. Higher resolution increases computational complexity for interpolation but provides smoother parameter fields [S1].
*   **Intensity Range**: The input intensity values should be normalized to $[0, 1]$ for operations like gamma correction [S1].

**Edge Cases and Pitfalls:**
*   **Performance Impact**: Additional lookups and computations (e.g., power function) reduce rendering performance [S1]. The impact depends on the number of samples and the complexity of the customization computation [S1].
*   **Resolution Trade-off**: Changing the customization volume resolution between $32^3$ and $128^3$ has no significant impact on rendering performance, but higher resolution increases computational complexity for Voronoi/interpolation, diminishing interactivity when adjusting seed points [S1]. Mitigation: Use lower resolution for interactive customization and higher resolution for actual rendering [S1].
*   **Memory Bottleneck**: For large out-of-core volume datasets, the bottleneck is memory transfer to the GPU, making the performance impact of the customization step negligible [S1].

## Operating characteristics
*   **Time Complexity**: Adds $O(1)$ overhead per sample for the additional lookup and parameter application [S1]. Total time is $O(N)$ where $N$ is the number of samples per ray [S1].
*   **Memory Complexity**: Requires storing an additional volume of size $R^3$ where $R$ is the resolution of the customization volume [S1].
*   **Performance**: The performance impact is generally small, especially for large datasets where memory bandwidth is the bottleneck [S1]. However, complex customization functions (e.g., non-linear interpolations) can degrade interactivity [S1].
*   **Strengths**: Allows spatially varying, position-dependent control over visualization parameters without modifying the core ray-casting algorithm significantly [S1]. Easy to integrate into existing pipelines [S1].
*   **Limitations**: Requires pre-computation of the customization volume (e.g., via Voronoi tessellation) [S1]. Performance can be affected by the complexity of the customization function and the resolution of the customization volume [S1].
*   **When to Use**: When spatially varying adjustments to visualization parameters (like gamma, opacity scaling, or color mapping) are needed, and the parameter field can be pre-computed or interactively adjusted with lower resolution [S1].
*   **When Not to Use**: When real-time, per-sample dynamic parameter generation is required without pre-computation, or when the customization function is extremely complex and performance is critical [S1].

## Background
*   **Origin**: The method was introduced in the paper "Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering" [S1].
*   **History**: The technique was developed to address the need for localized control over volume rendering parameters, allowing users to adjust visualization aspects (like gamma correction) in specific regions of the volume without affecting the entire dataset [S1].
*   **Usage**: It is used in interactive volume rendering systems where users need to fine-tune the appearance of specific structures or regions within a volume dataset [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
