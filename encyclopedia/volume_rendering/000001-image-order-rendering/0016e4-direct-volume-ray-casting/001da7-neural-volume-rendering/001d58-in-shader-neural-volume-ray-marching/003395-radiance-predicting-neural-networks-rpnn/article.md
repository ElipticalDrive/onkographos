# Radiance-Predicting Neural Networks (RPNN)

Also called: Kallweit et al. method, RDNN, Radiance-Predicting Neural Networks (RPNN) for clouds

## Overview
Radiance-Predicting Neural Networks (RPNN) is a neural volume rendering method for clouds that uses a multilayer perceptron (MLP) to predict radiance for various shading configurations, combined with Monte Carlo integration for direct and single scattering [S1, S2]. It sets itself apart from its sibling [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md) by explicitly handling multiple scattering through learned radiance predictions rather than explicit volumetric integration, and differs from [Neural Volume Scattering Prediction](8439-neural-volume-scattering-prediction.md) by predicting final radiance directly from shading configurations instead of solving the radiative transfer equation with sampling templates [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Shading configuration descriptors containing location, view and light directions, light source position, and cloud density structure represented by a hierarchy of point stencils.
*   **Outputs:** Predicted radiance for the desired shading configuration.

**Data Structures:**
*   **MLP:** A multilayer perceptron with fully connected layers that maps descriptors to radiance values.
*   **Hierarchical Point Stencils:** A grid of point samples representing cloud density at multiple spatial scales, fed progressively into the network.

**Steps:**
1.  Extract descriptors from the visible points of the clouds for the target scene [S1].
2.  Represent the cloud density structure using a hierarchy of point stencils at varying scales [S2].
3.  Feed the hierarchical descriptors progressively into the MLP to enhance learning speed and prediction accuracy [S1].
4.  The MLP predicts the radiance function for the desired shading configurations [S1].
5.  During rendering, compute direct lighting and single scattering using Monte Carlo integration, and query the trained MLP for multiple scattering based on the current shading configuration [S1, S2].

**Formulas:**
Not given in the sources. The method combines Monte Carlo integration results for direct and single scattering with the neural network's prediction for multiple scattering, but the exact mathematical combination formula is not specified [S1, S2].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ProgressiveRPNN(nn.Module):
    def __init__(self, descriptor_dim, hidden_dims=[256, 256, 256]):
        super().__init__()
        # Progressive feeding layers: each layer corresponds to a stencil scale
        self.layers = nn.ModuleList()
        input_dim = descriptor_dim
        for dim in hidden_dims:
            self.layers.append(nn.Linear(input_dim, dim))
            self.layers.append(nn.ReLU())
            input_dim = dim
        # Output: radiance prediction (RGB)
        self.output_layer = nn.Linear(input_dim, 3)

    def forward(self, descriptors, stencil_scales):
        # descriptors: tensor of shape (N, descriptor_dim)
        # stencil_scales: list of tensors, each (N, scale_dim), ordered coarse to fine
        x = descriptors
        for i, layer in enumerate(self.layers[:-1]):
            # Progressive feeding: concatenate current scale descriptor at each layer
            scale_input = stencil_scales[i] if i < len(stencil_scales) else descriptors
            x = torch.cat([x, scale_input], dim=-1)
            x = layer(x)
        # Final layer outputs radiance
        radiance = self.output_layer(x)
        return radiance

def render_rpnn(scene, camera, light):
    # 1. Extract descriptors from visible points
    visible_points = extract_visible_points(scene, camera)
    descriptors = extract_descriptors(visible_points, light)
    
    # 2. Build hierarchical point stencil representation of density
    density_stencils = build_hierarchical_point_stencils(scene.density_field)
    
    # 3 & 4. Progressive feeding to MLP and radiance prediction
    mlp_radiance = ProgressiveRPNN(descriptor_dim=10)
    multiple_scattering_radiance = mlp_radiance(descriptors, density_stencils)
    
    # 5. Monte Carlo for direct & single scattering
    mc_radiance = monte_carlo_integration(scene, camera, light, visible_points)
    
    # Combine MC (direct/single) with NN (multiple scattering)
    # Exact combination formula not specified in sources [S1, S2]
    final_radiance = mc_radiance + multiple_scattering_radiance
    return final_radiance

def extract_visible_points(scene, camera):
    # Standard visibility test against scene geometry/volume
    return [p for p in scene.points if camera.ray_through(p) and not scene.is_occluded(p)]

def extract_descriptors(points, light):
    # Extract location, direction, light source, and density structure
    return torch.stack([p.position, p.view_dir, p.light_dir, p.density_value] for p in points)

def build_hierarchical_point_stencils(density_field):
    # Generate point stencils at multiple spatial scales
    return [density_field.sample_at_scale(scale) for scale in [1, 2, 4, 8]]

def monte_carlo_integration(scene, camera, light, points):
    # Standard Monte Carlo estimation for direct and single scattering
    radiance = torch.zeros(len(points), 3)
    for p in points:
        radiance += estimate_direct_and_single_scattering(p, light)
    return radiance
```

## Subtopics

### [Neural Volume Scattering Prediction](8439-neural-volume-scattering-prediction.md)
Neural Volume Scattering Prediction is a method for rendering participating media that uses a neural network to predict the scattering distribution at the centers of sampling templates, which is then used to solve the radiative transfer equation. It sets itself apart from its sibling [Neural Radiative Transfer Equation Solver](8466-neural-radiative-transfer-equation-solver.md) and the earlier [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md) by decomposing input features into density, transmission, and phase, and by using specific sampling templates (diffuse and highlight) to simplify the network structure and input parameters.

### [Neural Radiative Transfer Equation Solver](8466-neural-radiative-transfer-equation-solver.md)
Solves the radiative transfer equation using neural networks, differing from RPNN's direct color prediction.

## Operating characteristics
*   **Time Complexity:** Rendering is approximately a thousand times faster than path tracing [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Rendering results are almost identical to path tracing [S2]. Achieves effects like cloud whiteness, edge-darkening, and silver lining with a few minutes of computation [S1].
*   **Strengths:** Handles multiple scattering efficiently, achieves high visual fidelity close to path tracing, and accelerates rendering significantly [S1, S2].
*   **Limitations:** Requires precomputed learning data from several cloud exemplars to train on spatial and directional light distribution [S1].
*   **When to use:** Real-time or accelerated cloud rendering where visual fidelity close to path tracing is required [S2].
*   **When not to use:** Scenes that lack precomputed cloud exemplars or require on-the-fly training without prior data [S1].

## Background
RPNN was proposed by Kallweit et al. [S1, S2]. It represents radiance for each shading configuration, including location, direction, light source, and density structure [S2]. The method uses a hierarchy of point stencils to represent varying scales of cloud density [S2]. It was developed to address the high computational cost of simulating multiple scattering events in atmospheric clouds, which are high-order scattering media [S2].

## Sources
[S1] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
