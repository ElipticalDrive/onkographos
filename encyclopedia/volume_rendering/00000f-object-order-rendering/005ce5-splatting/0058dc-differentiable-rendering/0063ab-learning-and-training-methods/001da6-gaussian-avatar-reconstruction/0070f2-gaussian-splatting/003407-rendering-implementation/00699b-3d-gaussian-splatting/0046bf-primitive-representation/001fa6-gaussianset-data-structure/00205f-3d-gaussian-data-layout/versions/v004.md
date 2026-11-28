# 3D Gaussian Data Layout

Also called: 3DGS Parameter Storage, Gaussian Struct Definition, GaussianModel, GaussianModel PyTorch Module

## Overview
The 3D Gaussian Data Layout defines the memory organization and field semantics for storing 3D Gaussian primitives within differentiable volume rendering frameworks, specifically 3D Gaussian Splatting (3DGS). It solves the problem of efficiently representing high-dimensional geometric and appearance attributes—position, scale, rotation, opacity, and spherical harmonics coefficients—on the GPU for batched rendering and gradient-based optimization. It sets itself apart from sibling methods like [Gaussian Data Packing](3257-gaussian-data-packing.md) by defining the logical struct fields and their semantic types rather than the low-level bit-packing or serialization format, and builds on [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md) which specifies the contiguous memory arrangement of these fields.

## How it works
The layout supports two primary implementation paradigms: a raw memory struct layout (common in C++/Rust renderers) and a PyTorch tensor-based parameter storage (common in optimization pipelines). Both store the same logical attributes but differ in activation functions and storage formats.

### Inputs and Outputs
*   **Input**: A set of $N$ 3D Gaussians, each defined by geometric parameters (mean, scale, rotation) and appearance parameters (opacity, spherical harmonics coefficients).
*   **Output**: A contiguous memory block or tensor structure ready for rasterization or gradient computation.

### Data Structures and Layouts

#### 1. Raw Memory Struct (C++/Rust Style)
Defined in [S1, unofficial], this layout stores precomputed values and raw parameters.

**Fields:**
*   `position`: `na::Vector3<f32>` (x, y, z). 12 bytes.
*   `scale`: `na::Vector3<f32>` (scale_0, scale_1, scale_2). 12 bytes.
*   `opacity`: `f32`. 4 bytes.
*   `rotation`: `na::Quaternion<f32>` (w, x, y, z). 16 bytes.
*   `sh`: `na::SVector<f32, 48>`. 192 bytes. Stores DC coefficients (3) and rest coefficients (45) for degree 3 spherical harmonics.
*   `cov3d`: `na::Matrix3<f32>`. 36 bytes. Precomputed 3x3 symmetric covariance matrix.

**Total Size**: 272 bytes per Gaussian (excluding padding).

#### 2. PyTorch Parameter Storage (Optimization Style)
Defined in [S3, unofficial], this layout stores learnable parameters in log-space or log-odds to ensure valid ranges during gradient descent.

**Internal Parameters (per Gaussian $i$):**
*   `_xyz`: Tensor of shape $(N, 3)$, float32. Raw positions.
*   `_features_dc`: Tensor of shape $(N, 1, 3)$, float32. 0th-degree spherical harmonics (DC) colors.
*   `_scaling`: Tensor of shape $(N, 3)$, float32. Log-scales.
*   `_rotation`: Tensor of shape $(N, 4)$, float32. Quaternions (not necessarily unit length during training).
*   `_opacity`: Tensor of shape $(N, 1)$, float32. Log-odds opacity.

**Derived Properties (Computed on Access):**
*   **Scaling**: $s_i = \text{clamp}(\exp(\text{_scaling}_i), \text{max}=10.0)$ [S3, unofficial]. Converts log-space to linear space with an upper bound.
*   **Rotation**: $q_i = \text{normalize}(\text{_rotation}_i)$ [S3, unofficial]. Ensures unit-length quaternions.
*   **Opacity**: $\alpha_i = \sigma(\text{_opacity}_i) = \frac{1}{1 + e^{-\text{_opacity}_i}}$ [S3, unofficial]. Converts log-odds to probability space $[0, 1]$.

