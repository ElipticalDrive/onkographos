# Conditional Radiance Field Architecture

Also called: Conditional NeRF, Shape-Color Disentangled NeRF

## Overview
Conditional Radiance Field Architecture extends the standard Neural Radiance Field (NeRF) by conditioning the volumetric representation on instance-specific latent codes, enabling the modeling of an entire class of objects (e.g., "chairs") rather than a single static scene [S1]. It distinguishes itself from standard NeRF by disentangling shape and appearance into separate latent codes ($z^{(s)}$ and $z^{(c)}$) and modifying the network topology to support efficient editing of these attributes [S1]. Unlike global conditioning methods, it allows for specific components of the network to be updated during editing tasks, balancing computational efficiency with reconstruction accuracy [S1].

## How it works
The architecture maps a 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and instance-specific latent codes to density $\sigma$ and radiance $\mathbf{c}$. The rendering process follows the standard differentiable volume rendering equation [44-radiance-field-optimization.md], but the network inputs and internal structure are modified to support conditional generation and editing.

### Inputs and Outputs
*   **Inputs:**
    *   3D position $\mathbf{x} \in \mathbb{R}^3$.
    *   Viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   Shape code $z^{(s)} \in \mathbb{R}^{d_s}$: Instance-specific latent vector controlling geometry.
    *   Color code $z^{(c)} \in \mathbb{R}^{d_c}$: Instance-specific latent vector controlling appearance.
*   **Outputs:**
    *   Density $\sigma \in \mathbb{R}$.
    *   Radiance $\mathbf{c} \in \mathbb{R}^3$.

### Data Structures and Network Layout
The network is composed of modular parts to disentangle shape and color. It builds on the standard NeRF architecture [4972-neural-radiance-fields.md] with the following specific modifications [S1]:

1.  **Skip Connections:** The shape code $z^{(s)}$ and the embedded input points (position $\mathbf{x}$ and direction $\mathbf{d}$) are fed into two places:
    *   The "fusion shape network" (the initial layers processing geometry).
    *   The very beginning of the network (standard NeRF skip connection).
2.  **Code Injection:** When injecting a shape code $z^{(s)}$ or color code $z^{(c)}$ into a specific layer $l$:
    *   The code is passed through a linear layer $W_l$ followed by a ReLU nonlinearity.
    *   The output is concatenated with the layer's input features.
    *   Formula for injection at layer $l$:
        $$ \text{input}_l' = \text{concat}(\text{input}_l, \text{ReLU}(W_l z^{(k)})) $$
        where $k \in \{s, c\}$.
3.  **Bottleneck:** A bottleneck layer is introduced in the architecture to reduce dimensionality, making feature caching computationally feasible during editing.
4.  **Color Branch Input:** The input to the color branch is an 8-dimensional vector. This vector is cached during color editing to avoid recomputation [S1].

### Numbered Steps
1.  **Embedding:** Embed position $\mathbf{x}$ and direction $\mathbf{d}$ using positional encoding (as in standard NeRF).
2.  **Shape Processing:**
    *   Concatenate embedded $\mathbf{x}$, embedded $\mathbf{d}$, and shape code $z^{(s)}$.
    *   Pass through the initial layers (fusion shape network).
    *   Inject $z^{(s)}$ into subsequent layers via the linear+ReLU+concat mechanism.
3.  **Color Processing:**
    *   Extract the 8-dimensional bottleneck feature vector.
    *   Inject color code $z^{(c)}$ into the later layers of the network via the linear+ReLU+concat mechanism.
4.  **Output Prediction:**
    *   The network outputs density $\sigma$ and radiance $\mathbf{c}$.
    *   Density $\sigma$ is typically passed through a softplus or ReLU to ensure non-negativity (implementation detail inferred from [S2] and standard practice, though [S1] does not explicitly specify the activation for $\sigma$ in the base architecture, it relies on the standard NeRF formulation).
5.  **Rendering:**
    *   Sample points along rays.
    *   Evaluate $\sigma$ and $\mathbf{c}$ for each point using the conditional network.
    *   Compute transmittance $T_i$ and accumulated color $\hat{C}(\mathbf{r})$ using the standard volume rendering equation [44-radiance-field-optimization.md]:
        $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
        where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$.

### Parameters
*   **Shape Code Dimension ($d_s$):** Not given in the sources.
*   **Color Code Dimension ($d_c$):** Not given in the sources.
*   **Bottleneck Dimension:** 8 dimensions for the color branch input [S1].
*   **Linear Layer Channels:** Not specified for the base conditional architecture [S1].

