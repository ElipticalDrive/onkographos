# SCALAR-NeRF

Also called: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction

## Overview
SCALAR-NeRF is a coarse-to-fine framework for scalable large-scale neural scene reconstruction that structures the neural representation as an encoder-decoder architecture to avoid the 'foggy' issue in Block-NeRF and enable training on a single GPU [S1, unofficial]. It decomposes expansive scenes into manageable blocks, trains a shared decoder from a coarse global model, and fuses local results using the global model to avoid foggy reconstructions [S1, unofficial]. Unlike Block-NeRF, which blends images using an inverse distance weighting function causing foggy appearance when block centroids are close, SCALAR-NeRF fuses local block outputs with a reference image from the coarse global model [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images with known camera poses covering a large-scale scene.
*   **Outputs:** A set of local neural radiance field models (one per block) and a coarse global model, which are fused to produce the final reconstruction.

**Data Structures:**
The method uses a multi-resolution hash table encoder (based on Instant-NGP [S1, unofficial]) and a shallow MLP decoder.
*   **Hash Table Encoder:** Stores feature vectors. The feature embedding $F$ is concatenated from the multi-resolution hash table:
    $$ F = [f_1, \dots, f_L] \in \mathbb{R}^{L \cdot F} $$
    where $L$ is the number of levels and $F$ is the feature dimension [S1, unofficial].
*   **Decoder:** A shallow MLP that maps features to density and color:
    $$ \sigma, c = \text{Decoder}(F) $$
    where $\sigma$ is volume density and $c$ is per-point color [S1, unofficial].
*   **Hash Table Size:** The method uses a hash table size of $2^{19}$ [S1, unofficial].

**Algorithm Steps:**

1.  **Train Coarse Global Model:**
    Train a coarse global model on the entire image dataset using the multi-resolution hash table encoder and a shared decoder [S1, unofficial]. This model serves as a reference for fusion and provides the shared decoder for local models.

2.  **Partition Images into Blocks:**
    Partition the training images into smaller blocks using KMeans clustering [S1, unofficial]. This decomposes the expansive scene into manageable spatial blocks [S1, unofficial].

3.  **Scale Bounding Boxes:**
    Enhance overlapping regions across different blocks by scaling up the bounding boxes of each local block. This is controlled by an AABB scale factor $s_{\text{AABB}}$. The optimal value found is 1.2 [S1, unofficial]. A too-large scale factor damages reconstruction quality and increases the probability of hash collisions [S1, unofficial].

4.  **Initialize Local Models with Shared Decoder:**
    For each distinct local block, inherit the decoder from the coarse global model. This shared decoder can be fixed or fine-tuned [S1, unofficial].
    *   **Fixing the Decoder:** Fix the shared decoder in each local model during training to eliminate saving its computational graph, enabling larger hash table sizes and faster training [S1, unofficial].
    *   **Fine-tuning:** Optionally fine-tune the shared decoder for each local model (not strictly required for satisfactory results) [S1, unofficial].
    *   **Local Encoder:** Each local block has its own dedicated hash table encoder [S1, unofficial].

5.  **Train Local Models:**
    Train each local model on its assigned subset of images. The local models use the shared decoder (fixed or fine-tuned) and their own local hash table encoders [S1, unofficial].

6.  **Fuse Local Results:**
    Fuse the outputs from the local models to attain the final reconstruction. This is done by synthesizing images from each block and fusing them with a reference image rendered from the coarse global model [S1, unofficial]. This approach leverages the coarse global model to discard poorly rendered results and mitigate foggy reconstructions, unlike Block-NeRF which uses inverse distance weighting (IDW) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# SCALAR-NeRF Training and Fusion

def train_coarse_global_model(images, poses, hash_table_size=2**19):
    """
    Train a coarse global model on the entire dataset.
    Returns: global_encoder, global_decoder
    """
    # Initialize multi-resolution hash table encoder
    global_encoder = MultiResolutionHashEncoder(hash_table_size=hash_table_size)
    
    # Initialize shared decoder (shallow MLP)
    global_decoder = ShallowMLP()
    
    # Train using standard NeRF loss (e.g., L2 on RGB)
    # The decoder maps features to density and color:
    # F = global_encoder(x)
    # sigma, c = global_decoder(F)
    # loss = L2(rendered_c, ground_truth_c)
    
    # ... training loop ...
    
    return global_encoder, global_decoder

def cluster_images_into_blocks(images, poses, k):
    """
    Partition training images into k blocks using KMeans clustering.
    Returns: list of block_indices, where block_indices[i] is a list of image indices
    """
    # Use camera positions or image features for clustering
    # ... KMeans implementation ...
    return blocks

def scale_aabb(block_indices, original_aabb, scale_factor=1.2):
    """
    Scale up the bounding box of each local block to cover intersection areas.
    Returns: scaled_aabb for each block
    """
    scaled_aabb = []
    for block in blocks:
        # Expand the AABB by the scale_factor
        # ... calculation ...
        scaled_aabb.append(expand_aabb(block.aabb, scale_factor))
    return scaled_aabb

def train_local_models(blocks, scaled_aabb, global_decoder, images, poses):
    """
    Train local models for each block, sharing the global decoder.
    Returns: list of local_encoders
    """
    local_encoders = []
    
    for i, block in enumerate(blocks):
        # Initialize local hash table encoder
        local_encoder = MultiResolutionHashEncoder(hash_table_size=2**19)
        
        # Share the decoder from the global model
        # Option 1: Fix the decoder (recommended for speed and memory)
        local_decoder = global_decoder
        local_decoder.requires_grad = False
        
        # Option 2: Fine-tune the decoder (optional)
        # local_decoder = copy.deepcopy(global_decoder)
        # local_decoder.requires_grad = True
        
        # Train local model on its subset of images
        # The local encoder processes coordinates within the scaled AABB
        # ... training loop ...
        
        local_encoders.append(local_encoder)
        
    return local_encoders

def fuse_results(blocks, local_encoders, global_decoder, global_encoder, images, poses):
    """
    Fuse local block outputs using the coarse global model.
    Returns: final rendered images
    """
    # Render reference image from coarse global model
    global_reference_images = render_from_model(global_encoder, global_decoder, images, poses)
    
    final_images = []
    for i, block in enumerate(blocks):
        # Render image from local model
        local_image = render_from_model(local_encoders[i], global_decoder, block.images, block.poses)
        
        # Fuse local image with global reference image
        # The exact fusion formula is not specified, but it involves
        # leveraging the global model to discard poorly rendered results.
        # Unlike Block-NeRF's IDW, this method uses the global model as a reference.
        fused_image = fuse_with_global_reference(local_image, global_reference_images[i])
        
        final_images.append(fused_image)
        
    return final_images

def contract(x):
    """
    Space contraction for unbounded scenes (from Mip-NeRF360).
    """
    norm_x = torch.norm(x, dim=-1, keepdim=True)
    # contract(x) = x if ||x|| <= 1 else x / (2 - 1/||x||)
    # Implementation detail:
    # mask = (norm_x <= 1.0)
    # contracted_x = torch.where(mask, x, x / (2.0 - 1.0 / norm_x))
    return contracted_x
```

**Parameters:**
*   **Hash Table Size:** $2^{19}$ [S1, unofficial].
*   **AABB Scale Factor ($s_{\text{AABB}}$):** Controls the overlap regions across different blocks. Optimal value found: 1.2 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** A too-large AABB scale factor increases the probability of hash collisions, damaging reconstruction quality [S1, unofficial].
*   **Foggy Appearance:** Using an inverse distance weighting (IDW) function for fusion performs worse than the global model and is unsuitable when blocks are close to each other [S1, unofficial]. SCALAR-NeRF avoids this by using the global model for fusion [S1, unofficial].
*   **Memory Constraints:** Simply increasing the hash table size slows down training and makes it infeasible to fit into a single GPU. SCALAR-NeRF splits scenes into blocks to allow finer details without increasing hash table size [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method enables training on a single GPU by splitting the scene into blocks and sharing the decoder. This reduces memory requirements compared to training a single large model with a larger hash table (e.g., $2^{21}$) [S1, unofficial].
*   **Measured Performance:** On the Tanks and Temples dataset, SCALAR-NeRF with a hash table size of $2^{19}$ surpasses Block-NeRF and Instant-NGP with a hash table size of $2^{21}$ in terms of PSNR, SSIM, and LPIPS [S1, unofficial].
*   **Strengths:** Scalability for large-scale scenes, ability to train on a single GPU, avoidance of foggy artifacts in overlapping regions.
*   **Limitations:** Requires KMeans clustering for block partitioning. The quality of the coarse global model impacts the final fusion.
*   **When to Use:** Large-scale scene reconstruction where memory is limited and foggy artifacts from block blending are a concern.
*   **When Not to Use:** Small-scale scenes where a single global model fits in memory, or when high-speed training is prioritized over reconstruction quality (though SCALAR-NeRF is also fast due to fixed decoder).

## Background
SCALAR-NeRF was introduced by Yu Chen and Gim Hee Lee from the Department of Computer Science, National University of Singapore, in November 2023 [S1, unofficial]. It addresses the challenges of large-scale neural scene reconstruction, specifically the limitations of Block-NeRF (foggy appearance) and the memory constraints of increasing hash table sizes in methods like Instant-NGP [S1, unofficial]. The method builds on the encoder-decoder architecture of NeRF and the multi-resolution hash encoding of Instant-NGP [S1, unofficial].

## Sources
[S1] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