### Conversion Steps

#### List to Gaussian (Serialization/Interchange)
Used for saving or exchanging data [S2, unofficial]. The Gaussian is flattened into a single vector.

1.  **Extract Attributes**:
    *   `centroid`: Mean position $(3)$
    *   `scale`: Linear scale $(3)$
    *   `rotation`: Quaternion $(4)$
    *   `opacity`: Scalar $(1)$
    *   `feat_dc`: DC SH coefficients $(3)$
    *   `feat_extra`: Extra SH coefficients (flattened) $(3 \times (D^2-1))$ where $D$ is the SH degree.
2.  **Concatenate**:
    $$ \text{data} = [\text{centroid}, \text{scale}, \text{rotation}, \text{opacity}, \text{feat\_dc}, \text{feat\_extra}] $$
3.  **Result**: A 1D array of size $11 + 3(D^2-1)$. For degree 3 ($D=3$), size is $11 + 24 = 35$.

#### Gaussian to List (Deserialization)
1.  **Parse Indices**:
    *   `centroid` = `data[0:3]`
    *   `scale` = `data[3:6]`
    *   `rotation` = `data[6:10]`
    *   `opacity` = `data[10]`
    *   `feat_dc` = `data[11:14]`
    *   `feat_extra` = `data[14:]`
2.  **Reconstruct**: Create Gaussian object from parsed fields.

#### Points to Gaussian (Initialization from PCD)
Used to initialize Gaussians from a point cloud [S2, unofficial].

1.  **Centroid**: Set to point position $[0,0,0]$ relative to local frame or global xyz.
2.  **Scale**: Fit ellipsoid via PCA to get principal axes $a,b,c$. Set `scale` = $\log(a,b,c)$.
3.  **Rotation**: Compute rotation matrix $R$ from PCA eigenvectors. Convert to quaternion.
4.  **Opacity**: Compute mean opacity of surrounding points. Set `opacity` = $\text{sigmoid}^{-1}(\text{mean})$.
5.  **SH**: Fit spherical harmonics to colors. `feat_dc` = $sh[:,0]$, `feat_extra` = $sh[:,1:]$.

#### Gaussian to Points (Sampling)
Used for visualization or loss computation [S2, unofficial].

1.  **Activate Parameters**:
    *   $s = \text{clamp}(\exp(\text{scale}), 10.0)$
    *   $q = \text{normalize}(\text{rotation})$
    *   $\alpha = \sigma(\text{opacity})$
2.  **Transform**: Compute rotation matrix $R$ from $q$.
3.  **Sample**: Sample points $(u,v)$ on unit sphere. Transform to local xyz: $p_{local} = R \cdot \text{diag}(s) \cdot p_{sphere}$.
4.  **Color**: Evaluate SH at direction $p_{local}$ using `feat_dc` and `feat_extra`.

### Parameters
*   **SH Degree ($D$)**: Determines number of coefficients.
    *   Degree 0: 3 coefficients (DC only).
    *   Degree 1: 12 coefficients.
    *   Degree 2: 27 coefficients.
    *   Degree 3: 48 coefficients [S1, unofficial].
*   **Max Scale**: 10.0 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Quaternion Normalization**: In PyTorch storage, `_rotation` is not normalized during storage to allow gradients to flow, but must be normalized before use in covariance computation or rotation matrices [S3, unofficial].
*   **Scale Clamping**: Exponential scaling can explode; clamping to 10.0 prevents numerical instability [S3, unofficial].
*   **SH Dimension Mismatch**: The `feat_extra` size depends on the max SH degree. If the renderer expects degree 3 but the data contains degree 1, padding or truncation is required.

