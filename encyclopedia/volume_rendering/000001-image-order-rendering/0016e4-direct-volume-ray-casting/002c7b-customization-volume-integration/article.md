# Customization Volume Integration

Also called: Customization Volume Integration in Ray Casting, Position-dependent Parameter Modification

## Overview
Customization Volume Integration is a method that integrates a pre-computed "customization volume" into the volume rendering pipeline to apply position-dependent modifications to visualization parameters, such as gamma correction of intensity values [S1]. It allows for spatially varying adjustments to the data before classification, enabling localized control over rendering appearance without altering the underlying scalar field [S1]. Unlike global transfer function adjustments, this method decouples the spatial distribution of visualization parameters from the input data, allowing for complex, non-uniform effects driven by a secondary volume [S1].

## How it works
The method extends the standard direct volume ray casting pipeline by introducing an additional volume lookup and a parameter application step within the ray marching loop [S1].

### Inputs and Outputs
*   **InputVolume**: The primary scalar volume to be visualized [S1].
*   **CustomizationVolume**: A pre-computed volume containing position-dependent visualization parameters (e.g., gamma values) [S1].
*   **Output**: The final rendered image with position-dependent modifications applied [S1].

### Data Structures
*   **InputVolume**: Standard 3D scalar grid.
*   **CustomizationVolume**: Standard 3D scalar grid. The resolution of this volume can vary (e.g., $32^3$ to $128^3$) but does not significantly impact rendering performance compared to the input volume [S1].
*   **Sampling Position**: 3D coordinates $\mathbf{p}$ along the ray [S1].

### Numbered Steps
1.  **Pass Volumes to Shader**: The `InputVolume` and `CustomizationVolume` are passed to the rendering shader (e.g., GPU ray caster) [S1].
2.  **Ray Marching Loop**: For each sample position $\mathbf{p}$ along the ray:
    a. Fetch the scalar intensity value $I_{raw}$ from `InputVolume` at $\mathbf{p}$ using linear filtering [S1].
    b. Fetch the customization parameter $P$ (e.g., gamma) from `CustomizationVolume` at $\mathbf{p}$ using linear filtering [S1].
    c. Apply the parameter $P$ to $I_{raw}$ to produce the modified intensity $I_{mod}$ [S1].
    d. Apply the transfer function to $I_{mod}$ to obtain color $C$ and opacity $\alpha$ [S1].
    e. Composite the result using front-to-back blending [S1].

### Formulas
The core modification step is defined as:
$$
I_{mod} = f(I_{raw}, P)
$$
In the specific example provided in the source, a gamma correction is applied:
$$
I_{mod} = I_{raw}^{\gamma}
$$
where $\gamma$ is the value fetched from the `CustomizationVolume` [S1].

**Note on Intensity Range**: The example assumes $I_{raw} \in [0, 1]$ to justify the gamma correction [S1].

### Parameters
*   **Customization Volume Resolution**: The source notes that changing the resolution between $32^3$ and $128^3$ has no significant impact on rendering performance [S1]. However, higher resolutions increase computational complexity for pre-computation (e.g., Voronoi interpolation), which may affect interactivity during the customization phase [S1].
*   **Filtering**: Linear filtering is used for fetching values from both volumes [S1].

### Edge Cases and Pitfalls
*   **Performance Impact**: The additional lookups and computations (e.g., power function) reduce overall rendering performance [S1]. The impact depends on the number of samples and the complexity of the customization computation [S1]. However, for large out-of-core volume datasets, the bottleneck is often memory transfer to the GPU, making the customization step's performance impact negligible [S1].
*   **Pre-computation Cost**: If the customization volume is generated using methods like Voronoi interpolation, higher resolutions increase the cost of generating the customization volume, potentially diminishing interactivity when adjusting seed points [S1]. Mitigation involves using lower resolution for interactive customization and higher resolution for final rendering [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Algorithm 3: Pseudocode of a ray-casting loop for volume rendering 
# including the required modifications to support customization volume.
# Input: InputVolume, CustomizationVolume
# Helper: getValue(volume, position) returns the value stored in the volume 
#         at the given position with linear filtering.
# Helper: applyTransFunc(intensity) returns vec4 (color, opacity)

for position from ray.begin to ray.end:
    # Regular code before modification (e.g., ray marching setup)
    
    # 1. Fetch intensity from input volume
    intensity = getValue(InputVolume, position)
    
    # 2. Fetch customization parameter (e.g., gamma) from customization volume
    gamma = getValue(CustomizationVolume, position)
    
    # 3. Apply modification (e.g., gamma correction)
    # Note: intensity is assumed to be in range [0, 1]
    intensity = pow(intensity, gamma)
    
    # 4. Apply transfer function to modified intensity
    color = applyTransFunc(intensity)
    
    # 5. Regular code after modification (e.g., compositing)
    # ...
```

## Subtopics

### Voronoi-based Customization Volume Generation
A method for pre-computing the customization volume using Voronoi diagrams and seed points, allowing for spatially varying parameters [S1].

### Interactive Customization
Adjusting seed points and parameters in real-time, where lower resolution customization volumes are used to maintain interactivity [S1].

## Operating characteristics
*   **Time Complexity**: Adds a constant overhead per sample for an additional texture lookup and a mathematical operation (e.g., power function) [S1].
*   **Memory Complexity**: Requires storage for the `CustomizationVolume` in addition to the `InputVolume` [S1].
*   **Performance**: The performance impact is generally small, especially for large datasets where memory bandwidth is the bottleneck [S1]. Resolution changes in the customization volume ($32^3$ to $128^3$) do not significantly affect rendering speed [S1].
*   **Strengths**: Enables position-dependent visualization parameters without modifying the source data; minor pipeline modifications required [S1].
*   **Limitations**: Requires pre-computation of the customization volume; additional computational cost per sample [S1].

## Background
The method was introduced in the paper "Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering" [S1]. It addresses the need for localized control over visualization parameters in volume rendering, which is not supported by standard global transfer functions [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
