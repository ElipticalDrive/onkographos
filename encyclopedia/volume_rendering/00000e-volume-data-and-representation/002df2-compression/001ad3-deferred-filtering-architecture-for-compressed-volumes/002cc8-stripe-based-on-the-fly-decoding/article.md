# Stripe-based On-the-fly Decoding

Also called: Fraedrich stripe decoding, Stripe-based Block Decompression (Fraedrich)

## Overview
Stripe-based On-the-fly Decoding is a GPU-accelerated method for decompressing volumetric data blocks directly during the rendering pipeline, enabling local and transient reconstruction of compressed volumes [S1, S2]. It solves the problem of high memory bandwidth and latency associated with full-volume decompression by mapping the decoding of sequential compression schemes onto fragment shaders, where each fragment processes a parallel stripe of voxels [S1, S2]. Unlike [Compression Domain Rendering](960-compression-domain-rendering.md), which often operates directly in the transform domain without full spatial reconstruction, or [CPU-Based Block Decompression](7197-cpu-based-block-decompression.md), which offloads decompression to the host, this method performs the decompression entirely on the GPU in a single render pass, integrating seamlessly with the [Deferred Filtering Architecture for Compressed Volumes](857-deferred-filtering-architecture-for-compressed-volumes.md) [Parent].

## How it works
**Inputs:** Compressed block data (index texture and/or compressed coefficients) and the block dimensions.
**Outputs:** A set of multiple render targets (MRTs) containing the decoded scalar voxel values, formatted as textures for subsequent volume rendering.

**Data Structures:**
1.  **Index Texture:** A 2D texture storing the compressed indices or control data for the block. Its dimensions correspond to the block's spatial extent or the number of stripes.
2.  **Multiple Render Targets (MRTs):** A set of framebuffer attachments. The number of targets depends on the data type and precision; typically, one target per color channel (e.g., RGBA) or per component of the scalar value is used to store the decoded voxels.

**Steps:**
1.  **Stripe Decomposition:** The compressed block is logically decomposed into parallel stripes of voxels. The number of stripes corresponds to the number of fragments that will be generated during the rendering pass [S1, S2].
2.  **Quad Rendering:** A single quad is rendered. The size of this quad matches the dimensions of the index texture (or the number of stripes) [S1, S2].
3.  **Fragment Shader Execution:** Each fragment shader invocation corresponds to one stripe of the block [S1, S2].
    *   The fragment shader reads the compressed data (indices/coefficients) from the index texture.
    *   It applies the specific sequential data compression decoding algorithm to reconstruct the voxel values for that stripe [S1, S2].
    *   The decoded scalar values are written into the different color channels of the multiple render targets [S1, S2].
