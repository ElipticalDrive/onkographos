# Signed Distance Function (SDF) Generation and Processing


## Overview
This family covers the generation, modification, and utilization of Signed Distance Functions (SDFs) for volume rendering and geometric representation [G]. It solves the problem of converting implicit geometric definitions into renderable volumetric data or interactive geometric primitives [G]. It builds on [Implicit Field Composition](2652-implicit-field-composition.md) by providing the specific mechanisms to generate, smooth, and query the SDF values required for composition [G].

## Shared principles
All methods in this family operate on a scalar field $d(\mathbf{x})$ representing the signed Euclidean distance from a point $\mathbf{x}$ to a zero-level set surface $S = \{\mathbf{x} : d(\mathbf{x}) = 0\}$ [G]. The gradient of the SDF, $\nabla d(\mathbf{x})$, provides the surface normal at the closest point on the surface, satisfying the Eikonal equation $|\nabla d(\mathbf{x})| = 1$ [G].

## Subtopics

### [SDF to Density Conversion](1282-sdf-to-density-conversion.md)
Transforms SDF values into volumetric density $\sigma$ for alpha-compositing. It differs from raw density fields by deriving density from the distance to the surface, enabling open surfaces and varying thickness. Choose this when rendering implicit surfaces defined by SDFs using standard volume rendering techniques.

### [Constrained Jacobi Smoothing for SDFs](1583-constrained-jacobi-smoothing-for-sdfs.md)
Converts binary volumes to smooth SDFs using Jacobi iteration with topological constraints. It differs from Gaussian smoothing by guaranteeing the preservation of the exact binary topology and producing a valid SDF. Choose this when smoothing binary segmentations without altering the underlying shape topology.

### [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md)
Generates and mutates 3D SDF volumes on the GPU using ping-pong buffering and Eikonal propagation. It differs from static generation by supporting real-time user input and ensuring geometric validity after local modifications. Choose this for interactive sculpting or real-time SDF updates.

### [Script-to-Shader SDF Preprocessor](6349-script-to-shader-sdf-preprocessor.md)
Parses a custom script language to generate GLSL code for SDF scene composition. It differs from volumetric methods by operating at the text-generation level, producing shader source code rather than texture data. Choose this for complex procedural implicit surfaces defined in code.

### [Volume SDF](11033-volume-sdf.md)
Rebuilds an SDF from a given isocontour within a volumetric dataset. It differs from ping-pong buffering by operating directly on existing volumetric data to establish distance metrics relative to an internal iso-surface. Choose this when converting scalar volumes (e.g., fog density) into geometrically valid SDFs.

### [RobotSDF](13070-robotsdf.md)
Generates SDFs for multi-link robots conditioned on joint configurations. It differs from static mesh SDFs by explicitly composing individual link SDFs via forward kinematics, providing exact geometric fidelity for any joint state. Choose this for dynamic robot simulation or collision detection.

### [Volume Fairing via Signed Distance Functions](14492-volume-fairing-via-signed-distance-functions.md)
Smooths material boundaries in segmented volumes using 2D Euclidean SDFs (ESDFs). It differs from 3D SDF methods by computing ESDFs only on individual 2D slices for efficiency, allowing interpolation and fairing of misaligned slices. Choose this for medical imaging or slice-based volume correction.

### Multi-Property Implicit Rendering
- **Definition**: Extends the implicit SDF representation to encode spatially variant properties such as color, semantic segmentation, or material alongside 3D geometry [S1].
- **Step**: Train a network to predict both signed distance and the target property (e.g., color) for each 3D location. During rendering, query the network along rays to output both depth and the property value [S1].
- **Parameter**: Latent code size conditioning the network. Example: 512-dim. Significantly smaller than explicit mesh representations (vertices + triangles + texture map), enabling potential model compression [S1].

### [Joint Learning of Implicit Surface and Meshing](6318-joint-learning-of-implicit-surface-and-meshing.md)
- **Definition**: An end-to-end method to optimize both SDF and reconstructed meshes jointly, predicting surface meshes from images.
- **Step**: Architecture: Use a ResNet-18 encoder to produce latent vectors $\lambda$ from input images. Use DeepSDF conditioned on these latent vectors to learn the implicit shape representation. Prediction: Predict SDF values on a regular grid $f_\theta(G)$ and use SDC to reconstruct mesh $M = \text{SDC}(S_G)$ via a forward pass of the SDC network. Training Schedule: Initially train DeepSDF (with image encoder) and SDC independently. Then jointly fine-tune them for 200 epochs.
- **Formula**: Joint Learning Loss: $\mathcal{L}_{\text{SVR}} = \mathcal{L}_{\text{SDF}} + \alpha_3 \mathcal{L}_{\text{SDR}} + \alpha_4 \mathcal{L}_{\text{D}}$, where $\mathcal{L}_{\text{SDF}}$ and $\mathcal{L}_{\text{SDR}}$ reconstruct/regularize the implicit representation, and $\mathcal{L}_{\text{D}}$ trains SDC.
- **Pitfall**: Using the normal consistency loss $\mathcal{L}_N$ during joint training causes instability because the evaluated SDF might be noisy.

