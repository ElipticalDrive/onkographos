# Inference-Time Clustering

Also called: K-Means Token Merging, Token Clustering, Spherical K-Means Clustering, complexity-weighted initialization clustering, k-means on geometry tokens, k-means token clustering, strided initialization clustering

## Overview
Inference-Time Clustering is a token compression technique that groups geometry tokens into clusters via k-means and merges each cluster into a single scene token, reducing the Gaussian budget while preserving cross-view consistency [S1, unofficial]. It allows adjustable compression at inference without retraining, distinguishing it from methods that require fixed budgets or retraining for different compression levels [S1, unofficial]. Unlike nearest-seed assignment, it uses iterative refinement to improve quality, though the gain is marginal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of geometry tokens $G = \{g_i\}$ from $N$ views, with $T$ patches per view. Each token $g_i$ is a vector representing the geometric properties of a patch.
- **Output:** A set of $K$ scene tokens (centroids) $Z = \{z_k\}$ and an assignment map $A: \{1, \dots, N \times T\} \to \{1, \dots, K\}$, where each geometry token is assigned to one scene token. The scene tokens are used to generate the final Gaussian splats.

**Data Structures:**
- **Geometry Tokens:** A list of vectors. The source specifies using $\ell_2$-normalized geometry tokens for the clustering space [S1, unofficial].
- **Scene Tokens (Centroids):** A list of $K$ vectors, initialized as the mean of their cluster members.
- **Assignments:** A discrete mapping from each geometry token index to a cluster index.
- **Supporting Observations:** Each scene token retains the list of geometry tokens (patches) assigned to it to compute the cone loss [S1, unofficial].

**Parameters:**
- **Compression Ratio ($r$):** Determines the target number of clusters $K$. Defined as $r = \frac{K}{N \times T}$, where $N$ is the number of views and $T$ is the number of patches per view. $r \in (0, 1]$ [S1, unofficial].
    - Typical values: $r=1.0$ (no compression), $r=0.5$, $r=0.25$, $r=0.1$, $r=0.05$ [S1, unofficial].
    - During training, $r$ is sampled uniformly from $[r_{min}, 1]$, where $r_{min} = 0.2/N$, ensuring a minimum budget of approximately 2K Gaussians (or at least ~65 scene tokens) [S1, unofficial].
- **Lloyd Iterations:** Capped at 50. Typically converges in 2–3 iterations at $r=0.5$ and 7–9 iterations at $r=0.1$ [S1, unofficial].
- **Initialization Scheme:**
    - **Complexity-Weighted:** Seeds are sampled without replacement with probability proportional to $(0.05 + c_i)^2$, where $c_i \in [0, 1]$ is the per-scene rank of patch $i$ by mean image-gradient energy. This biases initialization toward textured/complex regions [S1, unofficial].
    - **Strided Grid:** An alternative initialization method (less effective) [S1, unofficial].

**Algorithm Steps:**
1.  **Normalize Tokens:** Compute $\hat{g}_i = \frac{g_i}{\|g_i\|_2}$ for all geometry tokens [S1, unofficial].
2.  **Initialize Centroids:**
    - Compute complexity scores $c_i$ for each patch based on image-gradient energy.
    - Sample $K$ unique indices as initial seeds $S$ without replacement, with probability proportional to $(0.05 + c_i)^2$ [S1, unofficial].
    - Initialize centroids $z_k^{(0)}$ for each seed $k$. The source states: "Each scene token is initialized as the mean of its cluster members... and retains them as supporting observations." [S1, unofficial]. *Note: Since seeds are single points initially, the first centroid is likely the seed token itself or the mean of the single seed. The text implies the mean is recomputed after assignment.*