4.  **Texture Binding:** The resulting MRTs are bound as textures.
5.  **Rendering:** A subsequent render pass uses these decoded textures for volume rendering (e.g., ray casting) [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the fragment shader invoked for each stripe
# Input: index_texture (sampler2D), block_params (uniforms)
# Output: color buffers (MRTs)

def decode_stripe_fragment(stripe_id, index_texture):
    """
    Decodes one stripe of voxels from the compressed block.
    stripe_id: The ID of the current stripe (fragment coordinate).
    index_texture: The texture containing compressed indices/coefficients.
    """
    # 1. Fetch compressed data for the current stripe
    # The exact layout depends on the compression scheme (e.g., ASTC, DXT)
    # [G] Assuming a standard 2D index texture where each texel or group of texels
    # corresponds to a specific spatial region or stripe.
    compressed_data = sample_index_texture(index_texture, stripe_id)
    
    # 2. Decode the stripe
    # [G] The specific decoding logic depends on the compression algorithm.
    # For sequential schemes, this might involve inverse quantization,
    # inverse transform, or entropy decoding.
    decoded_voxels = decode_compressed_block(compressed_data, stripe_id)
    
    # 3. Write decoded values to MRTs
    # [G] Assuming the decoded voxels are scalar values stored in a 1D array
    # for the stripe. The number of voxels per stripe is determined by
    # the block dimensions and stripe orientation.
    for i, voxel_value in enumerate(decoded_voxels):
        # Distribute values across color channels if necessary
        # [G] Example: if voxel_value is a float, it might be packed into RGBA
        # or split across multiple targets.
        write_to_mrt(i % 4, i // 4, voxel_value)

# Main rendering setup
def render_decompression_pass(block_index_texture, num_stripes):
    """
    Sets up the framebuffer and renders the quad to trigger fragment decoding.
    """
    # Create Framebuffer with Multiple Render Targets
    fbo = create_framebuffer(num_targets=4) # Example: 4 targets for RGBA
    attach_textures(fbo, [target_r, target_g, target_b, target_a])
    
    # Bind the index texture as input
    bind_texture(unit=0, texture=block_index_texture)
    
    # Render a quad with size equal to the number of stripes
    # [G] The quad's vertex shader passes the stripe ID (fragment coordinate)
    # to the fragment shader.
    render_quad(width=num_stripes, height=1)
    
    # The MRTs now contain the decoded voxel data
    return [target_r, target_g, target_b, target_a]
```

**Parameters:**
*   **Block Size:** The dimensions of the compressed block (e.g., $4 \times 4 \times 4$ or $8 \times 8 \times 8$ voxels). Not given in the sources.
*   **Stripe Orientation:** The direction along which the block is decomposed into stripes (e.g., along the X, Y, or Z axis). Not given in the sources.
*   **Number of MRTs:** Determined by the data type and precision required for the decoded voxels. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The precision of the decoded values depends on the compression scheme and the bit depth of the MRTs. Lossy compression may introduce artifacts. Not given in the sources.
*   **Boundary Conditions:** The decoding logic must correctly handle the boundaries of the block and the stripes. Not given in the sources.
*   **Compression Scheme Compatibility:** The method is general but requires the compression scheme to be decodable in a fragment shader. Not given in the sources.

## Subtopics

### [Independent Block Encoding](5373-independent-block-encoding.md)
Independent Block Encoding is a local data compression pass where a volume is divided into independent sub-blocks, each encoded separately, potentially using a global representation learned in a preprocessing step. It solves the problem of compressing large volumetric datasets for GPU-based rendering by ensuring that memory usage during encoding remains independent of the total input size, while allowing linear scaling and easy parallelization. It is distinguished from global processing methods by its reliance on local block operations rather than whole-volume transforms, and from stripe-based decoding by being the encoding counterpart that prepares data for on-the-fly GPU reconstruction [Parent].

## Operating characteristics
*   **Time Complexity:** The decoding time is proportional to the number of voxels in the block, as each voxel is processed by a fragment shader. The overhead of the rendering pass is minimal. Not given in the sources.
*   **Memory Complexity:** The memory usage is proportional to the size of the compressed block and the MRTs. The method avoids storing the full decompressed volume in memory, reducing memory pressure. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Enables local and transient reconstruction, reducing memory bandwidth requirements [S1, S2].
    *   Leverages GPU parallelism for decompression [S1, S2].
    *   Integrates well with deferred rendering architectures [Parent].
*   **Limitations:**
    *   Requires the compression scheme to be decodable in a fragment shader [S1, S2].
    *   May introduce overhead due to the rendering pass setup [S1, S2].
*   **When to Use:** When rendering large compressed volumes on the GPU with limited memory bandwidth, and when the compression scheme supports on-the-fly decoding [S1, S2].
*   **When Not to Use:** When the compression scheme is too complex for fragment shader decoding or when full-volume decompression is feasible and faster [S1, S2].

## Background
Stripe-based On-the-fly Decoding was proposed by Fraedrich et al. in 2007 [S1, S2]. It was introduced as a general approach for applying common sequential data compression schemes to blocks of data, allowing them to be decoded on-the-fly by the GPU in a single render pass [S1, S2]. The method was designed to improve upon industry standards such as DXT [Bro01], ETC2 [SP07], and PVRTC [Fen03], and to support the 3D part of the ASTC specification [S1, S2]. It is a key component of the Deferred Filtering Architecture for Compressed Volumes, enabling efficient local and transient reconstruction of compressed volumes [Parent].

## Sources
[S1] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
