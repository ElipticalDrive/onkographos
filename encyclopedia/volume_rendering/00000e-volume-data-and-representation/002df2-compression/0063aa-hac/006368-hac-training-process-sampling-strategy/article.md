# HAC++ Training Process & Sampling Strategy

Also called: HAC++ Sampling Strategy, HAC++ Training

## Overview
HAC++ employs a staged training procedure with a specific sampling strategy to stabilize entropy model training and prevent out-of-memory (OOM) issues during the compression of 3D Gaussian Splatting (3DGS) [S1]. It distinguishes itself from standard 3DGS compression by decoupling the geometric fitting phase from the entropy coding phase, using noise injection and anchor subsampling to ensure the quantization-aware context models converge reliably [S1].

## How it works
The method consists of a three-stage training schedule and a per-iteration sampling strategy.

### Inputs and Outputs
*   **Inputs:**
    *   Initial Scaffold-GS model parameters [S1].
    *   Hyperparameters: RD trade-off $\lambda \in [0.5 \times 10^{-3}, 4 \times 10^{-3}]$, quantization scales $Q_0 = \{1, 0.001, 0.2\}$ for attributes $\{f_a, l, o\}$, intra-steps $N_c = 5$, anchor feature dimension $D_a = 50$, offsets per anchor $K = 10$, hash grid feature dimension $D_h = 4$ [S1].
    *   Hash grid: Mixed 3D-2D structured binary grid (12 levels of 3D embeddings $16 \dots 512$, 4 levels of 2D embeddings $128 \dots 1024$; max table sizes $2^{13}$ and $2^{15}$) [S1].
*   **Outputs:**
    *   Trained HAC++ model with optimized anchor attributes, hash grid, and context models [S1].

### Data Structures
*   **Anchor Attributes ($A$):** Contains feature $f_a$ (dim $D_a=50$), location $l$, and opacity $o$ [S1].
*   **Hash Grid ($H$):** Stores features of dimension $D_h=4$ [S1].
*   **Context Models:** Combined into a single 3-layer MLP with ReLU activation for quantization ($MLP_q$) and context ($MLP_c$); $MLP_a$ consists of multiple MLPs for intra-step inputs [S1].

### Numbered Steps
1.  **Stage 1: Initial Fitting (Iterations 0–3k).** Train the original Scaffold-GS model without HAC++ components to establish a stable geometric baseline for anchor attributes and spawning [S1].
2.  **Stage 2: Quantization Adaptation (Iterations 3k–10k).**
    *   Pause anchor spawning between iterations 3k and 4k to prevent instability from sudden quantization introduction [S1].
    *   Add noise to anchor attributes $A$ to help the model adapt to quantization [S1].
    *   Apply only the base quantization scale $Q_0$ without using the residual $r$ (hash grid is not yet active) [S1].
