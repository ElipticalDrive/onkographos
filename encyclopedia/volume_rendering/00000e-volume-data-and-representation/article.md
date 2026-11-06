# Volume data and representation

Also called: Volumetric data structures, Volume representations

## Overview
Volume data and representation defines how scalar or vector fields are stored, organized, and accessed for subsequent rendering or analysis. It solves the problem of mapping continuous physical phenomena or discrete measurements into a digital format that balances memory efficiency, access speed, and geometric fidelity. This family is distinguished from [Producing volume data](11-producing-volume-data.md) (which focuses on acquisition) and [Image-order rendering](16-image-order-rendering.md) (which focuses on pixel synthesis) by its exclusive concern with the intermediate data layer: the bridge between raw sensor data and the visual output.

## Shared principles
All volume representations must support two fundamental operations: **storage** (mapping a field $f(\mathbf{x})$ to memory) and **query** (evaluating $f(\mathbf{x})$ at arbitrary coordinates $\mathbf{x}$). The choice of representation dictates the trade-off between spatial locality, memory overhead, and interpolation accuracy.

1.  **Domain Discretization:** The continuous domain $\Omega \subset \mathbb{R}^3$ is discretized into a set of elements or samples. For regular grids, this is a lattice $\mathbf{x}_{ijk} = \mathbf{x}_0 + i\Delta x \hat{i} + j\Delta y \hat{j} + k\Delta z \hat{k}$. For irregular volumes, it is a set of nodes and connectivity graphs.
2.  **Value Encoding:** Scalar values are stored as discrete integers or floating-point numbers. The precision (e.g., 8-bit unsigned, 16-bit float, 32-bit float) determines the dynamic range and quantization error [S1].
3.  **Interpolation:** Since rays or query points rarely align with sample centers, values must be interpolated. Common kernels include:
    *   **Nearest Neighbor:** $V(P) = V(\text{round}(\mathbf{x}))$. Fast, discontinuous.
    *   **Trilinear:** $V(P) = \sum_{i,j,k \in \{0,1\}} w_i w_j w_k V_{ijk}$, where $w$ are distances to grid nodes. Continuous, first-order derivatives.
    *   **Higher-Order:** Bilinear/Bicubic splines or cubic convolution for smoother gradients [S1, S3].
4.  **Coordinate Mapping:** A transformation maps world coordinates to texture/grid coordinates. For GPU rendering, this often involves normalizing coordinates to $[0, 1]$ for texture sampling [G].

## Subtopics

### [Dense grids and memory layout](3-dense-grids-and-memory-layout.md)
Dense grids store data in regular, contiguous memory blocks (e.g., 3D textures, linear arrays). They offer $O(1)$ access time and high spatial locality, making them ideal for GPU texture units. Key mechanisms include bricking (subdividing large volumes into smaller bricks for cache efficiency) and Morton/Z-order curve traversal to improve spatial coherence during ray marching. Choose this when the volume is dense, fits in memory, and requires fast, uniform access.

### [Compression](4-compression.md)
Compression reduces the memory footprint and transmission bandwidth of volumetric datasets by exploiting spatial redundancy, value correlation, or structural sparsity, enabling the rendering of large-scale or time-varying volumes that would otherwise exceed hardware limits. This family encompasses methods ranging from lossless entropy coding and sparse data structures to lossy quantization, dictionary learning, and neural representations. What sets this family apart from sibling methods like [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) (which focuses on access patterns) or [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md) (which focuses on geometric approximation) is the explicit transformation of data values or topology to reduce bit-count, often at the cost of reconstruction fidelity or increased decoding complexity.

### [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md)
Multiresolution and level of detail (LOD) techniques provide a hierarchical framework for interpreting volumetric data, allowing the system to use the most economical resolution for each feature based on viewing distance, interaction state, or available memory bandwidth. This family solves the problem of rendering large-scale volumetric datasets interactively by replacing high-resolution data with lower-resolution approximations for distant or less important regions, thereby reducing GPU memory usage and rendering latency. It is distinguished from sibling methods like [Compression](4-compression.md) (which focuses on storage efficiency) and [Dense grids](3-dense-grids-and-memory-layout.md) (which focus on memory layout) by explicitly organizing data into multiple resolution levels that can be selected or blended during rendering.

### [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md)
This family addresses the rendering of volumetric datasets that exceed the memory capacity of a single processing unit or the bandwidth limits of the system bus. It solves the scalability bottleneck inherent in single-node rendering by partitioning data or computation across multiple nodes, or by streaming data from disk to GPU in chunks. What sets this family apart from [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) is the explicit handling of data movement and partitioning strategies, whereas dense grids focus on in-memory organization. It differs from [Compression](4-compression.md) by focusing on temporal/spatial streaming rather than static storage reduction, and from [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md) by prioritizing data delivery over visual approximation, though they often combine.

### [Non-regular volumes](9-non-regular-volumes.md)
Non-regular volumes represent scalar fields over unstructured, hierarchical, or particle-based domains rather than fixed Cartesian grids, solving the problem of efficiently storing and rendering sparse, anisotropic, or geometrically complex data without the memory overhead of uniform high-resolution grids. This family distinguishes itself from [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) by abandoning fixed topology in favor of adaptive structures (such as tetrahedral meshes, octrees, or point clouds) that concentrate resolution where data features exist, and from [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by relying on explicit geometric primitives and basis functions rather than implicit neural networks for field evaluation.

### [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md)
Neural and primitive-based representations encode volumetric or radiance fields using parameterized functions (typically neural networks) or explicit geometric primitives (such as Gaussians, ellipsoids, or splats) rather than dense voxel grids. This family solves the memory and scalability limitations of explicit storage by providing compact, continuous, or sparse representations that can be optimized from sparse input views. Unlike dense grids, these methods avoid the need for large, uniform memory allocation, and unlike pure implicit fields, primitive-based variants often offer faster rendering speeds and easier integration with standard graphics pipelines.

## Background
The concept of volume data representation originated with the development of medical imaging modalities like CT and MRI in the 1970s, which produced discrete 3D arrays of attenuation coefficients [S1]. Early visualization relied on dense 3D textures and ray casting. As data sizes grew, research shifted toward hierarchical structures (octrees) and compression to manage memory. Recent years have seen a surge in neural representations (NeRF, 3D Gaussian Splatting) driven by the need for compact, differentiable scene models for novel view synthesis [S1, S3].

## Sources
[S1] Volume Rendering: Principles and Practice, URL, date fetched
[S2] Multiresolution Volume Visualization, URL, date fetched
[S3] Neural Radiance Fields, URL, date fetched
[S4] Distributed Volume Rendering, URL, date fetched
[S5] Volume Data Compression, URL, date fetched
