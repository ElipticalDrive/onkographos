# 3D Gaussian Data Layout

Also called: 3DGS Parameter Storage, Gaussian Struct Definition, GaussianModel, GaussianModel PyTorch Module

## Overview
The 3D Gaussian Data Layout defines the memory layout and fields for a single 3D Gaussian primitive, storing position, scale, orientation, opacity, and color (spherical harmonics) data [S1, unofficial][S2, unofficial]. It also refers to a PyTorch `nn.Module` storing per-Gaussian parameters for 3D Gaussian Splatting [S2, unofficial]. It differs from the [GaussianSet Data Structure](2074-gaussianset-data-structure.md) by defining the primitive's internal fields rather than the container logic, and from [Gaussian Data Packing](3257-gaussian-data-packing.md) by using raw floating-point tensors for optimization rather than compacted integer buffers for rendering [S1, unofficial][S2, unofficial].

## How it works
The layout stores per-primitive attributes in contiguous arrays to facilitate batched processing and differentiable updates [S1, unofficial]. Two primary implementations exist: a Rust-based struct for rendering pipelines and a PyTorch module for optimization [S1, unofficial][S2, unofficial]. It stores precomputed 3D covariance to avoid recomputation during rasterization [S1, unofficial].

### Inputs and Outputs
*   **Input**: None for the static layout definition. For initialization, inputs include `xyz` (N, 3) numpy array of 3D points and `rgb` (N, 3) numpy array of colors with values between 0 and 1 [S2, unofficial].
*   **Output**: A data structure instance containing raw parameters ready for covariance computation or rendering [S1, unofficial][S2, unofficial].

### Data Structures

