# GPU-resident Neural Compression for TexF

Also called: Neural TexF Compression

## Overview
GPU-resident neural compression for TexF is a method for memory-efficient random access during rendering, using GPU-resident neural compression. It adapts multiresolution hash encoding to TexF. It replaces explicit voxel attributes with a directly queryable compressed field to avoid storing per-voxel lookup entries. Unlike bitstream-oriented TexF compression which uses established 3D attribute codecs for disk storage, GPU-resident neural compression targets memory-efficient random access during rendering.

## How it works
**Inputs and outputs:**
- Input: Surface-aligned TexF (TexF) attributes derived from a textured mesh $M$ via surface voxelization and inverse rendering refinement.
- Output: A compressed neural field comprising quantized latent tables and decoder parameters, enabling direct random access queries for voxel attributes.

**Data structures:**
- **Hash Tables:** Compact tables storing trainable, quantized features mapped from spatial coordinates.
- **Decoder MLP:** A multi-layer perceptron that maps hashed coordinates to attribute values.
- **Latents:** Quantized feature vectors stored in the hash tables.

**Steps:**
1. **Construction:** Construct TexF by voxelizing the mesh surface and transferring source texture attributes. Refine attributes through differentiable rendering to reduce transfer and discretization errors [S1, unofficial].
2. **Adaptation:** Adapt multiresolution hash encoding to the TexF representation. This involves mapping spatial coordinates to trainable features stored in compact hash tables [S1, unofficial].
3. **Quantization:** Quantize the features to reduce memory footprint [S1, unofficial].
4. **Decoding:** Implement lightweight decoding inspired by Neural Texture Compression (NTC). The decoder queries the hash tables using direct hash addressing without explicit per-voxel indices [S1, unofficial].
5. **Refinement:** Optionally refine the compressed field using inverse rendering settings, increasing the number of iterations to 100 [S1, unofficial].

**Formulas:**
- Decoding MACs for 3DNTC:
  $$ C = 144F + 6272 $$
  Where $F$ is the number of feature channels. This includes latent interpolation [S1, unofficial].
- Decoding MACs for NTC (for comparison):
  $$ C_{NTC} = 8192 + 8F_{NTC} $$
  Where $F_{NTC}$ is the feature channel configuration $(8, 12, 8, 12, 16)$ [S1, unofficial].

**Parameters:**
- **Training Steps:** 100,000 steps [S1, unofficial].
- **Loss:** Mean Squared Error (MSE) against TexF attributes [S1, unofficial].
- **Batch Sizes:** 65,536 for grid size $K=2048$; 262,144 for $K=4096$ [S1, unofficial].
- **Optimizer:** Adam [S1, unofficial].
- **Learning Rates:** Start at 0.1 for latents and 0.005 for the decoder, following cosine decay to zero [S1, unofficial].
- **Hash Table Configuration:** Defined by dataset point $(s_1, s_2)$, feature channels $F$, and table sizes for 6 operating points. Table sizes increase with texture area up to fourfold for 2048x2048 textures [S1, unofficial].
- **MLP Widths:** $(2F + 18, 64, 48, 32, 16)$ [S1, unofficial].

**Edge cases and pitfalls:**
- **Precision:** Quantization is required for memory efficiency; the specific quantization scheme is not detailed beyond "quantized features" [S1, unofficial].
- **Boundaries:** TexF construction uses an axis-aligned bounding box discretization into a $K \times K \times K$ grid [S1, unofficial].
- **Degenerate Input:** Inverse rendering optimization can improve input fidelity but may require additional bits to preserve recovered detail, affecting bitstream R-D performance [S1, unofficial].

## Subtopics

### 3D Neural Texture Compression for TexF
TexF compressed directly in 3D using the 3DNTC codec. It trains 3DNTC for 100k steps with MSE loss against TexF attributes. Batch sizes are 65,536 for K=2048 and 262,144 for K=4096. Adam learning rates start at 0.1 for latents and 0.005 for the decoder, following cosine decay to zero. The 3DNTC Hash Table Configuration is defined by dataset point $(s_1, s_2)$, feature channels $F$, and table sizes for 6 operating points. For 2048x2048 textures, table sizes increase with texture area up to fourfold. Decoding MACs for MLP widths $(2F + 18, 64, 48, 32, 16)$ cost $C = 144F + 6272$ MACs per query, including latent interpolation. This averages 8.96 KMACs on MPEG and 9.96 KMACs on AOM. Frame time on MPEG dataset is 10.8 ms (93 FPS); on AOM dataset: 23.1 ms (43 FPS), measured on RTX A6000 hardware. Attribute decoding time is 6.5 ms on MPEG and 15.8 ms on AOM. Per-query arithmetic cost is 8.96 KMACs on MPEG and 9.96 KMACs on AOM. Total encoding time is ~17 min on MPEG and ~28 min on AOM. Compared to NTC, TexF/3DNTC costs $C_{NTC} = 8192 + 8F_{NTC}$ MACs per query, averaging 8.28 KMACs for $F_{NTC} = (8, 12, 8, 12, 16)$ [S1, unofficial].

## Operating characteristics
**Time and memory complexity:**
- **Memory:** Uses direct hash addressing without explicit per-voxel indices, avoiding the addressing overhead of serialized methods (which can account for >80% of resident memory at low texture rates) [S1, unofficial].
- **Decoding Cost:** 8.96 KMACs on MPEG and 9.96 KMACs on AOM [S1, unofficial].

**Measured performance:**
- **Hardware:** RTX A6000 [S1, unofficial].
- **Frame Time:** 10.8 ms (93 FPS) on MPEG dataset; 23.1 ms (43 FPS) on AOM dataset [S1, unofficial].
- **Attribute Decoding Time:** 6.5 ms on MPEG; 15.8 ms on AOM [S1, unofficial].
- **Encoding Time:** ~17 min on MPEG; ~28 min on AOM [S1, unofficial].

**Scaling:**
- Table sizes increase with texture area up to fourfold for 2048x2048 textures [S1, unofficial].

**Strengths and limitations:**
- **Strengths:** Real-time rendering with random access decoding; low-latency; avoids UV parameterization; exploits 3D spatial correlations [S1, unofficial].
- **Limitations:** Higher arithmetic cost than NTC (8.96/9.96 KMACs vs 8.28 KMACs) due to wider decoder input and trilinear interpolation; encoding time is significantly longer than NTC (17-28 min vs 35 s) [S1, unofficial].

**When to use it and when not:**
- Use for memory-efficient random access during rendering on GPU.
- Do not use for disk storage or transmission where bitstream codecs are more appropriate [S1, unofficial].

## Background
Origin: Proposed in "Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields" [S1, unofficial]. It builds on [Bitstream-oriented TexF Compression](10086-bitstream-oriented-texf-compression.md) by offering a complementary scheme for GPU-resident access rather than disk storage.

## Sources
[S1] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
