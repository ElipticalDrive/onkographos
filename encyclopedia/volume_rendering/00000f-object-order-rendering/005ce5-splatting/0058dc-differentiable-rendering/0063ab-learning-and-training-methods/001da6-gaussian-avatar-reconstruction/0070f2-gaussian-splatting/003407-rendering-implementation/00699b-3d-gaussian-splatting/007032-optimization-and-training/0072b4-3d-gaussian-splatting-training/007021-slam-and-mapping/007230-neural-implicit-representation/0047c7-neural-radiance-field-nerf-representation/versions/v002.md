# Neural Radiance Field (NeRF) Representation

Also called: Neural Radiance Field

## Overview
NeRF represents a 3D scene as a continuous volumetric function that maps 5D coordinates (3D position and 2D viewing direction) to 4D output values (3D color and 1D volume density) using a multilayer perceptron (MLP) [S1, unofficial]. It solves the problem of novel view synthesis by optimizing this implicit representation to match a set of input images with known camera poses, enabling the rendering of photorealistic images from viewpoints not present in the training data [S1, unofficial]. Unlike explicit representations such as [3D Gaussian Splatting](12052-3d-gaussian-splatting.md) which store millions of discrete blobs, NeRF stores the scene implicitly within a neural network, resulting in significantly lower memory usage but requiring ray marching and network evaluation for rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $I_n$ with corresponding camera poses $T_n$ (extrinsics) and intrinsics.
*   **Outputs:** A trained MLP $\mathcal{F}_\Theta$ that approximates the scene function, and rendered images from novel viewpoints.

**Data Structures:**
The core data structure is a Multilayer Perceptron (MLP).
*   **Position Network:** Takes a 3D point $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   Input: Concatenation of $\mathbf{x}$ and $\mathbf{d}$.
    *   Hidden layers: Typically 8 fully connected layers with 256 units each, using ReLU activation.
    *   Skip connections: Inputs $\mathbf{x}$ are concatenated into hidden layers at indices 4 and 8 to preserve high-frequency details [S1, unofficial].
    *   Output: Intermediate feature vector $\mathbf{h}$ and volume density $\sigma$.
*   **Color Network:** Takes the feature vector $\mathbf{h}$ and viewing direction $\mathbf{d}$.
    *   Input: Concatenation of $\mathbf{h}$ and $\mathbf{d}$.
    *   Hidden layers: Typically 1 fully connected layer with 128 units, using ReLU activation.
    *   Output: View-dependent color $\mathbf{c} \in \mathbb{R}^3$ (RGB).

**Position Encoding:**
To enable the MLP to learn high-frequency details, input coordinates are mapped to higher dimensions using positional encoding $\gamma$. For a coordinate $x$, the encoding is:
$$ \gamma(x) = \left( \sin(2^0 \pi x), \cos(2^0 \pi x), \dots, \sin(2^{L-1} \pi x), \cos(2^{L-1} \pi x) \right) $$
where $L$ is the number of frequencies. This is applied to both $\mathbf{x}$ and $\mathbf{d}$ before entering the network [S1, unofficial].

**Rendering (Volume Rendering):**
To render a pixel, a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is cast from the camera origin $\mathbf{o}$ through the pixel center. The ray is sampled at $K$ points along its length within the scene bounds. The color $C(\mathbf{r})$ is computed via differentiable volume rendering:
$$ C(\mathbf{r}) = \sum_{i=1}^{K} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i $$
where:
*   $\sigma_i$ is the volume density at sample point $i$.
*   $\mathbf{c}_i$ is the color at sample point $i$.
*   $\delta_i$ is the distance between sample point $i$ and $i+1$.
*   $T_i$ is the accumulated transmittance up to point $i$:
    $$ T_i = \exp\left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right) $$
    $T_i$ represents the probability that the ray travels from the camera to point $i$ without being absorbed or scattered [S1, unofficial].

