# Customization Volume Integration

Also called: Customization Volume Integration in Ray Casting, Position-dependent Parameter Modification

## Overview
Customization Volume Integration is a method for applying position-dependent modifications to visualization parameters within a volume rendering pipeline, such as ray casting [S1]. It solves the problem of needing spatially varying control over rendering attributes (e.g., gamma correction) without altering the underlying scalar field data [S1]. It differs from sibling methods like the TVAM Integrator by decoupling the parameter field from the transfer function, storing it in a separate auxiliary volume that is sampled alongside the primary data [S1].

## How it works
**Inputs and Outputs:**
*   **InputVolume**: The primary scalar volume to be visualized [S1].
*   **CustomizationVolume**: A pre-computed volume containing position-dependent visualization parameters (e.g., gamma values) [S1].
*   **Output**: The final rendered image with modified intensity values based on the customization volume [S1].

**Data Structures:**
*   **CustomizationVolume**: A 3D grid of floating-point values. The resolution is typically between $32^3$ and $128^3$ [S1]. The values represent the visualization parameter to be applied at each spatial location [S1].
*   **InputVolume**: Standard 3D scalar volume [S1].

**Pseudocode:**
The following pseudocode illustrates the integration into a standard ray-casting loop, specifically demonstrating intensity gamma correction [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Input: InputVolume, CustomizationVolume
# Helper: getValue(volume, position) returns linearly filtered value at position

for position along ray from ray.begin to ray.end:
    # 1. Fetch primary data value
    intensity = getValue(InputVolume, position)
    
    # 2. Fetch customization parameter
    # Linear filtering is used to sample the customization volume
    gamma = getValue(CustomizationVolume, position)
    
    # 3. Apply modification
    # Note: This example assumes intensity is in range [0, 1]
    modified_intensity = pow(intensity, gamma)
    
    # 4. Apply transfer function and accumulate
    color = applyTransFunc(modified_intensity)
    # ... standard ray casting accumulation logic ...
```

**Parameters:**
*   **Intensity Range**: The example implementation assumes input intensity values are normalized to the range $[0, 1]$ to justify the gamma correction operation [S1].
*   **Customization Volume Resolution**: Resolutions between $32^3$ and $128^3$ are used. Higher resolutions increase computational complexity for interpolation but improve spatial accuracy [S1].

**Edge Cases and Pitfalls:**
*   **Performance Impact**: The additional lookups and computations (e.g., power function) reduce overall rendering performance [S1]. The impact scales with the number of samples and the complexity of the customization computation [S1].
*   **Interactivity vs. Quality Trade-off**: For interactive applications involving seed point adjustment (e.g., Voronoi-based customization generation), high-resolution customization volumes diminish interactivity due to increased interpolation complexity [S1]. Mitigation involves using lower resolution for interactive customization and higher resolution for final rendering [S1].
*   **Memory Bottleneck**: For large out-of-core volume datasets, the memory transfer bottleneck to the GPU often renders the performance impact of the customization step negligible [S1].

## Operating characteristics
*   **Time Complexity**: Adds $O(1)$ overhead per sample for the additional lookup and parameter application [S1]. Total time depends on the number of samples and the cost of the specific customization function (e.g., `pow`) [S1].
*   **Memory Complexity**: Requires additional memory proportional to the size of the CustomizationVolume [S1].
*   **Performance**: The performance penalty is generally small compared to the cost of fetching the primary volume data, especially for large datasets where memory bandwidth is the limiting factor [S1].
*   **Strengths**: Allows for spatially varying, position-dependent control of visualization parameters without modifying the source data or the transfer function structure [S1].
*   **Limitations**: Introduces an extra texture lookup and computation per sample, which can impact frame rates in highly interactive scenarios or with very high sample counts [S1].

## Background
The method was introduced in the context of interactive position-dependent customization of transfer function classification parameters [S1]. It enables users to adjust visualization parameters (such as gamma) based on spatial location, often generated via techniques like Voronoi interpolation from seed points [S1]. The approach was presented at the Workshop on Volume Based Computer Graphics (VCBM) in 2015 [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
