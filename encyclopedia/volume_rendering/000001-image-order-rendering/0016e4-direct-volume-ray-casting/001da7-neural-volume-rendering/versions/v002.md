# Neural volume rendering

Also called: Neural Radiance Fields (NeRF), Implicit Neural Representations for Volume Rendering

## Overview
Neural volume rendering synthesizes novel views of 3D scenes by replacing the explicit scalar voxel grid of traditional direct volume ray casting with a continuous implicit function represented by a neural network [S1, S14, unofficial]. It solves the problem of rendering complex, unstructured, or sparse data (such as dynamic scenes or unstructured meshes) by learning a mapping from 3D spatial coordinates (and optionally view directions) to volume density and color [S1, S14, unofficial]. It distinguishes itself from traditional methods by using differentiable rendering to optimize the network parameters from 2D images, allowing for end-to-end training of the scene representation [S1, S14, unofficial].

## Shared principles
The method builds on [Direct volume ray casting](19-direct-volume-ray-casting.md) but replaces the discrete volume texture lookup with a neural network query. The core mechanism follows the differentiable volume rendering integral [S1, S14, unofficial]:

1.  **Ray Sampling:** For each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is generated. Sample points $\mathbf{p}_i$ are sampled along the ray segment $[t_{start}, t_{end}]$ [S1, S14, unofficial].
2.  **Neural Query:** At each sample point $\mathbf{p}_i$, a neural network (typically an MLP) predicts the raw density $\sigma_i$ and color $\mathbf{c}_i$ (and optionally view-dependent color) [S1, S14, unofficial].
3.  **Alpha Conversion:** The raw density $\sigma_i$ is converted to opacity $\alpha_i$ using the discretized volume rendering equation:
    $$ \alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i) $$
    where $\delta_i$ is the distance between sample points [S1, S14, unofficial].
4.  **Differentiable Compositing:** The final pixel color is computed via front-to-back compositing, which is differentiable with respect to the network outputs, allowing gradients to flow back to the network weights during training [S1, S14, unofficial]:
    $$ C = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$

## Subtopics

### [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md)
A differentiable rendering method that synthesizes novel views by querying a neural radiance field (MLP) along camera rays and integrating color and density using classical volume rendering. It distinguishes itself from single-stage sampling by employing a two-tier strategy: a coarse network samples uniformly (or with stratified jitter) to estimate the density distribution, and a fine network samples more densely in high-density regions identified by the coarse weights to refine the estimate. This approach improves rendering quality and efficiency by concentrating computational effort on opaque regions, addressing the inefficiency of uniform sampling in empty space.

### [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md)
Iterative Ray Marching with Neural Volume Inference is a CUDA-based rendering method that splits the volume integration of a neural radiance field into multiple kernel launches to handle rays that do not fully converge (reach alpha $\ge 1$) in a single pass. It distinguishes itself from in-shader neural volume ray marching by decoupling neural network inference from shading logic, using a counter-based compaction mechanism to track and process only active rays across iterations.

### [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md)
In-Shader Neural Volume Ray Marching is a direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point, rather than pre-decoding the volume into a 3D texture or using sample-streaming. It supports multiple shading modes including gradient shading and single-shade heuristics.

### [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md)
The DVGO forward pass renders a scene by querying a learnable 3D voxel grid of density values and aggregating them along rays using volume rendering principles. It differs from NeRF-based methods by replacing the neural network density query with direct lookup and interpolation from a sparse or dense voxel grid, enabling faster training and inference by leveraging GPU texture hardware. It builds on [Neural volume rendering](1576-neural-volume-rendering.md) by discretizing the volume integral over a structured grid rather than a continuous implicit field.

### [MLP with Positional Encoding](2416-mlp-with-positional-encoding.md)
A neural rendering module that predicts RGB color from input features, view directions, and spatial positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to both position and view direction.

### [Volumetric Primitive Radiance Field Integrator](2536-volumetric-primitive-radiance-field-integrator.md)
The Volumetric Primitive Radiance Field Integrator is a path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation. Unlike its sibling [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md), which relies on dense MLP evaluations along fixed or adaptive ray steps, this method evaluates closed-form or semi-analytical transmittance functions for sparse geometric primitives, enabling efficient acceleration structures and physical-based light transport integration.