3.  **Iterate Lloyd's Algorithm (Spherical K-Means):**
    - **Assignment Step:** For each normalized geometry token $\hat{g}_i$, assign it to the cluster $k$ with the closest centroid $z_k$ using cosine distance (equivalent to maximizing cosine similarity).
        $$ k_i = \arg\max_k \frac{\hat{g}_i \cdot z_k}{\|\hat{g}_i\|_2 \|z_k\|_2} = \arg\max_k \hat{g}_i \cdot z_k $$
        (Since $\hat{g}_i$ is normalized, this simplifies to dot product with normalized centroids).
    - **Update Step:** For each cluster $k$, compute the new centroid as the mean of its assigned geometry tokens:
        $$ z_k^{new} = \frac{1}{|C_k|} \sum_{i \in C_k} g_i $$
        where $C_k$ is the set of indices assigned to cluster $k$.
    - **Renormalize:** Renormalize the new centroid: $z_k^{new} \leftarrow \frac{z_k^{new}}{\|z_k^{new}\|_2}$ [S1, unofficial].
    - **Convergence Check:** Repeat until the maximum number of iterations (50) is reached or assignments do not change. Convergence is typically fast (2–9 iterations) [S1, unofficial].
4.  **Merge Tokens:** For each cluster $k$, create a single scene token $z_k$ (the final centroid). The scene token inherits all patches in $C_k$ as supporting observations [S1, unofficial].
    - *Pitfall:* Skipping the merge step and keeping only the seed tokens degrades quality significantly (loss of 1.8 dB at $r=0.25$ and 5.7 dB at $r=0.1$) [S1, unofficial].

**Formulas:**
- **Compression Ratio:**
    $$ r = \frac{K}{N \times T} $$
    where $K$ is the number of clusters, $N$ is the number of views, and $T$ is the number of patches per view [S1, unofficial].
- **Cone Loss for Merged Token:**
    For a merged scene token $k$ with supporting patches $C_k$, the cone loss is:
    $$ \mathcal{L}_{cone} = \min_{i \in C_k} \|\pi_i(\mu_{kj}) - u_i\| $$
    This ensures each Gaussian falls into the full capacity (union of member cones) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def inference_time_clustering(geometry_tokens, num_views, patches_per_view, compression_ratio):
    """
    Performs Inference-Time Clustering on geometry tokens.
    
    Args:
        geometry_tokens: np.array of shape (N_views * T, D), raw geometry vectors.
        num_views: int, N.
        patches_per_view: int, T.
        compression_ratio: float, r.
        
    Returns:
        scene_tokens: np.array of shape (K, D), normalized centroids.
        assignments: np.array of shape (N_views * T,), cluster indices.
        supporting_observations: list of lists, supporting_observations[k] contains indices of tokens in cluster k.
    """
    N = num_views
    T = patches_per_view
    total_tokens = N * T
    
    # Calculate number of clusters K
    K = int(round(compression_ratio * total_tokens))
    if K < 1:
        K = 1
        
    # Normalize tokens for spherical k-means
    norms = np.linalg.norm(geometry_tokens, axis=1, keepdims=True)
    # Avoid division by zero
    norms[norms == 0] = 1e-8
    normalized_tokens = geometry_tokens / norms
    
    # Initialize complexity weights for seed selection
    # Note: The source mentions 'mean image-gradient energy' as c_i.
    # In a pure implementation without access to gradients, this might be approximated or assumed pre-computed.
    # Here we assume complexity_scores is provided or computed externally.
    # If not available, random initialization is a fallback, but complexity-weighted is preferred.
    # For the sake of self-contained pseudocode, we simulate random complexity scores if not provided.
    # In practice, c_i should be derived from the input data's gradient energy.
    complexity_scores = np.random.rand(total_tokens) # Placeholder for actual gradient energy ranking
    # Normalize to [0, 1]
    complexity_scores = (complexity_scores - complexity_scores.min()) / (complexity_scores.max() - complexity_scores.min() + 1e-8)
    
    # Sample seeds without replacement with probability proportional to (0.05 + c_i)^2
    weights = (0.05 + complexity_scores) ** 2
    weights /= weights.sum()
    
    # Use numpy's choice for sampling without replacement
    # Note: numpy.random.choice with replace=False uses a specific algorithm; 
    # for large populations and small K, this is efficient.
    seed_indices = np.random.choice(total_tokens, size=K, replace=False, p=weights)
    
    # Initialize centroids as the mean of their initial members (initially just the seed itself)
    # The source says: "Each scene token is initialized as the mean of its cluster members... 
    # and retains them as supporting observations."
    # Initially, each cluster has 1 member (the seed).
    centroids = normalized_tokens[seed_indices].copy()
    
    # Supporting observations: list of lists
    supporting_observations = [[idx] for idx in seed_indices]
    
    # Lloyd iterations
    max_iters = 50
    assignments = np.zeros(total_tokens, dtype=int)
    
    for iteration in range(max_iters):
        # Assignment Step: Cosine similarity (dot product for normalized vectors)
        # Similarity matrix: (total_tokens, K)
        similarities = np.dot(normalized_tokens, centroids.T)
        
        # Assign each token to the closest centroid
        new_assignments = np.argmax(similarities, axis=1)
        
        # Check for convergence
        if np.array_equal(new_assignments, assignments) and iteration > 0:
            break
        assignments = new_assignments
        
        # Update Step: Recompute centroids as mean of assigned tokens
        new_centroids = np.zeros_like(centroids)
        new_supporting = [[] for _ in range(K)]
        
        for k in range(K):
            mask = (assignments == k)
            members = np.where(mask)[0]
            new_supporting[k] = members.tolist()
            
            if len(members) > 0:
                # Mean of raw geometry tokens (not normalized) for the centroid direction?
                # The source says: "z0k = |Ck |−1 \sum_{i \in C_k} g_i"
                # And then "centroid renormalization after each update".
                # So we average the raw vectors, then normalize.
                mean_vec = np.mean(geometry_tokens[members], axis=0)
                norm = np.linalg.norm(mean_vec)
                if norm > 0:
                    new_centroids[k] = mean_vec / norm
                else:
                    new_centroids[k] = centroids[k] # Keep old if zero vector
            else:
                new_centroids[k] = centroids[k] # Keep old if empty cluster
        
        centroids = new_centroids
        supporting_observations = new_supporting
        
    return centroids, assignments, supporting_observations
