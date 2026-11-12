# Data Layout and Storage

Also called: Gaussian Data Layout, Splat Buffer Organization

## Overview
Data Layout and Storage defines the memory organization of 3D Gaussian primitives for efficient rendering and optimization. It solves the problem of cache inefficiency and memory bandwidth bottlenecks by organizing Gaussian attributes (position, scale, rotation, opacity, and spherical harmonics) into contiguous, attribute-aligned buffers. It differs from sibling methods by focusing on the fundamental Structure-of-Arrays (SoA) memory layout rather than coefficient indexing [5570-spherical-harmonics-coefficient-layout.md], covariance decoding [6840-covariance-splat-reading.md], or dynamic scene management [6907-dynamic-gaussian-splatting-scene-management.md].

## Shared principles
All child methods in this family adhere to a Structure-of-Arrays (SoA) layout, where each attribute type occupies a separate contiguous linear buffer. This organization ensures that rendering shaders iterate over single attributes without skipping irrelevant data, maximizing cache locality during batched processing [5568-splat-data-buffer-layout.md].

The core attributes stored in these buffers are:
1.  **Positions:** A buffer of size $N \times 3$ storing $(x, y, z)$ coordinates.
2.  **Scales:** A buffer of size $N \times 3$ storing $(s_x, s_y, s_z)$. As noted in [5568-splat-data-buffer-layout.md], sources may store logarithmic scales, which are exponentiated during rendering [G].
3.  **Rotations:** A buffer of size $N \times 4$ storing quaternion components. The standard order is scalar-last $(x, y, z, w)$ [5568-splat-data-buffer-layout.md].
4.  **Opacities:** A buffer of size $N \times 1$ storing a single scalar opacity value per splat.
5.  **Spherical Harmonics (SH):** A buffer of size $N \times D \times 3$, where $D$ is the number of SH coefficients per channel and 3 represents the RGB channels.

Indexing into the SH buffer for the $c$-th component (channel, $c \in \{0,1,2\}$) of the $d$-th coefficient of the $i$-th splat is calculated as:
$$ idx = ((i \times D) + d) \times 3 + c $$
For non-SH attributes, the stride is fixed by dimensionality (3 for position/scale, 4 for rotation, 1 for opacity) [5568-splat-data-buffer-layout.md].

## Subtopics

### [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md)
This method defines the memory layout for 3D Gaussian splats in a Structure-of-Arrays (SoA) format, where each attribute occupies a separate contiguous linear buffer. It solves the problem of cache inefficiency during batched rendering by ensuring shaders iterate over single attributes without skipping irrelevant data. It differs from sibling methods by focusing on the fundamental in-memory SoA organization rather than coefficient indexing [5570-spherical-harmonics-coefficient-layout.md], covariance calculation [6840-covariance-splat-reading.md], or dynamic scene management [6907-dynamic-gaussian-splatting-scene-management.md].

### [Spherical Harmonics Coefficient Layout](5570-spherical-harmonics-coefficient-layout.md)
This method defines the memory layout for storing Spherical Harmonics (SH) coefficients for 3D Gaussian primitives in a Structure-of-Arrays (SoA) format. It solves the problem of efficiently accessing per-splat color information for rendering and optimization by grouping coefficients by splat, then by coefficient index, and finally by RGB channel. It differs from texture-based or codebook-based layouts by keeping all coefficients in a single contiguous linear buffer, enabling direct indexing without texture lookups or codebook decoding.

### [Covariance Splat Reading](6840-covariance-splat-reading.md)
Covariance Splat Reading is a method for decoding a single 3D Gaussian Splat from compressed texture storage into a structured representation suitable for rendering. It solves the problem of efficiently retrieving Gaussian properties (position, color, opacity, and covariance) from a GPU-friendly texture array format. It differs from raw buffer access by utilizing two `usampler2DArray` textures to store packed integer data, requiring specific unpacking logic to reconstruct the continuous covariance matrix components from their discrete storage format.

### [Dynamic Gaussian Splatting Scene Management](6907-dynamic-gaussian-splatting-scene-management.md)
Dynamic Gaussian Splatting Scene Management provides a thread-safe, GPU-resident data structure and API for incrementally adding, resizing, and rendering 3D Gaussian splats in real-time environments, specifically designed to bridge asynchronous ROS2 topic data with a Vulkan-accelerated renderer. It differs from static Gaussian Splatting implementations by introducing a pending batch queue system, automatic buffer growth strategies, and a direct bridge between ROS2 message types and the renderer's internal device memory layout, ensuring synchronization without blocking the rendering loop.

### [WASM Data Packing](7761-wasm-data-packing.md)
WASM Data Packing manages WebAssembly memory allocation and data packing for 3D Gaussian splat attributes (positions, rotations, scales, colors, selection) to prepare them for GPU upload or WASM-side processing. It differs from standard CPU-side memory management by handling the specific binary layout requirements of the WebAssembly heap, ensuring that contiguous JavaScript typed arrays are correctly mapped to the linear memory regions expected by the compiled C/C++ rendering kernels.

### Log-Scale Storage
Scales are stored internally in log-space. When writing to buffers, linear values are converted to log-space via $\log(s)$. When reading for rendering or optimization, log-space values are converted to linear via $\exp(s)$ [S1, unofficial]. Linear scale is recovered from stored log-scale as $s_{linear} = \exp(s_{log})$ [S1, unofficial].

### Rotation Quaternion Ordering
Rotations are stored as 4-component quaternions in scalar-last order $(x, y, z, w)$ within a FloatBuffer of size $N \times 4$ [S1, unofficial]. For each splat $i$, the quaternion components are written sequentially to indices $i \times 4 + 0$ through $i \times 4 + 3$ [S1, unofficial].

## Background
The Structure-of-Arrays (SoA) layout for 3D Gaussian Splatting was formalized in the JSplat library, which provides a Java-based implementation for reading, writing, and managing Gaussian splat data [S1, unofficial]. This approach aligns with the broader trend in differentiable rendering to optimize memory access patterns for GPU shaders. The specific handling of log-scales for Gaussian dimensions and scalar-last quaternion ordering are key implementation details documented in the JSplat source code [S1, unofficial].

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