### [Neural Volume Raycasting Debug Visualization](2646-neural-volume-raycasting-debug-visualization.md)
This method provides a diagnostic visualization pipeline for neural volume models by rendering raw sample points as instanced spheres rather than compositing them into a final image. It visualizes the spatial distribution of samples and their predicted radiance, filtered by an opacity threshold, to debug model convergence and sampling density. Unlike standard neural volume rendering which outputs a pixel-perfect image via front-to-back compositing, this method isolates individual samples for inspection.

### [Hierarchical Sampling](4865-hierarchical-sampling.md)
Hierarchical sampling is a two-stage sampling strategy for direct volume ray casting that uses a coarse estimate of the scene's density to guide the placement of fine samples via importance sampling. It solves the problem of inefficient uniform sampling by concentrating computational effort in regions of high opacity or high-frequency detail, thereby reducing aliasing and noise while maintaining image quality. Unlike uniform sampling, which treats all segments of a ray equally, hierarchical sampling adapts the sample density based on the estimated contribution of each segment, often achieving 2-3x efficiency gains over uniform sampling without visible artifacts.

### [Neural Network Volume Classification](5271-neural-network-volume-classification.md)
Neural Network Volume Classification is an interactive method that partitions volume data into material classes by training an artificial neural network on user-painted samples, mapping voxel properties to an uncertainty value that serves as the transfer function for rendering. It eliminates the need for manual transfer function definition by replacing the transfer function space with a direct volume-space painting interface. Unlike traditional methods that require defining curves in derived intensity space, this approach learns a high-dimensional classification function from sparse user input, providing real-time visual feedback to guide the training process.

### [Neural Network Volume Rendering via Fragment Program](6331-neural-network-volume-rendering-via-fragment-program.md)
This method renders volumetric data by executing a neural network classifier directly within the GPU's fragment shader during rasterization. It solves the problem of real-time interactive classification and visualization by offloading the computational burden of the neural network from the CPU to the graphics hardware. Unlike traditional ray casting which samples a 3D texture at discrete points along rays, this approach renders a stack of view-aligned polygons that sample the volume, using the fragment program to perform the classification logic for each pixel. It is distinguished from other neural volume rendering methods by its reliance on fixed-function rasterization of polygons rather than explicit ray marching or implicit field evaluation, allowing for interactive feedback during training.

#### Neural Direct Volume Rendering
- **Definition:** A novel multiscale rendering architecture inspired by UNet and VNet, featuring a multiscale volumetric encoding part and a multiscale 2D decoder connected by DeepDVR modules replacing skip connections [S1, unofficial]. It is an encoder-decoder architecture inspired by VNet and U-Net, using a DVRUnit in skip connections to project from 3D to 2D feature space, combining lower-resolution multichannel images in U-Net decoder blocks [S1, unofficial].
- **Steps:** Connect the volumetric multiscale encoder part of a VNet with the upsampling decoder part of a U-Net by inserting a DVRUnit into the skip connections [S1, unofficial]. Each level performs a volume raymarching network similar to VNetL16-17, rendering a potentially spatially downsampled volume in a latent color space at a reduced image resolution [S1, unofficial]. Combine the lower-resolution multichannel images in the U-Net decoder blocks to produce the final 2D image at the desired resolution [S1, unofficial].
- **Comparison:** Unlike RenderNet, DVRNet extracts features in object space (independent of camera parameters), uses explicit occlusion modeling via absorption/alpha blending, and avoids excessive downsampling via skip connections [S1, unofficial].

## Background
Neural volume rendering emerged from the field of Neural Radiance Fields (NeRF), introduced by Mildenhall et al. in 2020 [S1, unofficial]. The method was developed to address the limitations of explicit 3D representations (like meshes and voxels) in novel view synthesis, particularly for complex, unstructured scenes. It builds on the concept of implicit neural representations, where a scene is defined by a continuous function parameterized by a neural network [S1, unofficial]. The key innovation was the integration of differentiable volume rendering, allowing the network to be trained directly from 2D images using gradient descent [S1, unofficial]. Subsequent developments, such as DVGO and Instant NGP, have focused on improving training and inference speed by using voxel grids and hash encodings instead of pure MLPs [S1, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
