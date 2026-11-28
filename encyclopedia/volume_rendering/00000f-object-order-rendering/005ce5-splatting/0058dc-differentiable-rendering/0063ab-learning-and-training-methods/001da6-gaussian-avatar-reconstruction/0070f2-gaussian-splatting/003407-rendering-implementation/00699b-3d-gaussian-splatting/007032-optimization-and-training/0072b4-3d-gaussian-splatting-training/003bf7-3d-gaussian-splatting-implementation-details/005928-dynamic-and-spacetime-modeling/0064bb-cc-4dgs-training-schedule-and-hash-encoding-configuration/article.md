# CC-4DGS Training Schedule and Hash Encoding Configuration

Also called: CC-4DGS implementation details, Canonical Deformation Field (CDF) with Compression (CCA) setup

## Overview
CC-4DGS is a training and encoding configuration for dynamic Gaussian splatting that replaces learnable XOR-based hash tables with modulation-based universal hash functions, dense hash encoding, and lightweight neural decoders, combined with a three-stage optimization schedule and dynamic-score gating [S1]. It distinguishes itself from sibling methods like Swift4D by eliminating large XOR-based hash tables to reduce storage, instead mapping hash-encoded integers to continuous features via neural decoders [S1]. This approach achieves a superior trade-off among storage efficiency, reconstruction quality, and rendering speed compared to grid-based hash table approaches [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view video sequences (e.g., N3DV or Technicolor datasets), initial COLMAP point cloud, hyperparameters for hash encoding and optimization schedule.
*   **Outputs:** Optimized Gaussian parameters (position, rotation, scale, opacity, color) in canonical space, deformation field parameters, dynamic scores, and neural decoder weights.

**Data Structures:**
*   **Multi-resolution Grids:** A set of 4 or 5 hash grids [S1].
*   **Hash Function Parameters:**
    *   `m`: Bucket size (integer). Must be sufficiently large to limit collisions [S1].
    *   `P`: Prime number (integer). Used alongside `m` to limit hash collisions [S1].
*   **Neural Decoders:** Lightweight neural networks, one per grid level, mapping hash-encoded integers to continuous features [S1].
*   **Dynamic Score:** A learned scalar `d` per Gaussian [S1].
*   **Threshold:** Scalar `ζ` for dynamic score gating [S1].

**Numbered Steps:**

1.  **Stage 1: Coarse Initialization (5K iterations).**
    *   Establish the initial canonical point cloud using COLMAP.
    *   Optimize base Gaussian parameters (position, rotation, scale, opacity, color) without deformation [S1].

2.  **Stage 2: Dynamic Score Fitting (3K iterations).**
    *   Fit the per-Gaussian dynamic score `d` [S1].
    *   This stage prepares the dynamic-score gating mechanism [S1].

3.  **Stage 3: Main Optimization (20K iterations).**
    *   Jointly optimize the deformation field and the Gaussian set [S1].
    *   Total iterations per N3DV scene: 28K (5K + 3K + 20K) [S1].
    *   **Dynamic-Score Gating:** During rendering and optimization, deformation is applied only to Gaussians whose learned dynamic score `d` exceeds the threshold `ζ` [S1]. This avoids unnecessary computation in static regions [S1].

**Hash Encoding Configuration:**
*   Unlike conventional hash-grid approaches that store vertex embeddings in large XOR-based hash tables, CC-4DGS stores no such learnable hash tables to reduce storage [S1].
*   It employs modulation-based universal hash functions with dense hash encoding [S1].
*   Lightweight neural decoders at each grid level map hash-encoded integers to continuous features [S1].
*   Each grid level uses a sufficiently large bucket size `m` and a prime `P` to limit hash collisions [S1]. Preliminary tests showed that smaller `m` values noticeably degrade PSNR [S1].
*   Due to GPU memory constraints, the number of multi-resolution grids is restricted to four or five [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CC-4DGS Training Configuration and Hash Encoding

def initialize_hash_grids(num_grids=4, bucket_size_m=1000000, prime_P=1000000007):
    """
    Initialize modulation-based universal hash functions.
    Note: Exact implementation of modulation-based universal hash function is not provided in sources.
    We assume a standard hash function h(x) = ((x * A) + B) % P, where A, B are random constants.
    """
    grids = []
    for i in range(num_grids):
        # Create a neural decoder for this grid level
        decoder = LightweightNeuralDecoder()
        grids.append({
            'level': i,
            'decoder': decoder,
            'bucket_size': bucket_size_m,
            'prime': prime_P
        })
    return grids

def hash_encode(position, grids):
    """
    Encode a 3D position using dense hash encoding and neural decoders.
    """
    features = []
    for grid in grids:
        # Hash the position to an integer index
        # Note: Exact hash function details are not provided. 
        # Assuming a standard hash function for demonstration.
        x, y, z = position
        # Modulation-based universal hash function
        # h = ((x * A + y * B + z * C) % P) % m
        # Where A, B, C are random constants, P is prime, m is bucket size
        # This is a placeholder for the actual modulation-based hash function
        hash_index = (x * 12345 + y * 67890 + z * 11111) % grid['prime'] % grid['bucket_size']
        
        # Map hash-encoded integer to continuous features using neural decoder
        feature = grid['decoder'](hash_index)
        features.append(feature)
    return concatenate(features)

def dynamic_score_gate(gaussian, threshold_zeta):
    """
    Determine if deformation should be applied based on dynamic score.
    """
    return gaussian.dynamic_score > threshold_zeta

def train_cc_4dgs(dataset, initial_point_cloud, num_iterations_stage1=5000, 
                  num_iterations_stage2=3000, num_iterations_stage3=20000, 
                  threshold_zeta=0.5, bucket_size_m=1000000, prime_P=1000000007):
    """
    Main training loop for CC-4DGS.
    """
    # Initialize hash grids
    grids = initialize_hash_grids(num_grids=4, bucket_size_m=bucket_size_m, prime_P=prime_P)
    
    # Initialize Gaussians from COLMAP
    gaussians = initialize_gaussians(initial_point_cloud)
    
    # Initialize deformation field and dynamic scores
    deformation_field = InitializeDeformationField()
    for g in gaussians:
        g.dynamic_score = 0.0  # Initialize dynamic score
    
    # Stage 1: Coarse Initialization
    for i in range(num_iterations_stage1):
        # Render image
        rendered_image = render(gaussians, deformation_field, grids, time=t)
        # Compute loss (L1 + D-SSIM)
        loss = compute_loss(rendered_image, ground_truth_image)
        # Backpropagate and update base Gaussian parameters
        update_parameters(gaussians, loss)
    
    # Stage 2: Dynamic Score Fitting
    for i in range(num_iterations_stage2):
        # Render image
        rendered_image = render(gaussians, deformation_field, grids, time=t)
        # Compute loss
        loss = compute_loss(rendered_image, ground_truth_image)
        # Backpropagate and update dynamic scores
        update_parameters(gaussians.dynamic_score, loss)
    
    # Stage 3: Main Optimization
    for i in range(num_iterations_stage3):
        # Render image with dynamic-score gating
        active_gaussians = [g for g in gaussians if dynamic_score_gate(g, threshold_zeta)]
        rendered_image = render(active_gaussians, deformation_field, grids, time=t)
        # Compute loss
        loss = compute_loss(rendered_image, ground_truth_image)
        # Backpropagate and update deformation field and Gaussian parameters
        update_parameters(deformation_field, loss)
        update_parameters(gaussians, loss)
    
    return gaussians, deformation_field, grids

def render(gaussians, deformation_field, grids, time):
    """
    Render image using differentiable Gaussian rasterization.
    """
    # Apply deformation to active Gaussians
    for g in gaussians:
        if dynamic_score_gate(g, threshold_zeta):
            # Get hash-encoded features
            features = hash_encode(g.position, grids)
            # Apply deformation using features and deformation field
            g.position = deformation_field(g.position, features, time)
    
    # Differentiable rasterization
    image = differentiable_rasterize(gaussians)
    return image
```

**Parameters with Typical Values:**
*   `m` (Bucket size): Not given in the sources, but must be sufficiently large [S1].
*   `P` (Prime): Not given in the sources, but used alongside `m` [S1].
*   `ζ` (Threshold): Not given in the sources, but used for dynamic-score gating [S1].
*   Number of multi-resolution grids: 4 or 5 [S1].
*   Stage 1 iterations: 5K [S1].
*   Stage 2 iterations: 3K [S1].
*   Stage 3 iterations: 20K [S1].
*   Total iterations per N3DV scene: 28K [S1].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Smaller `m` values noticeably degrade PSNR [S1].
*   **GPU Memory:** The number of multi-resolution grids is restricted to 4 or 5 due to GPU memory constraints [S1].
*   **Static Regions:** Without dynamic-score gating, unnecessary computation occurs in static regions [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Storage cost is significantly reduced compared to Swift4D. The CDF component alone reduces model size to a few tens of megabytes [S1]. When combined with CCA, the total storage further decreases to approximately 30 MB [S1].
*   **Measured Performance:**
    *   **N3DV Dataset:** PSNR in the low 32 dB range, SSIM approximately 0.94 [S1].
    *   **Technicolor Dataset (CDF):** PSNR 33.29 dB, SSIM 0.912, rendering speed 64.7 FPS [S1].
    *   **Technicolor Dataset (CDF+CCA):** PSNR 33.30 dB, SSIM 0.912, rendering speed 64.6 FPS, storage ~32 MB [S1].
    *   **Hardware:** Single NVIDIA RTX 4090 GPU [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Superior trade-off among storage efficiency, reconstruction quality, and rendering speed [S1]. Eliminates large XOR-based hash tables to reduce storage [S1].
*   **Limitations:** Requires careful tuning of `m` to avoid hash collisions [S1]. Limited by GPU memory constraints on the number of multi-resolution grids [S1].
*   **When to Use:** When storage efficiency is a priority and real-time rendering is required [S1].
*   **When Not to Use:** When higher reconstruction quality is needed at the cost of increased storage [S1].

## Background
CC-4DGS was introduced in the paper "CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting" [S1]. It builds on prior work such as 4DGS [S1] and Swift4D [S1]. The method is evaluated on the N3DV dataset [S1] and the Technicolor Light Field dataset [S1].

## Sources
[S1] CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3721136, fetched 2026-10-27
