# Wavelet-based compression rendering

Also called: Wavelet domain volume rendering, Wavelet rendering

## Overview
Wavelet-based compression rendering integrates volume rendering with wavelet compression to exploit the multiresolution hierarchy of the wavelet transform for both data compression and rendering acceleration [S1, book]. It solves the problem of efficiently rendering large volumetric datasets by leveraging local frequency variance to skip rendering in homogeneous areas, thereby reducing computational load [S1, book]. Unlike standard value compression which focuses primarily on storage, this method specifically targets rendering speed by allowing adaptive resolution based on wavelet coefficient significance [S1, book].

## How it works
**Inputs and Outputs:**
- Input: Volumetric dataset $V$.
- Output: Rendered image $I$.

**Data Structures:**
The volume is decomposed into a wavelet pyramid. Each level $j$ contains an approximation subband $A_j$ and detail subbands (horizontal, vertical, depth, and combinations) $D_j$. The structure is a tree where the root is the coarsest approximation.

**Steps:**
1. **Transform:** Apply a separable 1D wavelet filter bank along each axis of the volume $V$ to generate coefficients [S1, book]. This creates a hierarchy of approximation and detail coefficients.
2. **Quantization and Encoding:** Quantize the coefficients based on desired fidelity. Apply entropy coding to produce a compressed bitstream [S1, book].
3. **Decompression:** Decode the necessary coefficients from the bitstream. This can be done globally or locally [S1, book].
4. **Reconstruction:** Apply the inverse wavelet transform to reconstruct the volume data at the required resolution [S1, book].
5. **Rendering:** Perform volume rendering (e.g., ray casting) on the reconstructed data. In optimized pipelines, rendering may occur directly in the wavelet domain or using reconstructed data to skip homogeneous regions [S1, book].

**Formulas:**
The volume rendering equation is approximated using orthonormal wavelet functions [S1, book]. Let $V(x)$ be the volume function. The rendering integral is computed over the reconstructed volume.

**Parameters:**
- Wavelet basis: Not specified in sources.
- Quantization step: Not specified in sources.

**Edge Cases:**
- Homogeneous areas: The method exploits low frequency variance in these areas to accelerate rendering [S1, book].
- Large volumes: The method is designed for very large volumes, enabling interactive frame rates [S1, book].

## Subtopics

### [Compression Domain Volume Rendering Pipeline](16465-compression-domain-volume-rendering-pipeline.md)
A pipeline for networked volume rendering that compresses data in the wavelet domain, allowing progressive transmission, one-pass decompression, and direct rendering without full decompression. It converts wavelet-transformed volume data into a sequential bitstream for distributed rendering, optimizing for both compression ratio and rendering speed via spatial clustering and significance-based sorting. Unlike traditional methods, this approach avoids full decompression prior to splatting, reducing client-side memory requirements.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Guthe and Strasser used wavelet pyramid encoding to render very large volumes at interactive frame rates on texture mapping hardware [S1, book].

**Scaling:**
The method scales to very large volumes by leveraging the multiresolution hierarchy [S1, book].

**Strengths and Limitations:**
- Strengths: Exploits local frequency variance to accelerate rendering in homogeneous areas; supports interactive frame rates for large volumes [S1, book].
- Limitations: Early algorithms did not focus on acceleration; the potential for acceleration is greater but was not fully exploited in initial works [S1, book].

**When to Use:**
Use when rendering very large volumetric datasets where interactive frame rates are required and the hardware supports texture mapping [S1, book].

## Background
Muraki [232] first applied wavelet transform to volumetric data sets [S1, book]. Gross et al. [99] found an approximate solution for the volume rendering equation using orthonormal wavelet functions [S1, book]. Westermann [344][345] combined volume rendering with wavelet-based compression [S1, book]. Guthe and Strasser [102] recently used the wavelet transform to render very large volumes at interactive frame rates on texture mapping hardware [S1, book].

## Sources
[S1] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