**Training Steps:**
1.  **Initialization:** Initialize the MLP weights randomly.
2.  **Sampling:** For each training image, cast rays through each pixel. Sample points along each ray using uniform sampling or stratified sampling.
3.  **Query:** Pass sample points and viewing directions through the MLP to get $\sigma_i$ and $\mathbf{c}_i$.
4.  **Rendering:** Compute the rendered pixel color $C(\mathbf{r})$ using the volume rendering equation.
5.  **Loss Calculation:** Compare the rendered image with the ground truth image $I_{gt}$ using a loss function $L$. The standard loss combines L1 loss and Structural Similarity (SSIM):
    $$ L = (1 - \alpha) L_{1} + \alpha L_{SSIM} $$
    where $\alpha$ is a weight (typically 0.2) [S1, unofficial].
6.  **Optimization:** Backpropagate the loss through the differentiable volume rendering process to update the MLP weights $\Theta$ using an optimizer like Adam [S1, unofficial].

**Parameters:**
*   Number of samples per ray ($K$): Typically 128–256 for uniform sampling, or more for hierarchical sampling.
*   MLP hidden units: 256 (position network), 128 (color network).
*   Number of hidden layers: 8 (position), 1 (color).
*   Position encoding frequencies ($L$): Typically 10.

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision issues can arise when computing transmittance $T_i$ for very long rays or very small densities, potentially leading to NaNs or artifacts [G].
*   **Boundaries:** The scene bounds must be carefully defined. Points outside the bounds should be assigned zero density or handled via a bounding box [G].
*   **Degenerate Input:** If camera poses are inaccurate, the optimization may fail to converge or produce blurry artifacts [S1, unofficial].

## Subtopics

### [Scene NeRF](12484-scene-nerf.md)
Scene NeRF isolates and renders the static background of a scene by training a NeRF exclusively on pixels identified as background via human segmentation masks. It differs from standard NeRF by explicitly excluding dynamic foreground elements (e.g., humans) from the training data and employing an emptiness regularizer to prevent artifacts in empty space.

### [NeRF-OSR](14477-nerf-osr.md)
NeRF-OSR is a neural radiance field approach for outdoor scene relighting that learns a neural representation of scene geometry, diffuse albedo, and illumination-dependent shadows from a set of images capturing the same site from different viewpoints and at different times, enabling simultaneous editing of both the scene’s lighting and viewpoint. Unlike standard NeRF which learns a Lambertian model under fixed lighting, NeRF-OSR decomposes the scene into spatial occupancy, illumination, shadowing, and diffuse albedo reflectance, trained in a self-supervised manner from outdoor data captured from various viewpoints and at different illuminations. It sets itself apart from siblings like [Scene NeRF](12484-scene-nerf.md) by providing explicit intrinsic decomposition (albedo, geometry, lighting) rather than just view-dependent appearance, allowing for physical relighting and albedo editing.

## Operating characteristics
*   **Time Complexity:** Training is slow, often taking several hours to days, as the MLP must be queried for every sample point on every ray for every training image [S1, unofficial].
*   **Memory Complexity:** Memory usage is low, typically a few megabytes, as only the MLP weights are stored [S1, unofficial].
*   **Rendering Speed:** Rendering is slow, taking seconds per image, because it requires evaluating the MLP hundreds of times per pixel and performing volume integration [S1, unofficial].
*   **Strengths:** High-quality novel view synthesis, implicit representation handles complex geometry and fine details well, view-dependent effects (specularities) are naturally modeled.
*   **Limitations:** Slow training and rendering, cannot easily extract a clean surface or mesh, prone to floating artifacts and aliasing when zooming in or out, requires accurate camera poses [S1, unofficial].
*   **When to use:** When high-quality novel views are needed and real-time rendering is not required, or when memory is constrained.
*   **When not to use:** When real-time rendering is required (use [3D Gaussian Splatting](12052-3d-gaussian-splatting.md) instead), or when a clean mesh is needed.

## Background
NeRF was introduced by Mildenhall et al. at SIGGRAPH 2020 [S1, unofficial][S2]. It revolutionized the field of novel view synthesis by demonstrating that a simple MLP could learn a high-quality implicit representation of a scene from unstructured images. The method builds on the principles of volume rendering and positional encoding. It is widely used in virtual reality, film, robotics, mapping, digital humans, ecommerce, and cultural heritage [S1, unofficial].

## Sources
[S1] Gaussian Splatting Explained: Real-Time 3D from Photos (with a Real Demo), https://www.youtube.com/watch?v=HABVlXlRqOs, fetched 2026-10-27 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
