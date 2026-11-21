# Neural volume rendering

Also called: SRN Rendering

## Overview
Neural volume rendering is a family of differentiable rendering methods that synthesize novel views of 3D scenes by querying a neural network or neural representation along camera rays and integrating the outputs using classical volume rendering techniques [S2, unofficial]. It solves the problem of representing complex, high-dimensional volumetric data or implicit scenes without explicit geometric meshes or large voxel grids, enabling view-dependent appearance and high-fidelity reconstruction from 2D images [S1]. This family sets itself apart from traditional direct volume ray casting by replacing the lookup of scalar voxel values with a neural function that predicts density and color, and from sibling methods by relying on specific architectural choices such as hierarchical sampling, direct voxel grids, or in-shader evaluation [S1][S2, unofficial].

## Shared principles
All neural volume rendering methods share the following core mechanism:

1.  **Neural Representation**: The 3D scene is represented by a neural network (typically a Multi-Layer Perceptron, MLP) or a neural data structure (e.g., hash grids, feature grids) parameterized by weights $\theta$. This network maps spatial coordinates $\mathbf{x}$ (and optionally viewing directions $\mathbf{d}$) to raw density $\tilde{\sigma}$ and color $\tilde{\mathbf{c}}$ [S1][S2, unofficial].
2.  **Differentiable Rendering**: The rendering process is fully differentiable with respect to the network parameters $\theta$. This allows the method to be trained using gradient descent on the difference between rendered images and ground truth images [S2, unofficial].
3.  **Volume Rendering Equation**: The final pixel color is computed by integrating the neural network's outputs along the ray using the front-to-back volume rendering equation, identical to the parent method [Direct volume ray casting](19-direct-volume-ray-casting.md) [S2, unofficial]:
    $$
    \mathbf{C}_{out} = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\mathbf{c}_i$ and $\alpha_i$ are derived from the neural network's output $\tilde{\mathbf{c}}_i$ and $\tilde{\sigma}_i$ via activation functions (e.g., sigmoid for color, softplus for density) and the Beer-Lambert law [S1][S2, unofficial].
4.  **Positional Encoding**: To enable the MLP to capture high-frequency details, input coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ are typically mapped to higher-dimensional spaces using sinusoidal positional encoding $\gamma(\cdot)$ before being fed into the network [S2, unofficial].

## Subtopics

### [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md)
A differentiable rendering method that synthesizes novel views by querying a neural radiance field along camera rays and integrating outputs using classical volume rendering. It sets itself apart from siblings like [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md) and [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md) by relying on a simple, differentiable hierarchical sampling of coarse alpha weights without auxiliary acceleration structures. Choose it for high-fidelity novel view synthesis where view-dependent appearance is critical.

### [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md)
A GPU-accelerated technique that splits neural volume evaluation along a ray into multiple kernel launches to handle rays that do not fully converge in a single pass. It distinguishes itself from single-pass methods by using a counter-based compaction mechanism to track active rays between launches, allowing dynamic handling of varying ray depths and complex occlusion without fixed iteration limits. Choose it when rendering complex scenes with deep occlusion or varying density.

### [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md)
A direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point during ray marching, rather than pre-decoding the volume into a 3D texture or using sample-streaming. It supports multiple shading modes, including gradient shading and single-shade heuristics, allowing for real-time interactive visualization of neural radiance fields. Choose it for interactive applications requiring on-the-fly neural inference.

### [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md)
Renders a 3D scene represented by a direct voxel grid of density and color features into a 2D image by querying the grid along camera rays and compositing the results. It differs from standard NeRF by replacing the multi-layer perceptron (MLP) with a direct lookup in a 3D texture (voxel grid), enabling significantly faster training and inference while maintaining view-dependent color through auxiliary feature vectors. Choose it for fast training and inference of large-scale scenes.

### [Volumetric Primitive Radiance Field Integrator](2536-volumetric-primitive-radiance-field-integrator.md)
A path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation. It distinguishes itself from its sibling, the Volumetric Primitive Tomography Integrator, by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering. Choose it for physically-based rendering of participating media.

### [Neural Volume Raycasting Debug Visualization](2646-neural-volume-raycasting-debug-visualization.md)
Provides a diagnostic visualization pipeline for neural volume models by rendering individual sample points as instanced spheres, colored by the network's predicted radiance and filtered by an opacity threshold. It differs from standard rendering methods by bypassing front-to-back compositing to inspect the raw distribution and magnitude of neural network outputs in 3D space, serving as a tool to verify that the implicit representation is capturing scene geometry and appearance correctly. Choose it for debugging and analyzing neural volume training.

### [Neural Network Volume Rendering via Fragment Program](6331-neural-network-volume-rendering-via-fragment-program.md)
Renders volumetric data by mapping a 3D texture to a stack of view-aligned polygons and executing a neural network classifier directly within the fragment shader for each pixel. It solves the problem of interactive, high-dimensional classification of volume data by offloading the inference logic to the GPU, allowing users to evaluate training results in real-time. Unlike standard volume rendering which uses fixed transfer functions, this approach uses a learned neural network to determine opacity and color, and it differs from [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md) by relying on rasterization of textured polygons rather than explicit ray marching through the volume. Choose it for interactive classification and visualization of scalar fields.

