# SCALAR-NeRF

Also called: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction

## Overview
SCALAR-NeRF is a coarse-to-fine framework for scalable large-scale neural scene reconstruction that structures the neural representation as an encoder-decoder architecture to avoid the 'foggy' issue in Block-NeRF and enable training on a single GPU [S1, unofficial]. It decomposes expansive scenes into manageable blocks, trains a shared decoder from a coarse global model, and fuses local results using the global model to avoid foggy reconstructions [S1, unofficial]. Unlike Block-NeRF, which blends images using an inverse distance weighting function causing foggy appearance when block centroids are close, SCALAR-NeRF fuses local block outputs with a reference image from the coarse global model [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images capturing a large-scale scene; camera poses for each image; a bounding box defining the scene extent.
*   **Outputs:** A set of local neural radiance field models (one per block) and a coarse global model, which together allow rendering of the entire scene.

**Data Structures:**
The method uses a multi-resolution hash table as the feature encoder, following the Instant-NGP formulation [S1, unofficial].
*   **Hash Table:** A collection of $L$ levels. Each level $l$ has a hash table of size $2^F$ (where $F$ is the feature dimension). The total hash table size is typically $2^{19}$ [S1, unofficial].
*   **Feature Vector:** For a query point, features are retrieved from each level and concatenated: $F = [f_1, \dots, f_L] \in \mathbb{R}^{L \cdot F}$, where $L$ is the number of levels and $F$ is the feature dimension [S1, unofficial].
*   **Decoder:** A shallow Multi-Layer Perceptron (MLP) that maps the feature vector $F$ to volume density $\sigma$ and color $c$: $\sigma, c = Decoder(F)$ [S1, unofficial].

**Steps:**

1.  **Train Coarse Global Model:**
    *   Train a rapid, coarse global model on the entire image dataset using the multi-resolution hash table encoder and a shared decoder [S1, unofficial].
    *   This model retains considerable reconstruction quality despite expedited training [S1, unofficial].
    *   For unbounded scenes, apply space contraction to 3D points $\mathbf{x}$ using the Mip-NeRF360 method [S1, unofficial]:
        $$
        contract(\mathbf{x}) = \begin{cases} \mathbf{x}, & \|\mathbf{x}\| \le 1, \\ \frac{\mathbf{x}}{2 - \frac{1}{\|\mathbf{x}\|}}, & \|\mathbf{x}\| > 1. \end{cases}
        $$
    *   Store the trained decoder weights and the global model's rendering capability [S1, unofficial].

2.  **Partition Scene into Blocks:**
    *   Partition the training images into smaller blocks using KMeans clustering based on camera positions or image content [S1, unofficial].
    *   Define a bounding box for each block.

3.  **Enhance Overlapping Regions:**
    *   Scale up the bounding box of each local block by a scale factor $s_{\text{AABB}}$ to cover intersection areas with neighboring blocks [S1, unofficial].
    *   The optimal value for $s_{\text{AABB}}$ is 1.2 [S1, unofficial]. A too-large factor (e.g., 1.3) damages reconstruction quality and increases hash collision probability [S1, unofficial].

4.  **Train Local Models with Shared Decoder:**
    *   For each local block, initialize a local model with its own hash table encoder [S1, unofficial].
    *   **Share the Decoder:** Inherit the decoder weights from the coarse global model for each local block [S1, unofficial]. This promotes alignment in the feature space of local encoders [S1, unofficial].
    *   **Fix or Fine-tune Decoder:** Fix the shared decoder during local training to eliminate saving its computational graph, enabling larger hash table sizes and faster training [S1, unofficial]. Optionally, fine-tune the shared decoder for better reconstruction quality, though fixing it yields satisfactory results [S1, unofficial].
    *   Train each local model on the images assigned to its block [S1, unofficial].

5.  **Fuse Local Results:**
    *   Synthesize images from each local block model [S1, unofficial].
    *   Render a reference image from the coarse global model for the same viewpoint [S1, unofficial].
    *   Fuse the local block outputs with the reference image from the coarse global model [S1, unofficial]. This leverages the coarse-global to fine-local strategy to discard poorly rendered results and mitigate foggy reconstructions [S1, unofficial].
    *   *Note:* Using an inverse distance weighting (IDW) function for fusion performs worse than the global model and is unsuitable when blocks are close to each other [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def contract(x):
    """
    Space contraction for unbounded scenes (Mip-NeRF360).
    x: numpy array of shape (N, 3)
    """
    norm = np.linalg.norm(x, axis=1, keepdims=True)
    # Avoid division by zero
    norm = np.maximum(norm, 1e-8)
    mask = norm <= 1.0
    contracted = np.where(mask, x, x / (2.0 - 1.0 / norm))
    return contracted

def train_coarse_global_model(images, poses, hash_table_size=2**19, levels=16, feature_dim=2):
    """
    Train a coarse global model on the entire dataset.
    Returns: global_decoder_weights, global_model_instance
    """
    # Initialize multi-resolution hash table encoder
    encoder = MultiResolutionHashEncoder(hash_table_size, levels, feature_dim)
    
    # Initialize shared decoder (shallow MLP)
    decoder = ShallowMLP(input_dim=levels * feature_dim, output_dim=4) # 3 for color, 1 for density
    
    # Training loop for global model
    for epoch in range(num_epochs_global):
        for image, pose in zip(images, poses):
            # Sample rays from image
            rays = sample_rays(image, pose)
            # Contract 3D points on rays
            contracted_rays = contract(rays.points)
            # Encode points
            features = encoder(contracted_rays)
            # Decode to sigma and color
            sigma, color = decoder(features)
            # Compute loss against ground truth image
            loss = compute_loss(color, image)
            # Update encoder and decoder gradients
            update_gradients(encoder, decoder, loss)
            
    return decoder.weights, (encoder, decoder)

def partition_images_kmeans(images, poses, num_blocks):
    """
    Partition training images into blocks using KMeans clustering.
    Returns: list of lists, where each sublist contains indices of images for a block.
    """
    # Extract camera positions from poses
    camera_positions = extract_positions(poses)
    # Apply KMeans clustering
    labels = kmeans(camera_positions, num_blocks)
    # Group image indices by label
    blocks = [[] for _ in range(num_blocks)]
    for i, label in enumerate(labels):
        blocks[label].append(i)
    return blocks

def train_local_block(block_images, block_poses, global_decoder_weights, s_aabb=1.2):
    """
    Train a local model for a specific block.
    Returns: local_encoder, local_decoder_weights (shared with global)
    """
    # Initialize local hash table encoder
    local_encoder = MultiResolutionHashEncoder(hash_table_size=2**19, levels=16, feature_dim=2)
    
    # Initialize local decoder with global weights
    local_decoder = ShallowMLP(input_dim=16 * 2, output_dim=4)
    local_decoder.weights = global_decoder_weights
    
    # Training loop for local block
    for epoch in range(num_epochs_local):
        for image, pose in zip(block_images, block_poses):
            # Sample rays
            rays = sample_rays(image, pose)
            # Contract points
            contracted_rays = contract(rays.points)
            # Encode
            features = local_encoder(contracted_rays)
            # Decode
            sigma, color = local_decoder(features)
            # Compute loss
            loss = compute_loss(color, image)
            # Update ONLY encoder gradients (fix decoder)
            update_gradients(local_encoder, None, loss)
            
    return local_encoder, local_decoder.weights

def fuse_blocks(block_renderings, global_reference_rendering):
    """
    Fuse local block outputs with the coarse global model reference.
    block_renderings: list of rendered images for each block
    global_reference_rendering: reference image from coarse global model
    Returns: final fused image
    """
    # Simple averaging or weighted fusion based on confidence
    # Note: SCALAR-NeRF uses the global model to discard poorly rendered results
    # Implementation detail: The paper states fusing with the global model reference.
    # A simple implementation might be:
    fused_image = np.zeros_like(block_renderings[0])
    for rendering in block_renderings:
        fused_image += rendering
    fused_image /= len(block_renderings)
    
    # Further fusion with global reference to mitigate foggy artifacts
    # The exact fusion formula is not explicitly detailed beyond "fuse them with a reference image"
    # [G] Assuming a simple blend or selection based on global model confidence
    final_image = fused_image # Placeholder for specific fusion logic not fully detailed in source
    
    return final_image

def scalar_nerf_pipeline(images, poses, num_blocks):
    # Step 1: Train Coarse Global Model
    global_decoder_weights, global_model = train_coarse_global_model(images, poses)
    
    # Step 2: Partition Images
    blocks = partition_images_kmeans(images, poses, num_blocks)
    
    local_models = []
    for block_indices in blocks:
        block_images = [images[i] for i in block_indices]
        block_poses = [poses[i] for i in block_indices]
        
        # Step 3 & 4: Train Local Model with Shared Decoder
        local_encoder, _ = train_local_block(block_images, block_poses, global_decoder_weights)
        local_models.append((local_encoder, global_decoder_weights))
        
    # Step 5: Fuse Results (Conceptual)
    # Render from each local model and fuse with global reference
    # This step is conceptual as the exact fusion algorithm is not fully specified in pseudocode form
    return local_models, global_model
```

## Operating characteristics
*   **Time and Memory Complexity:** The method allows training on a single GPU by splitting the scene into blocks, avoiding the need for a large hash table size (e.g., $2^{21}$) which would be infeasible on non-commercial GPUs [S1, unofficial]. The hash table size is $2^{19}$, which is smaller than Instant-NGP's $2^{21}$ but still surpasses its performance [S1, unofficial].
*   **Measured Performance:** On the Tanks and Temples dataset, SCALAR-NeRF outperforms Block-NeRF and Instant-NGP (with hash table size $2^{21}$) in PSNR, SSIM, and LPIPS [S1, unofficial]. It achieves higher quality in detailed areas even with a smaller hash table size [S1, unofficial].
*   **Scaling:** The method scales to large-scale scenes by decomposing them into manageable blocks [S1, unofficial].
*   **Strengths:** Avoids the 'foggy' issue of Block-NeRF; enables single-GPU training for large scenes; superior scalability in reconstruction quality [S1, unofficial].
*   **Limitations:** Requires careful tuning of the AABB scale factor ($s_{\text{AABB}}$); a too-large factor increases hash collisions and degrades quality [S1, unofficial].
*   **When to use:** Large-scale scene reconstruction where single-GPU resources are limited and Block-NeRF's foggy artifacts are problematic [S1, unofficial].
*   **When not to use:** Small-scale scenes where standard NeRF or Instant-NGP suffice; scenes where blocks are very close together and IDW fusion is mistakenly used (SCALAR-NeRF's global model fusion is required) [S1, unofficial].

## Background
SCALAR-NeRF was introduced by Yu Chen and Gim Hee Lee from the Department of Computer Science, National University of Singapore, in November 2023 [S1, unofficial]. It addresses the challenges of large-scale neural scene reconstruction, specifically the limitations of Block-NeRF (foggy appearance) and the memory constraints of increasing hash table sizes in Instant-NGP [S1, unofficial]. The method builds on the encoder-decoder architecture of NeRF and the multi-resolution hash encoding of Instant-NGP [S1, unofficial].

## Sources
[S1] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
