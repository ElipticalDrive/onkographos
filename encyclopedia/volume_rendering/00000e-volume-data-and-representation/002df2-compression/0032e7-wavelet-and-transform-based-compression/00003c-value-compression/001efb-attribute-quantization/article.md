# Attribute Quantization

Also called: CIELAB color quantization, Perceptual quantization

## Overview
Attribute Quantization is a lossy compression technique that reduces the precision of voxel attributes (colors and normals) to lower memory footprint while maintaining perceptual quality [S1]. It solves the problem of excessive memory usage in attribute arrays, which often exceeds the geometry storage size in hierarchical volume representations [S1]. It is distinguished from its siblings, such as [Quantization](5935-quantization.md) (generic) and [Thresholding and Quantization](5722-thresholding-and-quantization.md) (coefficient reduction), by specializing the quantization strategy based on the data type (e.g., using CIELAB for colors and Octahedron Normal Vectors for normals) to leverage human perception limits and achieve higher precision per bit [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume attribute array $A$ containing continuous values for colors (RGB) and/or normals (3D vectors) [S1].
*   **Output:** A compressed attribute array with reduced bit depth and/or palette indices, ready for entropy coding [S1].

**Data Structures:**
*   **Color:** Quantized values in CIELAB space [S1].
*   **Normals:** Quantized Octahedron Normal Vectors (ONVs) [S1].
*   **Palette:** A lookup table mapping quantized indices to representative attribute values, used in conjunction with palette compression [S1].

**Steps:**

1.  **Color Quantization (CIELAB):**
    *   Convert input RGB colors to CIELAB color space, which is (locally) perceptually uniform, allowing clustering based on perceived difference rather than Euclidean distance in RGB [S1].
    *   Perform data-driven clustering (e.g., k-means) on the CIELAB values to determine representative colors; the number of colors is user-defined, with 12-bit (4096 colors) being typical [S1].
    *   Replace each voxel's color with the index of its nearest cluster center [S1].
    *   *Pitfall:* This method requires preprocessing to analyze the color distribution of the scene [S1].

2.  **Normal Quantization (ONVs):**
    *   For each normal vector $\mathbf{n} = (n_x, n_y, n_z)$, compute the Octahedron Normal Vector representation [S1].
    *   The ONV maps the sphere to an octahedron, allowing for uniform quantization and yielding higher precision for the same number of bits compared to storing one value per dimension [S1].
    *   Quantize the ONV coordinates to the desired bit depth (e.g., 10-bit or 12-bit) [S1].
    *   *Typical Values:* 12-bit ONVs produce minor banding; 10-bit may suffice for diffuse shading but causes visible banding on smooth surfaces; 16-bit is preferred for specular effects [S1].

3.  **Palette Compression:**
    *   Construct a palette of unique quantized attributes [S1].
    *   Replace the attribute array with indices into this palette [S1].
    *   This reduces the number of distinct values significantly, improving subsequent entropy coding ratios [S1].

**Formulas:**
*   **CIE94 Delta-E:** Parameters: $k_L = 1$, $K_1 = 0.045$, $K_2 = 0.015$, with D65 illuminant as reference white [S1].
*   **Color Difference Magnitude:** $\sqrt{dR^2 + dG^2 + dB^2}$, used for visual comparison metrics [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantize_attributes(attributes, color_bits=12, normal_bits=12):
    # attributes: list of (color_rgb, normal_vec) tuples
    colors = [a[0] for a in attributes]
    normals = [a[1] for a in attributes]

    # 1. Color Quantization (CIELAB clustering)
    lab_colors = standard_rgb_to_lab(colors) # [G] Standard color science transform
    centroids = kmeans(lab_colors, k=2**color_bits) # [G] Standard iterative clustering
    color_indices = []
    for lab in lab_colors:
        best_idx = min(range(len(centroids)), key=lambda i: sum((l - c)**2 for l, c in zip(lab, centroids[i])))
        color_indices.append(best_idx)

    # 2. Normal Quantization (ONV)
    onv_indices = []
    for n in normals:
        nx, ny, nz = normalize(n)
        sign_z = 1.0 if nz >= 0 else -1.0
        onv_x = sign_z * (1.0 - abs(ny))
        onv_y = sign_z * (1.0 - abs(nx))
        max_val = 2**normal_bits - 1
        q_x = round(onv_x * max_val)
        q_y = round(onv_y * max_val)
        onv_indices.append((q_x, q_y))

    # 3. Palette Construction
    unique_color_indices = sorted(set(color_indices))
    palette_colors = [centroids[i] for i in unique_color_indices]
    unique_onv_indices = sorted(set(onv_indices))
    palette_normals = unique_onv_indices

    return color_indices, onv_indices, palette_colors, palette_normals
```

## Subtopics

### [Quantization with Noise Injection](2090-quantization-with-noise-injection.md)
Quantization with Noise Injection compresses continuous attributes into discrete values for entropy coding by simulating quantization error during training via uniform noise injection, while applying explicit rounding during inference. It differs from standard quantization by preserving gradient flow through the otherwise non-differentiable rounding operation, enabling end-to-end training with quantization constraints.

### [Thresholding and Quantization](5722-thresholding-and-quantization.md)
Thresholding and Quantization is a lossy compression technique that reduces the data size of a compact volume model by neglecting or approximating its coefficients. It solves the problem of excessive memory footprint in hierarchical or transform-based representations by allowing the user to define an acceptable error bound. It is distinguished from its parent, Attribute Quantization, by focusing on the reduction of model coefficients (such as wavelet or tensor coefficients) rather than just attribute precision, and from its sibling, Quantization with Noise Injection, by relying on deterministic truncation or rounding rather than stochastic noise addition.

### [Colorspace Transformations for Compression](16470-colorspace-transformations-for-compression.md)
This method transforms RGB volume data into an optimized colorspace to decorrelate channels and assign significance weights for encoding and quantization. It sets itself apart from sibling methods by explicitly leveraging perceptual or statistical channel importance to prioritize wavelet coefficients during compression, rather than treating all color channels equally.

## Operating characteristics
*   **Memory Reduction:** Attribute arrays often exceed the geometry (DAG) size. Quantization combined with palette compression can reduce the attribute memory footprint to a similar order of magnitude as the geometry [S1].
*   **Compression Rates:** Reported compression rates for attributes vary by scene (e.g., 20% to 69% reduction) [S1].
*   **Perceptual Quality:** Assessed using CIE94 Delta-E and SSIM metrics. 12-bit colors and normals generally maintain high visual quality [S1].
*   **Preprocessing Cost:** Color quantization requires preprocessing to analyze colors, which adds an initial computational cost [S1].
*   **Bit Depth Trade-offs:** Lower bit depths (e.g., 10-bit normals) may cause visible banding on smooth surfaces or specular highlights, while higher bit depths (16-bit) are preferred for high-fidelity effects [S1].

## Background
Attribute Quantization was proposed in the context of geometry and attribute compression for voxel scenes [S1]. It addresses the limitation of lossless palette compression, which is overly conservative for scenes with millions of similar colors [S1]. By leveraging perceptual uniformity (CIELAB) and efficient normal representation (ONVs), it enables significant memory savings without noticeable visual degradation [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
