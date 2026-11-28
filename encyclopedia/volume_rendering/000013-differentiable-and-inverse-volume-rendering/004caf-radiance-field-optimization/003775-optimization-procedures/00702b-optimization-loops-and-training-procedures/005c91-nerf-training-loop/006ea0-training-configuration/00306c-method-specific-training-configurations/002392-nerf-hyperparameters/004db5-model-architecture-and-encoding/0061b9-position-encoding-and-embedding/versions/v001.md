# Position Encoding and Embedding

Also called: Positional Encoding, Ray-Space Embedding, Skeleton-Relative Encoding

## Overview
Position encoding and embedding methods transform low-dimensional input coordinates (position $\mathbf{x}$ and view direction $\mathbf{d}$) into high-dimensional feature spaces to enable neural networks to approximate high-frequency functions, thereby overcoming the spectral bias inherent in standard MLPs [S2]. This family distinguishes itself from generic coordinate mapping by explicitly designing the frequency spectrum, spatial reference frames, or ray-space parameterizations to address specific challenges such as few-shot overfitting, sparse view interpolation, and articulated motion [S1, unofficial][S2].

## Shared principles
The core mechanism involves transforming low-dimensional input coordinates into a high-dimensional feature space using sinusoidal positional encoding, which is then processed by a fully-connected MLP to predict radiance field properties.

**Positional Encoding**
The input coordinates $\mathbf{x} \in \mathbb{R}^3$ (position) and $\mathbf{d} \in \mathbb{R}^3$ (view direction) are mapped to higher dimensions using a hierarchical positional encoding function $\gamma(\cdot)$ [S1, unofficial]. For a scalar input $p$, the encoding is defined as:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
where $L$ is the number of frequency bands . This function is applied independently to each of the three components of the position vector and the direction vector [S1, unofficial].

**Network Architecture**
The radiance field is represented by an MLP that takes the encoded position and direction as input. The architecture typically consists of two branches:
1.  **Density Branch:** Processes the encoded position features to predict volume density $\sigma$.
2.  **Color Branch:** Processes the encoded direction features concatenated with intermediate features from the density branch to predict RGB color $\mathbf{c}$ [S1, unofficial].

**Ray Sampling**
Rendering requires sampling points along camera rays. A ray is parameterized as $r(\lambda) = o_{world} + \lambda d_{world}$, where $o_{world}$ is the world-space origin and $d_{world}$ is the world-space direction [S1, unofficial]. The direction is computed from camera intrinsics and extrinsics: $d_{cam} = (\frac{i - c_x}{f_x}, \frac{j - c_y}{f_y}, 1)$, $d_{world} = R d_{cam}$, and $o_{world} = t$ [S1, unofficial]. Position samples $x$ are generated along the ray within a defined near/far range [S1, unofficial].

## Subtopics

### [Frequency-Modulated Position Encoding](15897-frequency-modulated-position-encoding.md)
Frequency-Modulated Position Encoding is a positional encoding scheme used in point-based radiance field optimization to map 3D spatial coordinates and view directions into a high-dimensional feature space. It distinguishes itself from standard NeRF encoding by using a geometric progression of frequencies with a base of $2^{0.5}$ rather than $2^1$, providing finer-grained basis function support for frequency modulation. This method is specifically designed for point cloud representations where points are rasterized into disks, requiring careful handling of sparsity and edge artifacts.

### [Neural Light Fields with Ray-Space Embedding](17616-neural-light-fields-with-ray-space-embedding.md)
Neural Light Fields with Ray-Space Embedding is a view synthesis method that represents a scene as a function mapping 4D ray parameters directly to integrated radiance, enabling rendering with a single network evaluation per pixel. Unlike NeRF, which models a 5D radiance field and requires hundreds of samples per ray, this method assumes radiance is constant along rays (valid when the convex hull of the scene does not contain viewpoints), allowing it to bypass volumetric integration. It distinguishes itself from standard coordinate-based MLPs by using a ray-space embedding network to map input rays into an intermediate latent space, facilitating the registration of rays observing the same 3D points and improving interpolation quality for unobserved views.

### [Ray-Space Embedding Network with Local Affine Transformation](18128-ray-space-embedding-network-with-local-affine-transformation.md)
This method represents a neural light field by predicting local affine transformations for color computation, enabling view-dependent effects and multi-view consistency without explicit geometry. It sets itself apart from standard NeRF by embedding ray-space coordinates into a feature space that is locally linearized via affine transformations, allowing for efficient interpolation in sparse light field settings where standard MLPs struggle with high-frequency view dependence.

### [Skeleton-Relative Encoding](18148-skeleton-relative-encoding.md)
Skeleton-Relative Encoding transforms 3D query positions and view directions into local bone-relative coordinate systems before processing them through a neural radiance field, explicitly incorporating domain knowledge of human body part linkage. Unlike standard NeRF which treats space as continuous and unstructured, this method reparameterizes the input space to respect the piece-wise rigidity of articulated motion, making it suitable for dynamic human avatars where global coordinate systems are insufficient. It sets itself apart from siblings like [Mip-NeRF 360 MLP Architecture](14098-mip-nerf-360-mlp-architecture.md) by replacing global positional encoding with bone-centric relative metrics, and from [SUDS](14964-suds.md) by relying on explicit skeleton transforms rather than implicit deformation fields.