### Edge Cases and Pitfalls
*   **View Direction Dependence:** Using only one training view per instance can cause color inconsistency across rendered views. This is addressed by regularizing the view dependence of radiance $\mathbf{c}$ with a self-consistency loss [S1].
*   **Editing Stability:** Updating all network weights is slow; updating only latent codes yields low quality. The hybrid update strategy (updating specific layers and codes) is required for efficient editing [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conditional_nerf_forward(x, d, z_s, z_c):
    """
    Forward pass for Conditional Radiance Field.
    
    Args:
        x: 3D position, shape (..., 3)
        d: Viewing direction, shape (..., 3)
        z_s: Shape code, shape (..., d_s)
        z_c: Color code, shape (..., d_c)
        
    Returns:
        sigma: Density, shape (..., 1)
        c: Radiance, shape (..., 3)
    """
    # 1. Embedding (Standard NeRF positional encoding)
    x_embed = positional_encoding(x)
    d_embed = positional_encoding(d)
    
    # 2. Initial Fusion Shape Network
    # Concatenate embedded inputs and shape code
    initial_input = concat(x_embed, d_embed, z_s)
    
    # Process through initial layers
    # Note: Exact layer definitions are not provided in [S1], 
    # assuming a standard MLP structure with skip connections.
    h = initial_mlp(initial_input)
    
    # 3. Inject Shape Code into Intermediate Layers
    # The shape code is injected into the "fusion shape network" and 
    # potentially other layers as per the modular design.
    # For each layer l where shape code is injected:
    for l in shape_injection_layers:
        z_s_projected = relu(linear_layer(l, z_s))
        h = concat(h, z_s_projected)
        h = layer_forward(h)
        
    # 4. Bottleneck and Color Branch
    # Extract 8-dimensional bottleneck vector
    bottleneck_vec = get_bottleneck(h) # Shape (..., 8)
    
    # Inject Color Code into Later Layers
    # The color code is injected into the later layers of the network
    for l in color_injection_layers:
        z_c_projected = relu(linear_layer(l, z_c))
        h = concat(h, z_c_projected)
        h = layer_forward(h)
        
    # 5. Output Prediction
    sigma = output_sigma(h)
    c = output_color(h)
    
    return sigma, c

def volume_rendering(rays, conditional_nerf, z_s, z_c):
    """
    Standard differentiable volume rendering.
    """
    # Sample points along rays
    points = sample_points_along_rays(rays)
    
    # Evaluate network
    sigma, c = conditional_nerf_forward(points, rays.direction, z_s, z_c)
    
    # Compute transmittance and accumulated color
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # C = sum_{i=1}^N T_i * (1 - exp(-sigma_i * delta_i)) * c_i
    accumulated_color = standard_volume_rendering(points, sigma, c, rays.delta)
    
    return accumulated_color
```

## Subtopics

### [Hybrid Updates for Editing](15974-hybrid-updates-for-editing.md)
Hybrid Updates for Editing is a strategy that targets specific network components for updates to balance efficiency and accuracy during user interaction in conditional radiance fields. It updates only specific layers of the network and latent codes to achieve both accuracy and efficiency, avoiding the slowness of updating all weights and the low quality of updating only latent codes. Unlike updating the entire network or only codes, this method updates only a subset of components (later layers) to achieve significant computational speed-up.

### [Conditional Radiance Field Editing](15977-conditional-radiance-field-editing.md)
Not given in the sources.

### Locally Conditioned Radiance Field Network
A neural radiance field network that maps 3D coordinates and view directions to appearance $a$ and occupancy $\sigma$, conditioned on local latent codes $w_{ij}$ to represent multiple distinct radiance fields [S2]. It improves performance over global conditioning by decomposing the scene into many independent locally conditioned radiance fields [S2].

*   **Definition:** A modular network architecture for conditional radiance fields that includes a shape branch shared across object instances to capture part semantics [S1]. The shape branch provides an inductive bias to encode a shared representation whenever possible, improving shape reconstruction accuracy [S1].
*   **Layout:** The network weights are updated during color and shape editing, while codes remain static in the "Network with Shared Branch" variant [S1]. In the "Hybrid Update" variant, the shape code is updated during shape editing, network weights are updated during color editing, and network weights are updated during shape editing [S1].
*   **Comparison:** Sampling higher resolution feature maps directly from the radiance field yields higher quality results than sampling at low resolution and applying many refinement blocks, but incurs significantly higher computational cost [S2].

### [GSN Model Architecture Components](16659-gsn-model-architecture-components.md)
GSN uses locally conditioned radiance fields, where the scene is decomposed into many independent locally conditioned radiance fields, improving performance over global conditioning.

## Operating characteristics
*   **Time Complexity:** The base architecture has similar time complexity to standard NeRF per ray evaluation, but the modular design and code injection add minor overhead. Hybrid updates reduce training time for editing tasks by limiting the number of parameters updated [S1].
*   **Memory Complexity:** The bottleneck layer (8-dimensional input to color branch) allows for feature caching, reducing memory usage during editing [S1]. Standard NeRF memory usage applies to the forward pass.
*   **Performance:** The architecture enables effective user edits with significant computational speed-up compared to updating the entire network [S1]. It handles complex edits without explicit spatial or boundary constraints, and edits appear consistently across viewpoints [S1].
*   **Strengths:** Disentangles shape and appearance; supports editing of entire object classes; efficient editing via hybrid updates.
*   **Limitations:** Requires careful tuning of injection layers and code dimensions; view dependence can cause inconsistencies if not regularized [S1].

## Background
Conditional Radiance Field Architecture was introduced in "Editing Conditional Radiance Fields" [S1]. It builds on the original NeRF representation [4972-neural-radiance-fields.md] to enable editing over an entire shape class (e.g., "chairs") rather than a single static scene [S1]. The work investigates which parts of the network affect different editing tasks, showing that shape and color edits can effectively take place in the later layers of the network [S1]. It also introduces a shape branch shared across object instances to implicitly bias the network to encode a shared representation [S1].

## Sources
[S1] Editing Conditional Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00572, fetched 2026-10-26
[S2] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