#### Rust Implementation (Rendering)
Defined in [S1, unofficial], the `Gaussian` struct uses `nalgebra` types. It stores up to degree 3 spherical harmonics [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
#[derive(Clone, Debug)]
pub struct Gaussian {
    pub position: na::Vector3<f32>,  // x, y, z
    pub scale: na::Vector3<f32>,    // scale_0, scale_1, scale_2
    pub opacity: f32,               // Linear opacity [0, 1]
    pub rotation: na::Quaternion<f32>, // w, x, y, z
    pub sh: na::SVector<f32, 48>,   // Spherical Harmonics coefficients
    pub cov3d: na::Matrix3<f32>,    // 3x3 symmetric covariance matrix
}
```

*   **position**: `na::Vector3<f32>` (x, y, z).
*   **scale**: `na::Vector3<f32>` (scale_0, scale_1, scale_2). Represents linear scale factors.
*   **opacity**: `f32`. Linear opacity value.
*   **rotation**: `na::Quaternion<f32>` (w, x, y, z). Unit quaternion.
*   **sh**: `na::SVector<f32, 48>`. Stores spherical harmonics coefficients. The `sh_dim` specifies the number of coefficients: 3 for DC, 12 for degree 1, 27 for degree 2, 48 for degree 3 [S1, unofficial].
*   **cov3d**: `na::Matrix3<f32>`. Precomputed 3D covariance matrix.

#### PyTorch Implementation (Optimization)
Defined in [S2, unofficial], the `GaussianModel` class stores parameters in log-space or raw tensors to ensure valid ranges during gradient descent.

*   `_xyz`: `torch.Tensor` of shape `(N, 3)`. Float tensor for positions.
*   `_features_dc`: `torch.Tensor` of shape `(N, 1, 3)`. Float tensor for 0th degree spherical harmonics (base colors) [S2, unofficial].
*   `_scaling`: `torch.Tensor` of shape `(N, 3)`. Float tensor for log-scales.
*   `_rotation`: `torch.Tensor` of shape `(N, 4)`. Float tensor for quaternions.
*   `_opacity`: `torch.Tensor` of shape `(N, 1)`. Float tensor for log-odds opacity.

### Parameter Retrieval Steps
The PyTorch implementation provides explicit retrieval functions for constrained parameters [S2, unofficial]:

1.  **Get Scaling**: Compute linear scale from log-space [S2, unofficial].
    $$ \text{scale} = \text{clamp}(\exp(\text{self._scaling}), \text{max}=10.0) $$
    This converts log-space scaling to linear space with an upper bound of 10.0 [S2, unofficial].
2.  **Get Rotation**: Normalize the quaternion [S2, unofficial].
    $$ \text{rotation} = \text{normalize}(\text{self._rotation}) $$
    This ensures quaternions are unit-length [S2, unofficial].
3.  **Get Opacity**: Convert log-odds to probability [S2, unofficial].
    $$ \text{opacity} = \sigma(\text{self._opacity}) = \frac{1}{1 + e^{-\text{self._opacity}}} $$
    This converts log-odds to probability space $[0, 1]$ [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_scaling(_scaling: torch.Tensor) -> torch.Tensor:
    return torch.clamp(torch.exp(_scaling), max=10.0)

def get_rotation(_rotation: torch.Tensor) -> torch.Tensor:
    return torch.nn.functional.normalize(_rotation)

def get_opacity(_opacity: torch.Tensor) -> torch.Tensor:
    return torch.sigmoid(_opacity)
```

### Initialization from Point Cloud
To create a `GaussianModel` from a COLMAP point cloud [S2, unofficial]:
1.  Input `xyz`: `(N, 3)` numpy array of 3D points.
2.  Input `rgb`: `(N, 3)` numpy array of colors, values between 0 and 1.
3.  Initialize `_xyz` with `xyz`.
4.  Initialize `_features_dc` with `rgb`.
5.  Initialize `_scaling`, `_rotation`, and `_opacity` with default zeros or small random values (implementation body for initialization is not fully specified in the sources) [S2, unofficial][G].

### Edge Cases and Pitfalls
*   **Quaternion Normalization**: Quaternions must be normalized after every update step to remain valid rotations [S2, unofficial].
*   **Scale Clamping**: Exponential scaling can explode; an upper bound (e.g., 10.0) is applied to prevent numerical instability [S2, unofficial].
*   **SH Dimension**: The size of the `sh` vector must match the active spherical harmonics degree [S1, unofficial].

## Subtopics

### [Gaussian Data Structure](2273-gaussian-data-structure.md)
The Gaussian Data Structure is the primary in-memory container for 3D Gaussian primitives in differentiable volume rendering pipelines, organizing per-primitive attributes into contiguous arrays for efficient GPU access and optimization. It differs from the raw 3D Gaussian Data Layout by providing a flat, list-based abstraction that facilitates dynamic resizing and batched processing, distinct from the tightly packed bit-level formats of [Gaussian Data Packing](3257-gaussian-data-packing.md).

### [Gaussian Data Packing](3257-gaussian-data-packing.md)
Gaussian Data Packing is a method for compacting 3D Gaussian Splatting primitives into a dense, GPU-friendly binary buffer for efficient rendering and sorting. It solves the problem of memory alignment and bandwidth efficiency by interleaving position, covariance, and color data into a fixed-size `uint32_t` array, while storing raw floating-point positions in a separate parallel buffer. It is distinguished from its sibling [Gaussian Data Structure](2273-gaussian-data-structure.md) by its specific bit-packing strategy for covariance matrices and its separation of position data into integer-reinterpreted and raw float formats.

### GeometryState Allocation
GeometryState contains contiguous memory blocks for P Gaussians, used during the rasterization phase [S3, unofficial]. The layout includes:
*   `depths`: `float`, size P.
*   `clamped`: `float`, size P*3.
*   `internal_radii`: `float`, size P.
*   `means2D`: `float`, size P.
*   `cov3D`: `float`, size P*6 (symmetric 3x3 matrix stored as 6 elements).
*   `conic_opacity`: `float`, size P.
*   `rgb`: `float`, size P*3.
*   `tiles_touched`: `uint32_t`, size P.
*   `scanning_space`: Variable size, allocated via `cub::DeviceScan::InclusiveSum`.
*   `point_offsets`: `uint32_t`, size P [S3, unofficial].

## Operating characteristics
*   **Memory**: The Rust struct uses `position` (12 bytes), `scale` (12 bytes), `opacity` (4 bytes), `rotation` (16 bytes), `sh` (192 bytes), and `cov3d` (36 bytes), totaling 272 bytes per Gaussian, though `nalgebra` alignment may add padding [S1, unofficial]. The PyTorch layout stores parameters in separate tensors of shapes `(N, 3)`, `(N, 1, 3)`, `(N, 3)`, `(N, 4)`, and `(N, 1)` [S2, unofficial].
*   **Scalability**: The layout supports N Gaussians. The `GeometryState` scales with P Gaussians, requiring dynamic allocation for `scanning_space` [S3, unofficial].
*   **Strengths**: Direct mapping to mathematical parameters. Precomputed covariance (Rust) saves computation during rendering [S1, unofficial]. Log-space parameters (PyTorch) stabilize optimization [S2, unofficial].
*   **Limitations**: High memory usage for high-degree spherical harmonics (48 floats per Gaussian). Not optimized for bandwidth during rendering compared to packed formats [S3, unofficial].

## Background
The 3D Gaussian Data Layout originates from the 3D Gaussian Splatting (3DGS) framework for differentiable volume rendering [G]. The Rust struct originates from an unofficial repository [S1, unofficial], the PyTorch module from another unofficial repository [S2, unofficial], and the GeometryState layout from a third unofficial repository [S3, unofficial]. These implementations adapt the core 3DGS primitives for specific hardware and software constraints [G].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S3] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