### [Position Encoding](18204-position-encoding.md)
Position Encoding (PE) is a feature mapping technique that transforms low-frequency coordinate representations into high-frequency information using sine and cosine functions, thereby increasing dimensionality to help deep networks approximate high-frequency functions and prevent the loss of high-frequency spatial information. It sets itself apart from its siblings by serving as the foundational explicit encoding scheme for NeRF, whereas alternatives like Frequency-Modulated Position Encoding or Random Offset Annealed Fourier Encoding introduce stochasticity, modulation, or local affine transformations to address specific limitations such as spectral bias or domain shift.

### [Random Offset Annealed Fourier Encoding](18686-random-offset-annealed-fourier-encoding.md)
Random Offset Annealed Fourier Encoding (ROAFE) is a positional encoding scheme for 3D coordinates that combines Fourier basis functions with random Gaussian offsets and an annealing schedule to improve the optimization landscape for neural radiance fields. It addresses the axis-alignment artifacts and striping issues inherent in standard NeRF encoding by breaking symmetry through random shifts, while using a truncated Hann window to gradually introduce high-frequency details. It is primarily used in the SAMURAI framework.

### Low-Frequency Only Neural Radiance Field
- **Definition:** A neural field that uses only low-frequency positional encoding to model rough geometry. It is designed to interpolate smoothly and avoid overfitting in few-shot settings, serving as a regularizer for a subsequent high-frequency field [S1, unofficial].
- **Step:** The framework consists of two stages: first, train a low-frequency only ray-based field to predict depth (not color) for each ray; second, use this field to supervise/regularize a high-frequency added point-based field for fine details [S1, unofficial]. During inference, the low-frequency only ray-based field predicts a depth value for each ray instead of colors, allowing it to be trained easier by fitting a low-frequency smooth signal [S1, unofficial].
- **Layout:** The low-frequency field is ray-based (adapted from LFN [43]), taking ray parameters as input. The high-frequency field is point-based, taking 3D coordinates as input [S1, unofficial].
- **Comparison:** Unlike standard NeRF which uses high-frequency PE for both geometry and appearance, this method separates rough geometry (low-freq, ray-based) from fine details (high-freq, point-based) [S1, unofficial].

### NeRF with mipNeRF-style sampling and hashtable encoding
- **Definition:** A NeRF training configuration that integrates mipNeRF-style sampling with a hashtable-based positional encoding to improve rendering quality at lower resolutions and reduce stair-stepping artifacts [S2].
- **Step:** 1. Append the diagonalised variance to the sample position. 2. Feed blurred training samples into the network. 3. Use swish activations instead of relu. 4. Add a final layernorm before the last MLP layer [S2].
- **Parameter:** hashtable size: 220; hashtable levels: 12; samples: 64 × 64; MLP depth: 2; MLP width: 64; MLP width viewdir: 32; Proposal MLP depth: 2; Proposal MLP width: 64; sample dilation: 0.001; activation type: swish; density activation: squareplus [75]; density bias: −5; batch size: 16384; weight decay: 5 × 10−5; optimizer: adamw; lr init: 2 × 10−3; lr final: 4 × 10−5; warmup steps: 2048; max steps: 40000; charb padding: 0.001; distortion loss mult: 0.01; blur σmax: 12.0 [S2].
- **Pitfall:** The ceiling is not well captured by the NeRF due to a lack of data in the input video, which can cause policy failures if the robot looks upward [S2].
- **Usage:** Used in the NeRF2Real pipeline for sim2real transfer of vision-guided bipedal motion skills [S2].

### High-Frequency Added Neural Radiance Field
- **Definition:** A neural field equipped with high-frequency positional encoding that models fine details and extrapolates periodic patterns in unobserved regions, regularized by the low-frequency field [S1, unofficial].
- **Step:** After the low-frequency field is trained, the high-frequency field is trained to fit the details. The low-frequency field's output (geometry/depth) is used to supervise the high-frequency field [S1, unofficial].
- **Pitfall:** Blindly increasing the frequency of Positional Encoding (P.E.) leads to overfitting when the reconstruction problem is highly underconstrained (e.g., few-shot images), resulting in erroneous geometry [S1, unofficial].
- **Comparison:** Low-frequency only fields generate smooth interpolation but fail to extrapolate high-frequency periodic signals. High-frequency added fields can adapt to periodic patterns but bring high-frequency interpolation artifacts if used alone [S1, unofficial].

## Background
Positional encoding was introduced as a critical factor in the success of Neural Radiance Fields (NeRF) [29], enabling coordinate-based MLPs to recover scene details by mapping low-dimensional coordinates to high-dimensional space [S1, unofficial]. The standard hierarchical sinusoidal encoding allows networks to approximate high-frequency functions, addressing the spectral bias where MLPs naturally learn low-frequency functions first [S2]. Subsequent developments have focused on modifying this encoding to handle specific constraints, such as few-shot view synthesis [S1, unofficial], sparse light field interpolation [S1, unofficial], and dynamic articulated scenes [S1, unofficial].

## Sources
[S1] Harnessing Low-Frequency Neural Fields for Few-Shot View Synthesis, https://doi.org/10.48550/arxiv.2303.08370, fetched 2026-10-29 (unofficial)
[S2] NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields, https://doi.org/10.1109/icra48891.2023.10161544, fetched 2026-10-29
