# Neural Shading in Pulsar

Also called: Neural Shading Network, View-Dependent Shading with Neural Network

## Overview
Neural Shading in Pulsar is a method for modeling view-dependent appearance by integrating a neural shading network that conditions on per-pixel view direction, shared across all pixels [S1]. It addresses the limitation of standard sphere-based rendering where color is often treated as a static attribute, failing to capture complex specularities or material variations that change with the viewing angle [S1]. Unlike standard splatting which assigns a fixed color to each primitive, this method decouples geometry projection from appearance, allowing the network to predict the final color based on the underlying sphere features and the camera's viewing vector [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Per-pixel view direction vector $\mathbf{v}$, sphere color/features (optimizable channel information), and camera parameters (intrinsic $K$, rotation $R$, translation $t$) used to project spheres to screen space [S1].
*   **Outputs:** Rendered feature image $F$ (final pixel colors) [S1].

**Data Structures:**
*   **Sphere Representation:** A set of spheres defined by position, radius, color, and opacity, as defined in [Pulsar Sphere-based Neural Rendering](9397-pulsar-sphere-based-neural-rendering.md).
*   **Neural Shading Network:** A fully-connected neural network (MLP). The source cites a design similar to Pix2PixHD with 15 feature channels [S1]. The exact input dimension is not specified, but it conditions on the view direction.

**Algorithm Steps:**
1.  **Projection:** Project the 3D sphere representation $S$ onto the 2D image plane using camera parameters $(R, t, K)$ to generate screen-space splats [S1].
2.  **View Direction Computation:** For each pixel, compute the view direction vector $\mathbf{v}$ from the camera center to the pixel's corresponding 3D location (or the sphere centroid) [S1].
3.  **Neural Shading:** Feed the per-pixel view direction $\mathbf{v}$ and the sphere's color/features into the neural shading network [S1].
4.  **Color Prediction:** The network outputs the final color for the pixel, effectively modulating the sphere's base color based on the viewing angle [S1].
5.  **Blending:** Composite the shaded colors from overlapping spheres using the order-independent blending mechanism of Pulsar [S1].
6.  **Optimization:** Back-propagate gradients through the shading network to optimize the sphere's channel information (colors/features) and the network weights jointly [S1].

**Loss Functions:**
The training of the neural shading network utilizes a combination of losses [S1]:
*   Photometric L1-loss.
*   Perceptual loss.
*   Adversarial loss.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_shading_in_pulsar(spheres, camera_params, view_directions, network):
    """
    spheres: List of sphere objects with attributes (position, radius, color, opacity)
    camera_params: Dict containing rotation R, translation t, intrinsic K
    view_directions: Tensor of shape (H, W, 3) containing normalized view vectors for each pixel
    network: Neural network instance (MLP)
    """
    # 1. Project spheres to screen space (Implementation details from parent article)
    screen_splats = project_spheres(spheres, camera_params)
    
    # 2. Initialize output image
    output_image = zeros_like(view_directions)
    
    # 3. Render and Shade
    for splat in screen_splats:
        # Get the base color/feature from the sphere
        base_color = splat.color
        
        # Get view direction for the pixels covered by this splat
        # Note: In a real implementation, this requires mapping screen coordinates to view vectors
        local_view_dirs = get_view_directions_for_splat(splat, camera_params)
        
        # Condition the network on view direction
        # The network takes view direction and potentially sphere features as input
        shaded_color = network(local_view_dirs, base_color)
        
        # Blend shaded color into output image using Pulsar's order-independent blending
        output_image = blend(output_image, shaded_color, splat.opacity, splat.screen_coords)
        
    return output_image

def train_neural_shading(spheres, images, network, optimizer):
    """
    Optimizes sphere colors and network weights.
    """
    for iteration in range(num_iterations):
        # Forward pass
        rendered_image = neural_shading_in_pulsar(spheres, camera_params, view_directions, network)
        
        # Compute Losses
        l1_loss = L1Loss(rendered_image, ground_truth_image)
        perceptual_loss = PerceptualLoss(rendered_image, ground_truth_image)
        adversarial_loss = AdversarialLoss(rendered_image, discriminator)
        
        total_loss = l1_loss + perceptual_loss + adversarial_loss
        
        # Back-propagate gradients through the shading model to optimize channel info
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
```

**Parameters:**
*   **Network Architecture:** Fully-connected network. Source cites a design with 15 feature channels [S1].
*   **Optimization:** Adam optimizer [S1].
*   **Training Data:** Synthetic scenes (e.g., 'chair' scene from NeRF dataset with 200 views) [S1].

**Edge Cases and Pitfalls:**
*   **Sphere Pruning:** Spheres whose color converges to the background color or receive no gradient updates from any viewpoint are eliminated to maintain a thin hull representation [S1].
*   **Depth Regularization:** An opacity-depth regularizer with energy $-z_i \cdot O_i$ is applied to encourage spheres to move to the correct scene depth [S1].

## Subtopics

### Neural View-dependent Enhancement
A strategy that leverages a neural network conditioned on 3D Gaussian attributes and viewing direction to capture view-dependent information, mitigating transparency artifacts introduced by order-independent rendering [S2, unofficial]. It is a lightweight MLP designed to extract discriminative features from 3D Gaussian primitives [S2, unofficial]. The network predicts opacity $o$ and an auxiliary feature $\boldsymbol{\phi}$. Opacity is constrained to $[0, 1]$ via Sigmoid, and $\boldsymbol{\phi}$ uses ReLU [S2, unofficial]. The MLP has 3 layers with neuron counts 256, 128, and 64 [S2, unofficial]. It is trained jointly with 3D Gaussian primitives for 30k iterations [S2, unofficial]. This variant is chosen when rendering order-independent transparency artifacts are significant and view-dependent effects are critical [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The neural shading step adds a forward pass through an MLP per pixel/splat. The source notes that Pulsar enables real-time rendering and optimization of representations with millions of spheres by limiting complexity in the projection step [S1].
*   **Memory Complexity:** Depends on the number of spheres and the size of the neural network. The source mentions models with up to 1.6 million spheres temporarily during refinement [S1].
*   **Performance:** Optimization of a coarse head model with 400k spheres took 73 seconds [S1]. Optimization of a scene with 2.1M spheres took 20 minutes on an NVIDIA V100 GPU [S1].
*   **Strengths:** Enables realistic modeling of view-dependent appearance (e.g., satin covers, hair, eyes) at high resolution [S1]. Decouples geometry and appearance [S1].
*   **Limitations:** Requires training with adversarial and perceptual losses, which can be complex to tune [S1].

## Background
The method was introduced in the paper "Pulsar: Efficient Sphere-based Neural Rendering" [S1]. It was developed to showcase the potential of the Pulsar pipeline by combining sphere-based geometry projection with neural shading. The authors demonstrated the method on synthetic scenes like 'chair' (from the NeRF dataset) and real-world objects like hair and eyes, using datasets of 100 images at 800x1280 resolution [S1]. The approach allows for the separation of geometry projection and neural shading under challenging conditions [S1].

## Sources
[S1] Pulsar: Efficient Sphere-based Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00149, fetched 2026-10-26
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