### Metric-Phase Fields (MPF)
- **Definition**: MPFs decouple metric proximity from phase/sign information by learning two continuous fields: an unsigned metric field $r(\mathbf{x}) \ge 0$ and a phase field $\theta(\mathbf{x})$ [S2, unofficial].
- The phase field induces a bounded indicator $P(\mathbf{x}) = \tanh(\beta \theta(\mathbf{x}))$ in $(-1, 1)$ [S2, unofficial].
- The composite field is $\phi(\mathbf{x}) = r(\mathbf{x})P(\mathbf{x}) + \theta(\mathbf{x})$ [S2, unofficial].
- The surface is the zero level set $\{\mathbf{x} : \phi(\mathbf{x}) = 0\}$ [S2, unofficial].

### Procedural Synthetic TSDF Generation
- **Step**: Procedurally generate synthetic TSDFs using libsdf, an open-source C++ library implementing simple implicit geometric primitives [S3].
- Randomly pick a sequence of randomly parametrized and oriented shapes from categories: flat surfaces, concave edges, convex edges, concave corners, convex corners, rounded surfaces, and rounded edges [S3].
- Treat convexity and concavity as separate cases for non-oriented surfaces because the sign is inverted in the TSDF [S3].

### Holistic Scene Dynamics Function (HSD)
- **Definition**: A continuous function parameterized by time and object input data, used to procedurally generate unique, dynamic variations (such as affine transformations or material attributes) for 3D objects in a holistic SDF scene [S4].
- **Step**: Calculate the grid cell vector independently from the Instancing operator using the formula: $\text{gridCell} = \lfloor (\mathbf{p} + \text{interval} \times 0.5)/\text{interval} \rfloor$, where $\mathbf{p}$ is a point on the ray and interval is the repeat interval [S4].
- Compute the dynamic variation by evaluating the HSD function with the current time and the calculated grid cell: $\text{variation} = f_{\text{HSD}}(\text{time}, \text{gridCell})$ [S4].
- Apply the Instancing operator to the transformed point: $\text{InstancingOp}(\mathbf{p}, \text{interval})$ [S4].
- **Formula**: $f_{\text{HSD}} : D \times t \rightarrow V$, where $V$ is the dynamics variation for a given 3D object, $D$ is the object input data, and $t$ is the current simulation time [S4].
- **Parameter**: Input data $D$: Typically the object's world space position or its grid cell ID. Implementation methods: Can be a simple pseudorandom noise generator, trigonometric functions, Fractional Brownian Motions, or Vector Fields [S4].

### Auto-Encoder TSDF Compression
- **Definition**: A nonlinear compression method using an artificial neural network with a bottleneck layer to learn a low-dimensional feature representation of TSDF voxel blocks [S3].
- **Step**: Construct a fully connected neural network with symmetric encoder and decoder layers. Insert an intermediary hidden layer (bottleneck) with dimensionality smaller than the input and output layers. This layer serves as the code/feature layer. Train the network by setting the desired output equal to the input and minimizing the reconstruction error using back-propagation [S3].
- **Formula**: The descriptor $d_p$ is computed by chaining affine transformations and activation functions: $d_p = f(L_k \dots f(L_2 f(L_1 \mathbf{p} + \mathbf{b}_1) + \mathbf{b}_2) \dots + \mathbf{b}_k)$, where $L_i$ are layer weights, $\mathbf{b}_i$ are biases, and $f$ is an element-wise activation function [S3].

## Background
Signed Distance Functions have been a cornerstone of geometric modeling and rendering for decades [G]. The family article [S1] cites "DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing" (2020) as a key reference for differentiable rendering of SDFs. The use of auto-encoders for TSDF compression is detailed in [S3]. The joint learning of implicit surfaces and meshing is presented in [S1]. The handling of sign-ambiguous geometry via Metric-Phase Fields is introduced in [S2, unofficial]. Procedural TSDF generation for training data is described in [S3]. Dynamic scene variations using HSD are covered in [S4].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Volume Rendering with Metric--Phase Fields for Thin Structures and Sign-Ambiguous Geometry, https://doi.org/10.48550/arxiv.2610.13088, fetched 2026-10-26 (unofficial)
[S3] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
[S4] Real-time visualization of Dynamic Unlimited Objects Instancing, https://doi.org/10.24132/csrn.2018.2802.4, fetched 2026-10-26
