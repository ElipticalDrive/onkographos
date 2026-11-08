# SDF-based Volume Rendering

Also called: NeuS Opacity Derivation, SDF-to-Opacity via Sigmoid

## Overview
SDF-based Volume Rendering is a neural rendering technique that reconstructs surfaces by optimizing a Signed Distance Field (SDF) and rendering it via volume integration. It solves the problem of differentiating through the zero-level set of an SDF during volume rendering by mapping SDF values to opacity using a logistic function. It differs from naive SDF-to-density mappings by ensuring the weight function peaks at the surface intersection, avoiding bias, and builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md).

## How it works
**Inputs:**
- A neural network predicting the SDF value $f(\mathbf{x})$ for any 3D point $\mathbf{x} \in \mathbb{R}^3$.
- A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Ray marching steps $t_i$ along the ray, where $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$.

**Outputs:**
- Rendered color $C(\mathbf{r})$ and accumulated opacity $A(\mathbf{r})$ for the ray $\mathbf{r}$.

**Data Structures:**
- The SDF network is typically a Multi-Layer Perceptron (MLP) or a hash-encoded network (e.g., Multi-resolution Hash Encoding [S1]).
- No explicit voxel grid is required; the SDF is queried implicitly.

**Steps:**
1. **Ray Marching:** Sample points $\mathbf{x}_i$ along the ray at intervals $t_i$.
2. **SDF Query:** Compute $f_i = f(\mathbf{x}_i)$ for each sample point.
3. **Opacity Calculation:** Compute the opacity $\alpha_i$ for the segment $[t_i, t_{i+1}]$ using the logistic mapping derived from the SDF values at the endpoints.
4. **Color Prediction:** Predict color $c_i$ at each sample point (usually via a separate radiance network conditioned on position and view direction, though the specific color network is not detailed in the SDF derivation source [S1]).
5. **Volume Rendering:** Composite the color and opacity using standard volume rendering equations.

**Formulas:**
The opacity $\alpha_i$ for the segment between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ is computed as [S1]:

$$
\alpha_i = \max \left( \frac{\Phi_s(f(\mathbf{x}_i)) - \Phi_s(f(\mathbf{x}_{i+1}))}{\Phi_s(f(\mathbf{x}_i))}, 0 \right)
$$

Where:
- $\Phi_s$ is the sigmoid function: $\Phi_s(z) = \frac{1}{1 + e^{-z}}$.
- $f(\mathbf{x}_i)$ is the SDF value at point $\mathbf{x}_i$.
- The term $\Phi_s(f(\mathbf{x}_i))$ represents the cumulative probability mass from $-\infty$ to $f(\mathbf{x}_i)$.

**Parameters:**
- The sigmoid function $\Phi_s$ is used directly without an explicit scale parameter $s$ in the formula provided by [S1], implying a standard logistic distribution or a fixed scale absorbed into the SDF prediction.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $\Phi_s(f(\mathbf{x}_i)) = 0$, the denominator is zero. The $\max(\cdot, 0)$ operator handles the numerator, but the division must be guarded. In practice, if $f(\mathbf{x}_i)$ is very negative, $\Phi_s$ approaches 0, and $\alpha_i$ should be 0.
- **Precision:** SDF values can be large. The sigmoid function saturates for large positive or negative values, potentially causing vanishing gradients.
- **Boundary Conditions:** The method assumes the SDF is well-defined along the ray.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(z):
    """Standard logistic sigmoid function."""
    # Guard against overflow in exp
    if z >= 0:
        return 1.0 / (1.0 + math.exp(-z))
    else:
        ez = math.exp(z)
        return ez / (1.0 + ez)

def compute_sdf_opacity(f_i, f_i_plus_1):
    """
    Compute opacity alpha_i for segment [x_i, x_{i+1}] given SDF values.
    
    Args:
        f_i: SDF value at x_i
        f_i_plus_1: SDF value at x_{i+1}
        
    Returns:
        alpha_i: Opacity for the segment
    """
    phi_i = sigmoid(f_i)
    phi_i_plus_1 = sigmoid(f_i_plus_1)
    
    # Avoid division by zero
    if phi_i == 0:
        return 0.0
        
    numerator = phi_i - phi_i_plus_1
    alpha_i = numerator / phi_i
    
    # Clamp to [0, 1]
    return max(0.0, min(1.0, alpha_i))

def render_sdf_ray(ray_origin, ray_direction, sdf_network, color_network, num_samples):
    """
    Render a single ray using SDF-based volume rendering.
    
    Args:
        ray_origin: 3D vector
        ray_direction: 3D vector (normalized)
        sdf_network: Function returning f(x)
        color_network: Function returning c(x, view_dir)
        num_samples: Number of marching steps
        
    Returns:
        color: Rendered color
        opacity: Accumulated opacity
    """
    # Define ray bounds (simplified)
    t_min = 0.0
    t_max = 10.0 # Example bound
    t_vals = [t_min + i * (t_max - t_min) / num_samples for i in range(num_samples + 1)]
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    for i in range(num_samples):
        t_i = t_vals[i]
        t_i_plus_1 = t_vals[i+1]
        
        x_i = ray_origin + ray_direction * t_i
        x_i_plus_1 = ray_origin + ray_direction * t_i_plus_1
        
        # Query SDF
        f_i = sdf_network(x_i)
        f_i_plus_1 = sdf_network(x_i_plus_1)
        
        # Compute opacity
        alpha_i = compute_sdf_opacity(f_i, f_i_plus_1)
        
        # Query color (assuming view direction is constant for simplicity)
        c_i = color_network(x_i, ray_direction)
        
        # Volume rendering contribution
        # W_i = alpha_i * product(1 - alpha_j) for j < i
        # This requires tracking the product of (1 - alpha)
        # For simplicity, using standard alpha compositing step
        weight = alpha_i * (1.0 - accumulated_opacity)
        accumulated_color = [accumulated_color[j] + weight * c_i[j] for j in range(3)]
        accumulated_opacity += weight
        
        # Early termination if fully opaque
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color, accumulated_opacity
```

## Subtopics

### [MPF Volume Rendering](10260-mpf-volume-rendering.md)
MPF Volume Rendering is a neural volume rendering technique that replaces the Signed Distance Field (SDF) backbone with Metric-Phase Fields (MPFs) to handle sign-ambiguous geometry and thin structures. It derives opacity from a signed composite field $\phi$ using a NeuS-style logistic mapping, while color is predicted by a separate radiance network, isolating the geometry representation change from the rendering pipeline.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of ray marching steps. Each step requires an SDF query and a sigmoid evaluation.
- **Memory Complexity:** Depends on the SDF network size. Implicit representations (MLP/Hash) require less memory than explicit voxel grids.
- **Strengths:** Allows optimization of surface geometry via volume rendering losses. Handles thin structures better than naive density mappings.
- **Limitations:** Requires careful tuning of the sigmoid scale. SDF networks can be sensitive to initialization.
- **When to use:** When reconstructing high-fidelity surfaces from multi-view images or sparse views.
- **When not to use:** For volumetric data with internal structures not defined by a zero-level set (e.g., smoke, clouds).

## Background
The method originates from the work of Wang et al. [41], who proposed converting volume density predictions in NeRF to SDF representations using a logistic function to allow optimization with neural volume rendering. This approach was adopted in Neuralangelo [S1] for high-fidelity neural surface reconstruction.

## Sources
[S1] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
