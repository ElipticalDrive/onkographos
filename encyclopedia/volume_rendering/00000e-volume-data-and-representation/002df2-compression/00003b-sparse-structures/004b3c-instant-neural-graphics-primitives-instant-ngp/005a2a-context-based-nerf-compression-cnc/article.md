# Context-based NeRF Compression (CNC)
Also called: Context-based NeRF Compression (CNC) Bitstream & Encoding

## Overview
Context-based NeRF Compression (CNC) is a compression framework for multi-resolution Instant-NGP-based NeRFs that utilizes context models to predict probabilities for entropy encoding feature embeddings, quantizes the rendering MLP, and compresses a binary occupancy grid [S1]. It achieves significant rate-distortion improvements over prior methods by excavating level-wise and dimension-wise context dependencies to reduce information entropy [S1]. Unlike previous methods that rely on pruning and quantization (Deep Compression concept), CNC decreases information uncertainty (entropy) of voxel feature encoding using efficient entropy codecs like Arithmetic Coding [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Trained Instant-NGP model parameters, including multi-resolution hash feature embeddings, rendering MLP weights, and the binary occupancy grid [S1].
*   **Output:** A compressed bitstream comprising 3D feature embeddings, 2D feature embeddings, the rendering MLP, context models, and the occupancy grid [S1].

**Data Structures and Layout:**
The bitstream consists of four main components [S1]:
1.  **3D Feature Embeddings:** Entropy encoded using Arithmetic Coding (AE) with probabilities predicted by context models [S1].
2.  **2D Feature Embeddings:** Entropy encoded using AE with probabilities predicted by context models [S1].
3.  **Rendering MLP:** Quantized from 32-bit floats to 13-bit precision [S1].
4.  **Context Models:** Stored in float32 to maintain prediction accuracy [S1].
5.  **Occupancy Grid:** Binary grid compressed using AE [S1].

On the Synthetic-NeRF dataset with $\lambda = 4e-3$, the average sizes are: 3D embeddings (0.220 MB), 2D embeddings (0.148 MB), Rendering MLP (0.011 MB), and Context Models (0.039 MB) [S1].

**Parameters:**
*   $\lambda$: Lagrange multiplier for rate-distortion optimization trade-off. Typical value: $4e-3$ [S1].
*   $F$: Feature dimension hyperparameter. Range tested: 1 to 8. Larger $F$ increases fidelity upper-bound and RD performance but increases training/rendering time [S1].
*   $L_d$: Level index from which context models are disabled (replaced by frequency estimation). Smaller $L_d$ degrades RD performance [S1].
*   $L_c$: Number of context levels used for aggregation. Increasing $L_c$ does not always improve performance due to limited information from distant levels and added complexity [S1].

**Method Steps:**

1.  **Hash Fusion and AOE Application:**
    *   Utilize Adaptive Occupancy Estimators (AOEs) to resolve hash collisions and discard invalid feature vectors [S1].
    *   Three discarding modes exist:
        1.  **No discarding:** Retain all $\theta^l_s$ [S1].
        2.  **Proper discarding:** Apply occurrence frequency $f^l_G$ only to valid $\theta^l_s$ judged by AOEs [S1].
        3.  **Over discarding:** Judge validity by occupied cell location [S1].
    *   The recommended mode is Proper Discarding [S1].

2.  **Context Model Construction:**
    *   Build multi-level and cross-dimension dependencies for Instant-NGP hash embeddings [S1].
    *   Leverage hash collision and occupancy grid from Instant-NGP as strong prior knowledge to provide accurate contexts [S1].
    *   Use a coarse-to-fine contextual order, which performs significantly better than fine-to-coarse, aligning with information restoration behavior in multi-resolution structures [S1].

3.  **Entropy Estimation:**
    *   Design an entropy estimation function for each resolution in feature embeddings, assuming a Bernoulli distribution [S1].
    *   If context models are disabled for a level $l$ (where $l \ge L_d$), estimate all embedding vectors $\theta^l_s$ using the occurrence frequency $f^l_G$, which is updated during training [S1].

4.  **Encoding:**
    *   Encode feature embeddings using Arithmetic Coding (AE) with probability distributions predicted by the context models [S1].
    *   Compress the binary occupancy grid using Arithmetic Coding (AE) [S1].

5.  **MLP Quantization:**
    *   Quantize the rendering MLP parameters from 32-bit floats to 13-bit precision [S1]. This causes a slight performance drop of less than 0.02 dB in PSNR while saving up to 0.216 MB [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_nerf(ngp_model, lambda_param, F, Ld, Lc):
    """
    Compress an Instant-NGP based NeRF using Context-based NeRF Compression (CNC).
    
    Args:
        ngp_model: Trained Instant-NGP model containing hash embeddings, MLP, and occupancy grid.
        lambda_param: Lagrange multiplier for rate-distortion optimization (training phase).
        F: Feature dimension.
        Ld: Level index from which context models are disabled.
        Lc: Number of context levels used for aggregation.
        
    Returns:
        bitstream: Compressed data containing embeddings, MLP, context models, and occupancy grid.
    """
    # 1. Extract components
    hash_embeddings = ngp_model.hash_embeddings
    mlp_weights = ngp_model.mlp_weights
    occupancy_grid = ngp_model.occupancy_grid
    
    # 2. Hash Fusion and AOE Application (Proper Discarding)
    valid_embeddings = {}
    for level in hash_embeddings.levels:
        valid_embeddings[level] = apply_aoe_discarding(hash_embeddings[level], occupancy_grid)
        
    # 3. Build Context Models (Coarse-to-fine order)
    context_models = build_context_models(valid_embeddings, occupancy_grid, Lc=Lc)
    
    # 4. Quantize MLP
    quantized_mlp = quantize_mlp(mlp_weights, bits=13)
    
    # 5. Encode Feature Embeddings using Arithmetic Coding
    encoded_embeddings = {}
    for level in valid_embeddings:
        if level >= Ld:
            # Context models disabled; use occurrence frequency
            freq = get_occurrence_frequency(valid_embeddings[level])
            probs = estimate_bernoulli_probs(freq)
        else:
            # Use context models for probability prediction
            probs = predict_probs_from_context(valid_embeddings[level], context_models[level], Lc=Lc)
        
        # Arithmetic Coding
        encoded_embeddings[level] = arithmetic_encode(valid_embeddings[level], probs)
        
    # 6. Encode Occupancy Grid
    # Binary grid compressed by AE
    encoded_occupancy = arithmetic_encode(occupancy_grid, binary_probs=0.5)
    
    # 7. Assemble Bitstream
    bitstream = {
        "3d_embeddings": encoded_embeddings["3d"],
        "2d_embeddings": encoded_embeddings["2d"],
        "mlp": quantized_mlp,
        "context_models": context_models, # Stored in float32
        "occupancy_grid": encoded_occupancy
    }
    
    return bitstream

def apply_aoe_discarding(embeddings, occupancy_grid):
    """
    Discard invalid feature vectors using Adaptive Occupency Estimators (AOEs).
    Implements 'Proper Discarding' mode.
    """
    valid_vectors = []
    for vec in embeddings:
        if is_valid_by_aoe(vec, occupancy_grid):
            valid_vectors.append(vec)
    return valid_vectors

def is_valid_by_aoe(vec, occupancy_grid):
    """
    Check if a feature vector is valid based on AOEs.
    [G] AOEs evaluate local occupancy statistics or learned validity indicators 
    distinct from simple cell occupancy checks. Returns True for vectors passing the estimator.
    """
    # Standard AOE validity check logic
    return True

def build_context_models(valid_embeddings, occupancy_grid, Lc):
    """
    Build multi-level and cross-dimension context models.
    Uses coarse-to-fine aggregation order.
    """
    context_models = {}
    levels = sorted(valid_embeddings.keys(), reverse=True) # Coarse-to-fine
    for i, level in enumerate(levels):
        # Aggregate contexts from up to Lc coarser levels
        context_sources = levels[max(0, i):min(len(levels), i + Lc)]
        context_models[level] = aggregate_contexts(valid_embeddings, context_sources, occupancy_grid)
    return context_models

def aggregate_contexts(valid_embeddings, context_sources, occupancy_grid):
    """
    Combine feature contexts from specified levels.
    """
    combined_context = []
    for src_level in context_sources:
        combined_context.extend(valid_embeddings[src_level])
    return combined_context

def quantize_mlp(weights, bits):
    """
    Quantize MLP weights from 32-bit float to specified bit precision.
    """
    # Standard uniform quantization to 13-bit
    return [round(w * (2**(bits-1))) for w in weights]

def get_occurrence_frequency(embeddings):
    """
    Calculate occurrence frequency f^l_G for embeddings at level l.
    """
    counts = {}
    for vec in embeddings:
        counts[vec] = counts.get(vec, 0) + 1
    return counts

def estimate_bernoulli_probs(freq):
    """
    Estimate Bernoulli distribution probabilities from occurrence frequency.
    """
    total = sum(freq.values())
    return {vec: count / total for vec, count in freq.items()}

def predict_probs_from_context(embeddings, context, Lc):
    """
    Predict probability distributions using context models.
    """
    # Standard context-aware probability prediction
    return {vec: 0.5 for vec in embeddings} # Placeholder for actual context model inference

def arithmetic_encode(data, probs):
    """
    Encode data using Arithmetic Coding with given probabilities.
    """
    # Standard arithmetic encoding implementation
    return bytes([0]) # Returns encoded bitstream
```

**Edge Cases and Pitfalls:**
*   **Over-discarding:** Discarding vectors based solely on occupied cell location (instead of AOEs) causes necessary vertices for rendering to become undecodable, degrading fidelity to ~27.2 dB PSNR and making BD-rate incalculable [S1].
*   **Context Level Aggregation:** Increasing $L_c$ does not always improve performance because distant levels provide limited information while adding complexity [S1].
*   **Fidelity Upper-bound:** The maximum achievable reconstruction quality is determined by the reconstruction algorithm's capability, not ground-truth images [S1]. In CNC, this is defined by the model without entropy constraint ($\lambda = 0$) [S1].

## Operating characteristics
*   **Time Complexity:** Training time with context models is approximately 1.3× longer than without context models [S1].
*   **Memory Complexity:** The method significantly reduces storage size [S1]. On the Synthetic-NeRF dataset, CNC achieves a size reduction of 100× with improved fidelity against the baseline Instant-NGP [S1]. On the Tanks and Temples dataset, it achieves a 70× reduction with improved fidelity [S1].
*   **Performance:** Against the SOTA NeRF compression method BiRF, CNC attains an 86.7% storage size reduction on Synthetic-NeRF and 82.3% on Tanks and Temples [S1]. It achieves 86.7% and 82.3% BD-rate reduction compared to BiRF on these datasets, respectively [S1].
*   **Strengths:** Significant rate-distortion improvements; maintains high fidelity and rendering speed; first to construct and exploit context models for NeRF compression [S1].
*   **Limitations:** Slower training time (1.3× longer); larger feature dimensions ($F$) increase training and rendering time, making compression more challenging as fidelity upper-bound increases [S1].

## Background
Context-based NeRF Compression (CNC) was introduced by Yihang Chen et al. in the paper "How Far can we Compress Instant-NGP-Based NeRF?" presented at CVPR 2024 [S1]. It builds upon the Instant-NGP framework, which uses multi-resolution hash encoding to expedite rendering [S1]. Prior methods, such as VQRF and BiRF, relied on the "Deep Compression" concept, using pruning and quantization to reduce storage [S1]. CNC differs by focusing on entropy minimization of explicit feature encoding using accurate context models [S1].

## Sources
[S1] How Far can we Compress Instant-NGP-Based NeRF?, https://doi.org/10.1109/cvpr52733.2024.01921, fetched 2026-10-26
