# Sparse Coding with K-SVD and Coresets

Also called: Coreset Subsampling, K-SVD Dictionary Training, K-SVD with Coresets, Sparse Coding Global Processing, Sparse Coding with Coresets

## Overview
This method addresses the prohibitive computational cost of training data-specific dictionaries on massive volume datasets by combining the K-SVD algorithm with coreset-based data reduction [S1]. Unlike uniform subsampling, which fails to capture critical information at material boundaries, this approach uses importance sampling to construct a small, weighted subset of training blocks that approximates the original data distribution [S1].

## How it works
**Inputs:**
- A set of $N$ training blocks extracted from the volume data [S1].
- Target coreset size $C$ (where $C \ll N$) [S1].
- Dictionary size $K$ for the K-SVD algorithm [S1].

**Outputs:**
- A trained dictionary matrix $D \in \mathbb{R}^{m \times K}$, where $m$ is the dimensionality of a block [S1].
- Sparse coefficients for the training blocks (implicit in the training process) [G].

**Data Structures:**
- **Training Blocks:** A matrix of shape $(N, m)$ [G].
- **Importance Scores:** An array $\iota$ of length $N$, where $\iota_i$ is the importance of block $i$ [S1].
- **Coreset:** A subset of indices $I \subset \{1, \dots, N\}$ with $|I|=C$, and associated weights $w$ [S1].

**Algorithm Steps:**

1. **Compute Importance Scores:**
   Calculate the importance $\iota_i$ for each training block $i$. The sources specify that $\iota_i$ is proportional to the standard deviation of the block's values [S1].
   $$ \iota_i \propto \sigma_i $$
   where $\sigma_i$ is the standard deviation of the pixel/voxel values in block $i$ [S1].

2. **Normalize Probabilities:**
   Convert importance scores into picking probabilities $p_j$ for each block $j$ [S1].
   $$ p_j = \frac{\iota_j}{\sum_{k=1}^{N} \iota_k} $$

3. **Construct Coreset via Importance Sampling:**
   Select $C$ blocks from the original $N$ blocks using a streaming method [S1]. Each block $j$ is picked with probability $p_j$ [S1]. Let the selected indices be $j_1, j_2, \dots, j_C$ [S1].

4. **Assign Weights:**
   For each selected block $j$ in the coreset, assign a weight $w_j$ to remove bias introduced by the non-uniform sampling [S1]. The weight is calculated as:
   $$ w_j = \frac{1}{\sqrt{p_j}} $$
   where $p_j$ is the probability with which block $j$ was selected [S1].

5. **Train Dictionary with K-SVD:**
   Apply the K-SVD learning algorithm to the resampled and re-weighted set of $C$ blocks [S1]. The algorithm minimizes reconstruction error on this weighted subset [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def train_sparse_coding_coreset(blocks, target_coreset_size, dict_size):
    """
    blocks: numpy array of shape (N, m), N training blocks of dimension m.
    target_coreset_size: integer, number of blocks to select for the coreset.
    dict_size: integer, number of atoms in the dictionary.
    """
    N, m = blocks.shape
    
    # Step 1: Compute importance scores (proportional to standard deviation)
    std_devs = np.std(blocks, axis=1)
    importance = std_devs
    
    # Step 2: Normalize to get probabilities
    total_importance = np.sum(importance)
    probabilities = importance / total_importance
    
    # Step 3: Select C blocks using importance sampling
    selected_indices = np.random.choice(N, size=target_coreset_size, p=probabilities, replace=True)
    
    # Step 4: Calculate weights to remove bias
    selected_probabilities = probabilities[selected_indices]
    weights = 1.0 / np.sqrt(selected_probabilities)
    
    # Extract the coreset blocks
    coreset_blocks = blocks[selected_indices]
    
    # Step 5: Train dictionary using K-SVD on the weighted coreset
    # The sources do not specify the internal K-SVD update rules [G].
    # Standard weighted K-SVD training on coreset_blocks with weights [G]
    dictionary = None  # Standard K-SVD solver implementation [G]
    
    return dictionary
```

**Edge Cases and Pitfalls:**
- **Uniform Subsampling Failure:** Uniform sub-sampling is explicitly rejected because material boundaries carry more information than empty or uniform space [S1].
- **Numerical Instability:** Massive datasets can lead to numerical instabilities during K-SVD calculations [S1].
- **Computational Cost:** Direct K-SVD on massive volumes is prohibitively expensive in time and memory [S1].

## Operating characteristics
- **Time Complexity:** The dominant cost is the K-SVD training on the coreset of size $C$ [S1]. Since $C \ll N$, this is significantly faster than training on $N$ blocks [S1]. The sampling step is $O(N)$ for computing standard deviations and $O(C)$ for selection [G].
- **Memory Complexity:** Reduced from $O(N \cdot m)$ to $O(C \cdot m)$ for the training data held in memory during K-SVD [G].
- **Performance:** Results demonstrate that extremely good approximations can be obtained using a tiny fraction of the original data [S1].
- **Strengths:** Scalability to massive volumes; captures critical features (boundaries) better than uniform sampling [S1].
- **Limitations:** Requires careful tuning of $C$; relies on the assumption that standard deviation is a good proxy for information content [S1].

## Background
- **Origin:** The method was introduced by Gobbetti et al. [GIM12] for volume rendering compression, applying the concept of coresets [AHPV05, CS07] to K-SVD dictionary training [S1].
- **Development:** Prior approaches for BTF compression by Ruiters and Klein [RK09] used truncated SVD for data reduction [S1]. Gobbetti et al. improved upon this by using importance sampling based on standard deviation [S1].
- **Context:** It is part of the global processing techniques for sparse coding, aiming to improve upon vector quantization by using data-specific dictionaries [S1].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
