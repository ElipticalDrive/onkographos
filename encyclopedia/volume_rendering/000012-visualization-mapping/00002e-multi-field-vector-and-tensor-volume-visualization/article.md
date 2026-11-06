# Multi-field, vector and tensor volume visualization

Also called: Multi-field visualization, Vector and tensor volume visualization

## Overview
This family encompasses techniques for visualizing multi-component data fields—specifically vector, tensor, and complex-valued fields—within a volume rendering context. It solves the problem of representing directional, anisotropic, or phase-dependent information that scalar volume rendering cannot convey. It sets itself apart from [Classification and transfer functions](38-classification-and-transfer-functions.md) by mapping non-scalar data attributes (direction, magnitude, phase) to visual properties, and from [Illustrative and non-photorealistic volume rendering](40-illustrative-and-non-photorealistic-volume-rendering.md) by focusing on geometric or color-based encoding of field structure rather than artistic style.

## Shared principles
The core mechanism involves sampling multi-component data at each voxel or particle position and mapping the resulting vector/tensor/complex value to visual attributes such as color, geometry, or opacity.

1.  **Data Sampling**: The field is sampled at specific locations. For vector fields, this yields a 3D direction and a scalar magnitude. For diffusion tensors, this yields a $3 \times 3$ symmetric positive-definite matrix. For complex fields, this yields real and imaginary components.
2.  **Normalization and Decomposition**:
    *   **Vectors**: The direction is normalized to a unit vector $\mathbf{v} = \frac{\mathbf{d}}{\|\mathbf{d}\|}$. The magnitude $m = \|\mathbf{d}\|$ is often used for color or length scaling.
    *   **Tensors**: Eigenvalue decomposition is typically performed to extract principal directions and diffusivity measures (e.g., Fractional Anisotropy).
    *   **Complex Numbers**: The argument (phase) $\theta = \arg(z)$ and magnitude $|z|$ are computed.
3.  **Visual Mapping**:
    *   **Glyph-based**: Geometric primitives (lines, arrows) are generated at sample points. The primitive's orientation aligns with the field direction, and its size/color encodes magnitude or other derived metrics.
    *   **Color-based**: The field value is mapped directly to RGB color. For vectors, this often involves mapping the direction to a cyclic colormap and magnitude to brightness. For tensors, eigenvalues may map to color channels.
4.  **Rendering**: Glyphs are rendered as geometric primitives (triangles/lines) using standard rasterization or ray-marching. Color-based fields are integrated along rays using standard volume rendering equations, where the color at each step is determined by the local field value.

## Subtopics

### [Domain Colouring](1833-domain-colouring.md)
Domain Colouring maps complex numbers to colors using a cyclic colormap scaled by a degree parameter and brightness. It solves the problem of visualizing the phase and magnitude of complex-valued fields, such as spin orientation. It sets itself apart from sibling methods like [Line Glyphs](2569-line-glyphs-for-vector-field-visualization.md) by providing a dense, pixel-wise color field rather than discrete geometric primitives.

### [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md)
Line Glyphs represent vector fields by line segments, with orientation matching the vector and length/color determined by magnitude. It is distinguished from sibling methods like [Arrow Glyphs](2663-arrow-glyphs-for-vector-field-visualization.md) by its extreme computational efficiency and memory footprint, allowing for millions of glyphs, but suffers from ambiguity regarding vector direction and 3D orientation due to the lack of shading.

### [Arrow Glyphs for Vector Field Visualization](2663-arrow-glyphs-for-vector-field-visualization.md)
Arrow glyphs resolve the direction ambiguity inherent in line-based vector field visualizations by using arrow-shaped primitives that explicitly encode flow direction. Unlike [Line Glyphs for Vector Field Visualization](2569-line-glyphs-for-vector-field-visualization.md), which suffer from orientation ambiguity, arrow glyphs use triangular faces to allow for 3D shading and clear directional cues. This method is implemented entirely on the GPU to avoid CPU-GPU data transfer, resulting in performance approximately 2x faster than line glyphs (about 10 ms for typical scenes) due to the larger visual size of arrows allowing for lower sampling density.

### [Diffusion Tensor Visualization using Random Field Correlation](3441-diffusion-tensor-visualization-using-random-field-correlatio.md)
A volume rendering method that visualizes diffusion tensor fields by correlating a random noise field anisotropically using the diffusion tensor, preserving high-frequency components in transversal directions while smearing noise in the direction of greater diffusivity.

### [Stream Lines](4796-stream-lines.md)
Visualization of vector fields by computing trajectories of mass-less particles (stream lines) seeded at specific locations, using an integrator to advance the particle position.

### [Diffusion Tensor Imaging (DTI) Visualization](5713-diffusion-tensor-imaging-dti-visualization.md)
An MRI based acquisition modality yielding 3x3 symmetric diffusion tensors as its native measurement quantity.

### [Multi Field Visualization](6557-multi-field-visualization.md)
Not given in the sources.

## Background
The visualization of vector and tensor fields is a fundamental problem in scientific visualization, arising in fluid dynamics, electromagnetism, and medical imaging (specifically Diffusion Tensor Imaging). Early methods relied on glyph-based approaches (arrows, lines) for their intuitive geometric representation. As computational power increased, GPU-based implementations allowed for dense glyph rendering. Color-based methods, such as domain coloring for complex fields, provide a dense alternative to glyphs. The integration of these techniques into volume rendering pipelines allows for the simultaneous visualization of scalar context (e.g., density) and vector/tensor structure.

## Sources
[S1] Unofficial notes on multi-field, vector and tensor volume visualization.
