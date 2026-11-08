# Quantization


## Overview
Quantization is a lossy compression technique that maps continuous or high-precision source values to a discrete set of indices, reducing the memory footprint of volumetric data [S1][S2]. It solves the problem of storing large datasets (e.g., 32-bit simulation data or dense neural fields) by collapsing them into smaller bit-depths, typically achieving significant compression ratios [S1][S2]. Unlike general [Value compression](6-value-compression.md) methods that may use transform domains or block codecs, quantization directly reduces the precision of the data values themselves, either globally, locally, or perceptually, setting it apart as the fundamental step for bit-depth reduction in many compression pipelines.

## Shared principles
The family shares the core mechanism of mapping a source value $x$ from a continuous or high-bit-depth domain to a discrete index $i$ in a lower-bit-depth domain, followed by a reconstruction function $R(i)$ to approximate the original value. The general error introduced is the quantization error $e = x - R(Q(x))$. The choice of quantization function $Q(x)$ and reconstruction function $R(i)$ depends on the data type (scalar, color, normal) and the desired trade-off between compression ratio and perceptual or numerical fidelity. Common principles include:
1.  **Range Definition:** Determining the source range $[x_{min}, x_{max}]$, which can be global, local (per-block), or adaptive.
2.  **Step Size/Levels:** Defining the number of quantization levels $M$ (e.g., $2^b$ for $b$ bits) and the step size $\phi = (x_{max} - x_{min}) / M$.
3.  **Mapping:** Applying a mapping function, often linear (uniform) or non-linear (perceptual, adaptive), to convert $x$ to $i$.
4.  **Reconstruction:** Mapping $i$ back to $x'$, often by centering the value in the quantization bin ($x' = x_{min} + i \cdot \phi + \phi/2$) or using linear interpolation between stored extremes.
5.  **Error Management:** Handling edge cases such as constant blocks, out-of-range values, and floating-point rounding errors through clamping or special handling [G].

## Subtopics

### [Attribute Quantization](1912-attribute-quantization.md)
Specializes in reducing the precision of voxel attributes like colors and normals. It differs from scalar methods by using perceptually uniform color spaces (CIELAB) and geometric mappings (Octahedron Normal Vectors) to maintain visual quality, rather than generic linear mappings. Choose this when attribute fidelity (color/normal) is critical for rendering quality.

### [Scalar Quantization](5726-scalar-quantization.md)
A per-block random-access method that maps voxel values to local indices based on block-specific min/max values. It differs from global uniform quantization by adapting to local data ranges, enabling parallel GPU reconstruction. Choose this for random-access decompression where local variance is high.

### [Uniform Quantization](5855-uniform-quantization.md)
The simplest lossy method, mapping values to a discrete set using a constant step size across the global data range. It differs from adaptive methods by distributing levels evenly, which is simple but less effective for data with wide dynamic ranges. Choose this for data with uniform distribution or when simplicity is paramount.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Reduces bitrate by lowering the spatial resolution of chrominance (U, V) components while keeping luminance (Y) at full resolution. It differs from direct subsampling by using median aggregation over 8-corner groups, preserving color quality better for sparse volumes. Choose this when color fidelity is less critical than luminance, or for standard video-like volumetric data.

### [TetSplat Pre-processing and Data Layout](7221-tetsplat-pre-processing-and-data-layout.md)
Compresses scalar fields and geometric metadata into a binary bitstring for hierarchical tree structures (Shell/Solid). It differs from general quantization by embedding traversal pointers and targeting unbalanced quadtrees/octrees for fast top-down rendering. Choose this for interactive rendering of tetrahedral meshes.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Maps spatial coordinates to field values using a quantized neural network (SIREN). It differs from traditional compressors by serving as both storage and a queryable data source, allowing direct evaluation. Choose this for compact representation of complex implicit fields where random access is needed.

### Lossy Quantization for Coarsened Patches
Lossy compression for coarsened patches in a bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$). Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else 0. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S1].

### Differentiable Quantization for Neural Fields
Quantizes grid values to single bytes during optimization to reduce render-time memory, using a sigmoid to map to $[0,1]$, quantization to byte, and affine mapping to $[-m, m]$. It uses a straight-through estimator for gradients to allow backpropagation through the non-differentiable rounding step [S2]. The quantization function is $q(x) = x + \text{stop-gradient}\left( \frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x \right)$ [S2]. Parameters are $m=14$ for densities and $m=7$ for diffuse colors and features [S2]. A key pitfall is that simply quantizing after training causes quality drops; quantizing during optimization with the straight-through estimator is required [S2].

## Background
Quantization is a fundamental concept in signal processing and data compression, widely used in volume rendering to reduce memory bandwidth and storage requirements. Its origins in this context are tied to the need to handle large scientific datasets (e.g., fluid dynamics, medical imaging) that exceed available memory. Early methods focused on uniform quantization for its simplicity [S1][S2]. Later developments introduced adaptive and perceptual quantization to handle the specific characteristics of volumetric data, such as wide dynamic ranges and the importance of certain features for visualization [S1][S2]. The rise of neural implicit representations has further driven the development of differentiable quantization techniques to enable efficient training and rendering of neural fields [S2].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
