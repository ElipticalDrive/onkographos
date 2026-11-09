# Parametric Function Generation
## Overview
Parametric Function Generation constructs transfer functions by evaluating mathematical functions of a scalar input to determine RGBA output values, rather than relying on piecewise linear interpolation between discrete control points [G]. It solves the problem of defining smooth, continuous, and often perceptually optimized color mappings with a compact set of parameters [G]. It differs from its siblings by using explicit analytical formulas (such as cosines, Gaussians, or sigmoids) to define the color space trajectory, offering a global, oscillating, or localized basis for color transitions instead of the local linear segments of [Voxel Color Mapping](2970-voxel-color-mapping.md) or the fixed palettes of [Colormap LUT Generation](7229-colormap-lut-generation.md) [G].

## Shared principles
The family shares the mechanism of mapping a normalized scalar input $t \in [0, 1]$ to an RGBA vector $\mathbf{C}$ via a parametric equation [G]. The input $t$ is typically derived from volume data values normalized to the domain $[0, 1]$ by the parent method [1D Transfer Function Generation and Sampling](2689-1d-transfer-function-generation-and-sampling.md) [G].

The core mathematical structure for the RGB channels is generally:
$$ \mathbf{C}_{rgb} = f(t; \mathbf{p}) $$
where $\mathbf{p}$ represents a set of parameters (vectors or scalars) that control the shape, amplitude, frequency, and phase of the function [G]. The alpha channel $A$ is often treated separately or derived from the same parametric logic, though in some variants (like Cosine Palette) it is constant or handled by a separate function [G].

Key shared implementation details include:
1. **Normalization**: The input scalar must be clamped or normalized to $[0, 1]$ to ensure the parametric function behaves predictably within its defined domain [G].
2. **Clamping**: The resulting RGB values are clamped to $[0, 1]$ to handle numerical precision issues or out-of-bounds results from the parametric evaluation [G].

## Subtopics

### [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md)
- **What it is**: Generates smooth, perceptually distinct color gradients from a scalar input using a parametric cosine function with specific vector parameters.
- **Difference**: Unlike [Gaussian Transfer Function](4408-gaussian-transfer-function.md) which uses bell curves for localized highlighting, this method uses a single cosine wave per channel to create global, oscillating color transitions. It differs from [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) by providing a multi-channel color palette rather than a single-channel scalar mapping.
- **When to choose**: When a visually rich, continuous, and non-monochromatic gradient is required without the complexity of multi-component basis functions or piecewise linear control points.
- **Link**: [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md)

### [Gaussian Transfer Function](4408-gaussian-transfer-function.md)
- **What it is**: Constructs a 4-channel RGBA transfer function texture using three Gaussian bumps to control opacity and color contribution based on scalar value position.
- **Difference**: Explicitly normalizes RGB channels by their maximum sum to prevent overflow and ensures smooth, differentiable transitions via Gaussian basis functions, unlike the piecewise or sigmoidal approaches of [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md) or the palette-based mapping of [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md).
- **When to choose**: When localized highlighting of specific scalar ranges is needed with smooth, bell-shaped transitions for both color and opacity.
- **Link**: [Gaussian Transfer Function](4408-gaussian-transfer-function.md)

### [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md)
- **What it is**: Computes the logistic sigmoid function and its analytical derivative for scalar inputs. It is used in volume rendering transfer function generation to map normalized scalar values to opacity or color intensity with smooth, differentiable transitions.
- **Difference**: Uses a monotonic S-shaped curve rather than the bell-shaped curve of [Gaussian Transfer Function](4408-gaussian-transfer-function.md), and provides a single-channel scalar mapping rather than the multi-channel color palette of [Cosine Palette Color Mapping](3359-cosine-palette-color-mapping.md).
- **When to choose**: When a smooth, monotonic transition (e.g., for opacity ramps or thresholding) is required, and the derivative is needed for optimization or gradient-based methods.
- **Link**: [Sigmoid Function with Jacobian](6497-sigmoid-function-with-jacobian.md)

### Rainbow Transfer Function Preset
- **Definition**: Generates a transfer function initialized with a 7-segment rainbow color map spanning $[0, 1]$, with no Gaussian objects [S1, unofficial].
- **Steps**:
    1. Create TF with given resolution.
    2. Clear existing controls.
    3. Add 7 ColorControls at positions $0/6, 1/6, 2/6, 3/6, 4/6, 5/6, 6/6$ [S1, unofficial].
    4. Assign RGB values: $(0, 0.364706, 1)$, $(0, 1, 0.976471)$, $(0, 1, 0.105882)$, $(0.968627, 1, 0)$, $(1, 0.490196, 0)$, $(1, 0, 0)$, $(0.662745, 0, 1)$ [S1, unofficial].
    5. Clear Gaussian objects.
    6. Call `updateColorMap()` [S1, unofficial].
- **Link**: See [Colormap LUT Generation](7229-colormap-lut-generation.md) for general LUT generation context.

### Style Function Animation
- **Definition**: A time-varying coefficient function $S(t)$ used to modify rendering style, providing smooth transitions based on user interaction start/stop events [S2, unofficial].
- **Steps**:
    - For rotation, the style function value is the same for the whole object [S2, unofficial].
    - For zooming, a spatially-dependent factor is transferred: style parameters are adjusted so that the closer a voxel's projection is to the interaction point, the higher the style-function value is [S2, unofficial].
- **Formula**:
    $$ S(t_{current}) = S(t_{start}) \cdot e^{-(t_{current} - t_{start})} + S(t_{dest}) \cdot (1 - e^{-(t_{current} - t_{start})}) $$
    where $S(t_{current})$ is the current style function value, $S(t_{start})$ is the value before interaction start/end, $S(t_{dest})$ is 0 if interaction stopped and 1 if started, $t_{current}$ is current time, and $t_{start}$ is start time [S2, unofficial].
- **Parameters**: Output range is a floating point value in interval $[0, 1]$. 0 corresponds to no interaction, 1 to very long interaction [S2, unofficial].
- **Link**: See [Dynamic Focus+Context for Volume Rendering](2689-1d-transfer-function-generation-and-sampling.md) for interaction context [S2, unofficial].

### Random Transfer Function Generation
- **Definition**: Generates a random transfer function LUT [S3, unofficial].
- **Steps**:
    1. Clear the LUT.
    2. Iterate `i` from 0 to `n_bins - 1`. If `i == 0`, push `glm::vec4(0)`. Otherwise, push `glm::vec4(randf(), randf(), randf(), randf())` [S3, unofficial].
    3. Call `upload_gpu()` [S3, unofficial].
- **Parameters**: `n_bins`: Number of bins. Default value is 8 [S3, unofficial].
- **Link**: See [Colormap LUT Generation](7229-colormap-lut-generation.md) for LUT structure.

## Background
Parametric function generation in volume rendering transfer functions has evolved from simple piecewise linear interpolation to more sophisticated analytical models [G]. Style functions using exponential decay for interaction-based rendering were introduced to provide smooth visual transitions during user interaction, such as zooming or rotating [S2, unofficial].

## Sources
[S1] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S2] (PDF) Dynamic Focus+Context for Volume Rendering - ResearchGate, https://www.researchgate.net/publication/220839241_Dynamic_FocusContext_for_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S3] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