### [Denoising via Neural Auxiliary Features](10536-denoising-via-neural-auxiliary-features.md)
Learns to reconstruct high-quality images from noisy, low-sample path-traced volume renderings by exploiting spatial and volumetric correlations, trading unbiased Monte Carlo convergence for biased neural reconstruction. It is distinguished from surface-only denoisers by its requirement for specialized volumetric auxiliary features (such as optical thickness and scattering normals) because standard features like depth and albedo are ill-defined or unavailable throughout participating media. Choose it for accelerating Monte Carlo path tracing in volumetric scenes.

### [Hardware Accelerators for Neural Volume Rendering](13789-hardware-accelerators-for-neural-volume-rendering.md)

### Neuralangelo
- **Definition**: A framework for high-fidelity 3D surface reconstruction from RGB images using neural volume rendering, combining multi-resolution 3D hash grids with neural surface rendering without auxiliary data [S3]. It uses numerical gradients to compute higher-order derivatives (e.g., surface normals) as a smoothing operation to stabilize optimization [S3]. It uses coarse-to-fine optimization on hash grids controlling different levels of details [S3].
- **Steps**: Samples 3D locations along camera view directions [S3]. Encodes the sampled positions using a multi-resolution hash encoding [S3]. Feeds encoded features into an SDF MLP and a color MLP [S3]. Composites images using SDF-based volume rendering [S3].
- **Comparison**: Unlike concurrent work which uses a displacement network to correct shape, Neuralangelo uses hierarchical hash grids and controls detail levels based on analysis of higher-order derivatives [S3].

### Neural Rays (NeuRay)
- **Definition**: An approach that uses cost volumes estimated from input views via multi-view stereo algorithms, processed by a CNN to create feature maps, from which visibility and local features are extracted during volume rendering [S4].
- **Steps**: A CNN is used to create feature maps G from the estimated cost volumes [S4]. During volume rendering, visibility and local features are extracted from these features and processed using MLPs to extract color and alpha [S4].
- **Formula**: Visibility is computed as a cumulative density function written as a weighted sum of sigmoid functions [S4].

### Real-time rendering via mesh extraction and feature aggregation
- **Definition**: A rendering acceleration technique that replaces slow neural volume rendering with a mesh-based approach by pre-computing appearance features and using a compact shape model for real-time frame generation [S5, unofficial].
- **Steps**: Use the marching cubes algorithm to extract a mesh as an iso-surface of the SDF [S5, unofficial]. Store the features computed from the set of input images by the encoder as well as their associated depth maps [S5, unofficial]. At render time, aggregate these features based on the target viewing direction [S5, unofficial]. Evaluate the decoder on the aggregated features for each frame [S5, unofficial].
- **Parameter**: Decoder evaluation time: 8.7ms for DTU images and network architecture [S5, unofficial]. Feature aggregation time: 22.3ms for DTU images and network architecture [S5, unofficial].
- **Number**: Total rendering time per frame is approximately 31ms (8.7ms + 22.3ms), enabling real-time rates [S5, unofficial].
- **Comparison**: Unlike neural volume rendering methods which cannot generate frames at nearly a real-time rate, this surface-based method renders in real time [S5, unofficial].

### [Volumetric Ambient Occlusion prediction](10480-volumetric-ambient-occlusion-prediction.md)
Deep Volumetric Ambient Occlusion (DVAO) is a deep learning-based technique that predicts per-voxel ambient occlusion (AO) volumes for direct volume rendering (DVR). It solves the problem of computing realistic volumetric illumination efficiently by training a 3D convolutional neural network to map input scalar field data and transfer functions to an AO volume. Unlike traditional numerical integration methods, DVAO predicts the AO field once, which is then sampled via texture lookup during rendering, supporting real-time interaction when the transfer function changes. It is distinguished from sibling methods by its use of a global encoder-decoder architecture that incorporates transfer function information implicitly or explicitly, rather than computing occlusion locally per ray.

## Background
Neural volume rendering builds on the principles of differentiable rendering and neural implicit representations. The core mechanism involves querying a neural network along camera rays and integrating the outputs using classical volume rendering techniques [S2, unofficial]. The field has expanded to include various representations such as Direct Voxel Grid Optimization (DVGO) [S1][S2, unofficial] and hash-grid-based methods [S3]. It is applied in computer vision for 3D reconstruction and novel view synthesis [S3, S4].

## Sources
[S1] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
[S2] Taichi NeRF (Part 1): Develop and Deploy Instant NGP without writing CUDA | Taichi Docs, https://docs.taichi-lang.org/blog/taichi-instant-ngp, fetched 2026-10-29 (unofficial)
[S3] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S4] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S5] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
