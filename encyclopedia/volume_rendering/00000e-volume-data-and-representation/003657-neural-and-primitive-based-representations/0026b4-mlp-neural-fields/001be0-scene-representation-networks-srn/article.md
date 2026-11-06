# Scene Representation Networks (SRN)


## Overview
Scene Representation Networks (SRN) represent a continuous scene as an opaque surface implicitly defined by a Multi-Layer Perceptron (MLP) that maps 3D coordinates to feature vectors [S1]. Unlike methods that render volumetric density, SRN uses a recurrent neural network to march along rays, predicting step sizes to locate the surface intersection and decoding the final feature vector into a single color [S1]. It is distinguished from its sibling [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by its assumption of an opaque surface rather than a volumetric medium, and from [SIREN Neural Field](1250-siren-neural-field.md) by its use of recurrence for ray marching rather than frequency-aware activations for direct coordinate mapping [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Output:** A single RGB color value $\mathbf{c}$ for the point where the ray intersects the scene surface.

**Data Structures:**
1.  **Scene MLP ($\phi$):** A standard MLP that takes a 3D coordinate $\mathbf{x} \in \mathbb{R}^3$ as input and outputs a feature vector $\mathbf{f} \in \mathbb{R}^k$. The specific dimension $k$ is not given in the sources [S1].
2.  **Recurrent Network (RNN):** A recurrent unit that maintains a hidden state to predict the next step size along the ray.
3.  **Decoder:** A function that maps the final feature vector $\mathbf{f}_{final}$ to an RGB color.

**Algorithm Steps:**
1.  Initialize the ray position $\mathbf{x}_0 = \mathbf{o}$ and the RNN hidden state $h_0$.
2.  For each step $i$ in the ray march:
    a. Input the current coordinate $\mathbf{x}_i$ to the Scene MLP $\phi$ to obtain the feature vector $\mathbf{f}_i = \phi(\mathbf{x}_i)$ [S1].
    b. Input $\mathbf{f}_i$ and the previous hidden state $h_{i-1}$ to the RNN to predict the next step size $\Delta t_i$ and update the hidden state $h_i$ [S1].
    c. Update the ray position: $\mathbf{x}_{i+1} = \mathbf{x}_i + \Delta t_i \cdot \mathbf{d}$ [S1].
3.  Terminate the march when the RNN predicts a termination condition (e.g., a specific step size or a learned stop signal) [S1].
4.  Decode the feature vector $\mathbf{f}_{final}$ from the final step into a single color $\mathbf{c}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_srn(ray_origin, ray_direction, scene_mlp, rnn, decoder):
    """
    Renders a single ray using Scene Representation Networks.
    
    Args:
        ray_origin: 3D vector, origin of the camera ray.
        ray_direction: 3D vector, direction of the camera ray.
        scene_mlp: Function phi(x) -> f, maps 3D coord to feature vector.
        rnn: Recurrent unit that takes (f, h_prev) -> (step_size, h_curr).
        decoder: Function that maps final feature vector to RGB color.
        
    Returns:
        color: RGB color of the surface intersection.
    """
    # Initialize position and RNN hidden state
    current_pos = ray_origin
    hidden_state = rnn.initial_state()
    
    # Ray marching loop
    # Note: The exact termination condition (e.g., max steps, specific step size threshold)
    # is not specified in the sources [S1]. We assume a loop that runs until termination.
    while not terminated(hidden_state):
        # Get feature vector at current position
        feature_vector = scene_mlp(current_pos)
        
        # Use RNN to predict step size and update state
        step_size, hidden_state = rnn(feature_vector, hidden_state)
        
        # Advance along the ray
        current_pos = current_pos + step_size * ray_direction
        
    # Decode the final feature vector into color
    color = decoder(feature_vector)
    return color

def terminated(hidden_state):
    """
    Placeholder for termination logic.
    Sources do not specify the exact termination criterion [S1].
    [G] Common implementations might check if step_size < epsilon or a learned 'stop' probability.
    """
    # Not given in the sources.
    return False
```

**Parameters:**
*   **MLP Architecture:** Not given in the sources [S1].
*   **RNN Architecture:** Not given in the sources [S1].
*   **Feature Vector Dimension:** Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Smoothed Geometry:** SRN produces heavily smoothed geometry and texture, limiting its ability to reproduce fine details [S1].
*   **Single Depth/Color:** The method is limited by selecting only a single depth and color per camera ray, making it unsuitable for transparent or multi-layered scenes [S1].
*   **Precision:** The sources do not specify numerical precision requirements for the RNN or MLP [S1].

## Subtopics

### Signed Surface Approximation Network (SSAN)
SSAN is a module that creates a truncated signed distance field (TSDF) from NeRFs to enable the extraction of a unique and accurate surface and appearance [S2]. It is optimized from pretrained NeRFs, harnessing the rough 3D approximation of scene geometry learnt by the NeRF and useful priors such as rendered depths via volumetric rendering [S2]. SSAN is chosen when a geometrically accurate mesh and appearance are required from a NeRF representation [S2].

**Definition:**
SSAN is a neural network $\phi: \mathbb{R}^3 \rightarrow (\mathbb{R}, \mathbb{S}^2, \mathbb{R}^8)$ that takes a 3D coordinate $\mathbf{x}$ as input and predicts:
*   Truncated signed distance approximation $\hat{t} \in [-0.1, 0.1]$
*   Normal $\hat{\mathbf{n}} \in \mathbb{S}^2$
*   8-dimensional appearance feature $\hat{\mathbf{f}} \in [0, 1]$

The output is defined as:
$$ [\hat{t}, \hat{\mathbf{n}}, \hat{\mathbf{f}}] = \phi(\mathbf{x}) $$

**View-Dependent Color:**
An appearance network $\eta: (\mathbb{R}^8, \mathbb{S}^2, \mathbb{S}^2) \rightarrow \mathbb{R}^3$ takes the outgoing features $\hat{\mathbf{f}}$, normals $\hat{\mathbf{n}}$, and viewing direction $\mathbf{d} \in \mathbb{S}^2$ to produce the view-dependent RGB color $\hat{\mathbf{c}} \in [0, 1]$:
$$ \hat{\mathbf{c}} = \eta(\hat{\mathbf{f}}, \hat{\mathbf{n}}, \mathbf{d}) $$

**Parameters:**
*   **TSDF Range:** $\hat{t} \in [-0.1, 0.1]$ [S2].
*   **Appearance Feature Dimension:** 8-dimensional vector $\hat{\mathbf{f}} \in [0, 1]$ [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** SRN captures low-frequency geometry and color variation in each scene but is unable to reproduce any fine detail [S1]. It produces heavily smoothed geometry and texture [S1].
*   **Strengths:** Represents a continuous scene as an opaque surface [S1].
*   **Limitations:** Limited representational power for view synthesis due to selecting only a single depth and color per camera ray [S1]. Unable to reproduce fine detail [S1].
*   **When to Use:** Suitable for scenes with low-frequency geometry and color variation where an opaque surface representation is sufficient [S1].
*   **When Not to Use:** Unsuitable for scenes requiring fine detail, transparency, or multiple surfaces along a ray [S1].

## Background
Scene Representation Networks (SRN) were introduced in the paper "NeRF: Representing scenes as neural radiance fields for view synthesis" [S1]. SRN is a better-performing followup to DeepVoxels [41] by the same authors [S1]. It was designed to represent continuous scenes as opaque surfaces, implicitly defined by an MLP [S1]. The method trains a recurrent neural network to march along a ray through the scene representation [S1].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