3.  **Stage 3: Joint Training (Iterations 10k–30k).** Fully integrate the HAC++ framework. Jointly train the context models and hash grid using the full entropy loss [S1].
4.  **Per-Iteration Sampling.** In every iteration, randomly select 5% of all anchors to perform entropy training. The remaining 95% are skipped for entropy loss calculation to reduce memory usage and training time [S1].
5.  **RD Trade-off Adjustment.** Adjust $\lambda$ within the range $[0.5 \times 10^{-3}, 4 \times 10^{-3}]$ to control the bitrate [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_haccc_scaffold_gs(scaffold_gs_model, total_iterations=30000):
    """
    Trains the HAC++ model using the staged process and sampling strategy.
    """
    # Hyperparameters from [S1]
    lambda_min = 0.5e-3
    lambda_max = 4e-3
    Q0_fa = 1.0
    Q0_l = 0.001
    Q0_o = 0.2
    sampling_ratio = 0.05
    spawn_pause_end_iter = 4000
    spawn_pause_start_iter = 3000
    stage2_end_iter = 10000
    
    # Initialize HAC++ components (hash grid, context MLPs)
    hash_grid = initialize_hash_grid()
    context_mlp = initialize_context_mlp()
    
    for iter in range(total_iterations):
        # 1. Stage 1: Pure Scaffold-GS training (0-3k)
        if iter < 3000:
            loss = train_scaffold_gs(scaffold_gs_model)
            update_parameters(scaffold_gs_model, loss)
            continue
            
        # 2. Stage 2: Quantization Adaptation (3k-10k)
        if iter < stage2_end_iter:
            # Pause spawning during transition
            if spawn_pause_start_iter <= iter < spawn_pause_end_iter:
                disable_anchor_spawning()
            else:
                enable_anchor_spawning()
                
            # Add noise to attributes for quantization adaptation
            add_noise_to_anchor_attributes(scaffold_gs_model, scale=Q0)
            
            # Quantize using Q0 only, no hash grid residuals yet
            quantized_attrs = quantize(scaffold_gs_model.attributes, Q0)
            
            # Compute loss based on quantized attributes (no entropy context yet)
            loss = compute_scaffold_loss(scaffold_gs_model, quantized_attrs)
            update_parameters(scaffold_gs_model, loss)
            continue
            
        # 3. Stage 3: Joint HAC++ Training (10k-30k)
        # Adjust lambda for variable bitrate
        lambda_val = interpolate_lambda(iter, 10000, 30000, lambda_min, lambda_max)
        
        # Sampling Strategy: Select 5% of anchors
        all_anchors = get_all_anchors(scaffold_gs_model)
        sampled_anchors = random_sample(all_anchors, ratio=sampling_ratio)
        
        # Forward pass with full HAC++ framework
        # Retrieve hash grid features for sampled anchors
        hash_features = query_hash_grid(hash_grid, sampled_anchors)
        
        # Predict probabilities using context models
        probs = predict_probabilities(context_mlp, hash_features, sampled_anchors)
        
        # Compute entropy loss for sampled anchors
        entropy_loss = compute_entropy_loss(sampled_anchors, probs, Q0)
        
        # Compute Scaffold and Hash losses for all anchors (or sampled subset depending on implementation)
        # Note: [S1] implies entropy training is sampled, but geometric fitting usually uses all.
        # We assume geometric loss uses all anchors for stability, entropy uses 5%.
        geometric_loss = compute_scaffold_and_hash_loss(scaffold_gs_model, hash_grid)
        
        # Total Loss
        total_loss = geometric_loss + lambda_val * entropy_loss
        
        update_parameters(scaffold_gs_model, hash_grid, context_mlp, total_loss)

def quantize(attributes, Q0):
    """
    Applies quantization scale Q0 to attributes.
    Q0 is a dict: {'fa': 1.0, 'l': 0.001, 'o': 0.2}
    """
    fa_q = round(attributes.fa / Q0['fa']) * Q0['fa']
    l_q = round(attributes.l / Q0['l']) * Q0['l']
    o_q = round(attributes.o / Q0['o']) * Q0['o']
    return {'fa': fa_q, 'l': l_q, 'o': o_q}
```

### Parameters and Typical Values
*   **$\lambda$ (RD Trade-off):** $0.5 \times 10^{-3}$ to $4 \times 10^{-3}$ [S1].
*   **$Q_0$ (Quantization Scale):** $1.0$ for $f_a$, $0.001$ for $l$, $0.2$ for $o$ [S1].
*   **$N_c$ (Intra Steps):** 5 [S1].
*   **$D_a$ (Anchor Feature Dim):** 50 [S1].
*   **$K$ (Offsets per Anchor):** 10 [S1].
*   **$D_h$ (Hash Grid Feature Dim):** 4 [S1].
*   **Sampling Ratio:** 5% of anchors per iteration [S1].

### Edge Cases and Pitfalls
*   **Out-of-Memory (OOM):** Using all anchors for entropy training causes OOM and prolonged training. Mitigation: Sample 5% of anchors [S1].
*   **Training Instability:** The Scaffold-GS feature bank causes instability. Mitigation: Disable the feature bank [S1].
*   **Spawning Instability:** Sudden quantization introduction destabilizes anchor spawning. Mitigation: Pause spawning between iterations 3k and 4k [S1].
*   **Precision:** Quantization scales are small for location ($0.001$) and opacity ($0.2$), requiring careful handling of floating-point precision during rounding [S1].

## Subtopics

### Progressive Compression Training Loss
A variant of the training loss used in PCGS, which extends HAC++ to support progressive bitrates [S2]. It combines entropy loss based on incremental bit consumption with scaffold and hash losses [S2]. At each iteration, a level $s$ and its corresponding $\lambda_s$ are randomly sampled [S2]. The entropy loss $L_{entropy}$ for level $s$ is calculated as:
$$
L_{entropy} = \sum_{i=1}^N b_i^s
$$
where $b_i^s$ for anchor $i$ is:
$$
b_i^s = m^a_s \Delta m^a_s \sum_{f \in \{f^a, l\}} \sum_{j=1}^D -\log_2 p_G(\hat{f}_{s,j}) + m^a_s (1 - \Delta m^a_s) \sum_{f \in \{f^a, l\}} \sum_{j=1}^D -\log_2 p_T(\hat{f}_{s,j}) + \sum_{k=1}^K \Delta m^g_{s,k} \sum_{j=1}^3 -\log_2 p_G(\hat{o}_{s,3k+j})
$$
Here, $\Delta m^a_s = 1$ indicates a newly decoded anchor (Gaussian entropy $p_G$), $\Delta m^a_s = 0$ indicates a refined anchor (Trinomial entropy $p_T$), and $\Delta m^g_{s,k} = 1$ indicates a newly added Gaussian offset (Gaussian entropy) [S2]. The total loss at level $s$ is:
$$
\text{Loss} = L_{Scaffold} + L_{hash} + \frac{\lambda_s}{N_a (D + 6 + 3K)} L_{entropy}
$$
where $L_{Scaffold}$ and $L_{hash}$ are from HAC++ [S2].

## Operating characteristics
*   **Time Complexity:** Reduced by 95% per iteration for entropy training due to the 5% sampling strategy [S1]. Total iterations are 30k, consistent with Scaffold-GS [S1].
*   **Memory Complexity:** Significantly reduced by sampling anchors, preventing OOM on standard GPUs (e.g., NVIDIA L40s with 48 GB) [S1].
*   **Performance:** Achieves >100× size reduction compared to vanilla 3DGS and >20× compared to Scaffold-GS on average [S1].
*   **Fidelity:** Surpasses Scaffold-GS in fidelity due to entropy loss regularization and increased $D_a=50$ [S1].
*   **Hardware:** Implemented on a single NVIDIA L40s GPU [S1].

## Background
HAC++ was introduced to address the storage inefficiency of 3D Gaussian Splatting. It builds on Scaffold-GS, which uses unorganized anchors and a structured hash grid [S1]. The training process was developed to stabilize the joint training of geometric and entropy models, which is prone to instability and memory issues in standard implementations [S1]. The method is part of the broader effort to compress 3DGS for efficient storage and transmission [S1].

## Sources
[S1] HAC++: Towards 100X Compression of 3D Gaussian Splatting, https://doi.org/10.1109/tpami.2025.3594066, fetched 2026-10-28
[S2] PCGS: Progressive Compression of 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2503.08511, fetched 2026-10-26