```

**Edge Cases and Pitfalls:**
- **High Compression:** At very high compression ratios (e.g., $r=0.05$, 20× compression), thin structures and grass may not be fully recovered, requiring over 8K Gaussians even with refinement [S1, unofficial].
- **Empty Clusters:** If a cluster becomes empty during updates, the centroid should be retained or re-initialized to avoid collapse [G].
- **Zero Norm Tokens:** Tokens with zero norm should be handled to avoid division by zero during normalization [G].
- **Initialization Sensitivity:** Complexity-weighted initialization provides better quality (0.3 dB gain at $r=0.25$) compared to strided initialization [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The k-means cost scales with $Tokens \times Clusters \times Iterations$. At 192 views with $r=0.5$, k-means iterations add 4.2 seconds over full capacity, making it the dominant overhead at high view counts [S1, unofficial]. Convergence typically takes 17–45 iterations for large budgets, but fewer (2–9) for typical compression ratios [S1, unofficial].
- **Memory Complexity:** Reduces the number of Gaussian parameters stored by a factor of $1/r$. The scene tokens and assignments are significantly smaller than the full set of geometry tokens [S1, unofficial].
- **Performance:** At $r=0.5$, storage is halved and rendering speed nearly doubles compared to full capacity [S1, unofficial].
- **Strengths:** Adjustable compression at inference without retraining; preserves cross-view consistency (91% of clusters span at least two views at $r=0.1$ and 6 views) [S1, unofficial]. Geometry tokens are the best clustering space, outperforming backbone tokens, patch colors, and even ground-truth 3D points [S1, unofficial].
- **Limitations:** High compression (20×) fails to recover fine details like grass [S1, unofficial]. K-means overhead can be significant at very high view counts [S1, unofficial].

## Background
Inference-Time Clustering is part of the ZipSplat method, which aims to compress Gaussian Splatting parameters. It was introduced in the paper "ZipSplat: Fewer Gaussians, Better Splats" [S1, unofficial]. The method leverages the redundancy of geometry tokens across multiple views to merge them into a smaller set of scene tokens. It builds on the idea of token merging in vision transformers but applies it to 3D Gaussian Splatting parameters. The use of spherical k-means and complexity-weighted initialization is specific to this application to preserve fine details and cross-view consistency.

## Sources
[S1] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
