# Scaling

Also called: Scalar Field Scaling

## Overview
Scaling is a fundamental data preprocessing technique in volume rendering that transforms input scalar field values or spatial coordinates into a normalized range, typically [0, 1] or [-1, 1]. It solves the problem of disparate data magnitudes and coordinate scales, which can cause numerical instability, poor convergence in neural optimization, or incorrect visual mapping. It sets itself apart from non-linear normalization methods (like [Power Transformation](2346-power-transformation.md)) by preserving linear relationships within the mapped range, and from global statistical methods (like [Percentile-based Normalization](5158-percentile-based-normalization.md)) by relying on fixed or global extrema rather than distribution percentiles.

## How it works
### Inputs and Outputs
- **Inputs**: A scalar field $V$ (e.g., Hounsfield Units, raw density) or a set of 3D coordinates $P$. A scale factor $s$ and optionally a bias $b$.
- **Outputs**: A normalized scalar field $V'$ or transformed coordinates $P'$.

### Data Structures
- **Scalar Field**: A 3D grid or tensor of floating-point values.
- **Coordinates**: A vector of shape $(N, 3)$ representing spatial positions.

### Steps
1. **Global Min/Max Scaling**: Compute the global minimum $min_v$ and maximum $max_v$ of the input scalar field.
2. **Normalization**: Apply the linear transformation:
   $$ V' = \frac{V - min_v}{max_v - min_v} $$
   This maps the range $[min_v, max_v]$ to $[0, 1]$.
3. **Coordinate Scaling**: For neural rendering or optimization, scale spatial coordinates $p$ by a factor $s$:
   $$ p' = \frac{p}{s} $$
   This ensures coordinates fall within a canonical range (e.g., $[-1, 1]$) [S1, unofficial][S3].

### Parameters
- **Scale Factor ($s$)**: Determines the spatial extent. For Voronoi-based methods, $s$ is chosen such that sites $\phi \in [-1, 1]^{N \times 3}$ [S3].
- **Clamping**: Values outside the target range are often clamped to the boundaries [S2, unofficial].

### Edge Cases and Pitfalls
- **Zero Division**: If $max_v = min_v$, the denominator is zero. Handle by returning zeros or a constant [G].
- **Optimizer Instability**: Without coordinate scaling, optimizers like ADAM may oscillate or converge slowly due to variable magnitude invariance issues [S3].
- **Precision**: In fixed-point or low-precision formats, extreme dynamic ranges may lose detail.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_scalar_field(volume, min_val=None, max_val=None):
    """
    Scale a scalar volume to [0, 1].
    If min_val/max_val are None, compute from volume.
    """
    if min_val is None:
        min_val = volume.min()
    if max_val is None:
        max_val = volume.max()
    
    # Guard against zero division
    if max_val == min_val:
        return volume * 0.0
    
    normalized = (volume - min_val) / (max_val - min_val)
    return normalized

def scale_coordinates(points, scale_factor):
    """
    Scale 3D points by a factor.
    points: Tensor of shape (N, 3)
    scale_factor: float
    """
    return points / scale_factor
```

## Subtopics

### [Power Transformation](2346-power-transformation.md)
The Power Transformation is a parameterized function used to map metric distance $t$ to a normalized distance $s$, enabling effective resampling and distortion loss calculation in neural radiance fields. It sets itself apart from discrete normalization choices (like pure log or inverse distance) by allowing smooth interpolation between linear, logarithmic, and inverse behaviors via a single shape parameter $\lambda$, while maintaining a linear-like shape near the origin to avoid singularities at the ray start.

### [Scaling and Biasing](2888-scaling-and-biasing.md)
Scaling and Biasing is a fragment shader-based technique that applies a linear transformation to volume texture samples to adjust contrast and brightness, commonly known as window/level adjustment. It maps a specific range of input intensities (the window) to the full display range, while clamping values outside this range to the display minimum or maximum. Unlike non-linear methods such as Power Transformation, it preserves the linear relationship between input intensity and output brightness within the window.

### Linear HU to Attenuation Map
A simple linear mapping from Hounsfield Units (HU) to attenuation coefficients ($\mu$) with clamping [S2, unofficial].
- **Step**: Calculate intermediate HU value: $hu = vol \times (3024 - (-3524)) + (-3524)$.
- **Step**: Calculate $\mu$: $\mu = 0.05 \times (1.0 + hu / 800.0)$ [S2, unofficial].

### Parameter Magnitude Scaling
A preprocessing step scaling the scene coordinate system so that Voronoi sites $\phi \in [-1, 1]^{N \times 3}$ [S3]. This is necessary when using optimizers like ADAM to prevent training oscillation or slow progress due to variable magnitude invariance issues [S3]. Without scaling, common optimizers like ADAM may fail to converge properly because the training process is not invariant to the magnitude of individual variables [S3].

## Operating characteristics
- **Time Complexity**: $O(N)$ for computing min/max and scaling, where $N$ is the number of voxels or points.
- **Memory Complexity**: $O(1)$ auxiliary space if computed in-place, or $O(N)$ for a new normalized array.
- **Strengths**: Simple, fast, and effective for normalizing data for neural networks or visualization.
- **Limitations**: Sensitive to outliers in global min/max scaling. Does not preserve local contrast variations as well as percentile-based methods.
- **When to Use**: When data range is known and stable, or when preparing coordinates for neural optimization.
- **When Not to Use**: When data contains significant outliers that distort the global range, or when non-linear perceptual mapping is required.

## Background
Scaling is a foundational technique in computer graphics and scientific visualization. Its application in neural volume rendering, particularly for coordinate scaling to prevent optimizer instability, was highlighted in the DeRF paper [S3]. The specific linear mapping from HU to attenuation coefficients is used in medical imaging pipelines [S2, unofficial]. The Power Transformation for distance mapping in neural fields is described in the Torch-Sphere-Tracer repository [S1, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S3] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