## Subtopics

### [Gaussian Data Structure](2273-gaussian-data-structure.md)
The Gaussian Data Structure defines the memory layout and field organization for 3D Gaussian primitives used in differentiable volume rendering, specifically within the 3D Gaussian Splatting (3DGS) framework. It solves the problem of efficiently storing and accessing high-dimensional geometric and appearance attributes (position, scale, rotation, opacity, spherical harmonics, and covariance) on the GPU for batched rendering and optimization. It sets itself apart from sibling methods like [Gaussian Data Packing](3257-gaussian-data-packing.md) by defining the logical struct fields and their semantic types rather than the low-level bit-packing or serialization format, and builds on [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md) which specifies the contiguous memory arrangement of these fields.

### [Gaussian Data Packing](3257-gaussian-data-packing.md)
Gaussian Data Packing serializes 3D Gaussian Splat attributes into a fixed-width binary buffer to facilitate efficient GPU-side access during rasterization. It differs from sibling methods like [SPZ Packed Format](3926-spz-packed-format.md) by using an interleaved, texture-friendly layout with a fixed width of 8 `uint32_t` per Gaussian, rather than parallel buffers or image-based compression.

### GeometryState Allocation
GeometryState contains contiguous memory blocks for $P$ Gaussians, used during the rasterization phase. It stores derived rendering attributes rather than the base Gaussian parameters.

**Layout (per Gaussian):**
*   `depths`: `float` (4 bytes). Depth of the Gaussian center.
*   `clamped`: `float` (12 bytes). Clamped 2D covariance components or similar intermediate values.
*   `internal_radii`: `float` (4 bytes). Radius of the Gaussian in screen space.
*   `means2D`: `float` (8 bytes). 2D mean position in screen space.
*   `cov3D`: `float` (24 bytes). 6 unique components of the 3D covariance matrix (symmetric).
*   `conic_opacity`: `float` (4 bytes). Combined conic matrix and opacity term.
*   `rgb`: `float` (12 bytes). Precomputed RGB color from SHs.
*   `tiles_touched`: `uint32_t` (4 bytes). Bitmask or count of tiles touched.
*   `scanning_space`: Variable size. Temporary buffer for tile scanning (from `cub::DeviceScan::InclusiveSum`).
*   `point_offsets`: `uint32_t` (4 bytes). Offset into the output buffer for this Gaussian.

Total fixed size per Gaussian: $4+12+4+8+24+4+12+4+4 = 76$ bytes + variable scanning space [S4, unofficial].

## Operating characteristics
*   **Memory Complexity**: $O(N \cdot D^2)$ where $N$ is the number of Gaussians and $D$ is the SH degree. For degree 3, each Gaussian consumes ~272 bytes in raw struct layout or ~100+ bytes in PyTorch storage (excluding SH if not stored).
*   **Time Complexity**: Access is $O(1)$ per Gaussian due to contiguous layout. Conversion from log-space to linear-space is $O(N)$.
*   **Strengths**: Contiguous memory layout enables efficient GPU vectorization. Separation of storage (log-space) and access (linear-space) allows stable optimization.
*   **Limitations**: High memory usage for high-degree SH. Precomputed covariance in raw struct doubles memory for geometric data.
*   **When to Use**: Standard 3DGS rendering and optimization.
*   **When Not to Use**: Memory-constrained embedded systems (use [Gaussian Data Packing](3257-gaussian-data-packing.md) or lower SH degree).

## Background
The 3D Gaussian Data Layout originates from the 3D Gaussian Splatting (3DGS) paper by Kerbl et al. (2023). It was designed to enable differentiable rasterization of 3D Gaussians, allowing end-to-end optimization of scene representation from images. The layout evolved to support both efficient rendering (C++/CUDA) and flexible optimization (PyTorch). The PyTorch variant with log-space parameters was introduced to stabilize gradient descent for scale and opacity.

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
[S3] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S4] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
