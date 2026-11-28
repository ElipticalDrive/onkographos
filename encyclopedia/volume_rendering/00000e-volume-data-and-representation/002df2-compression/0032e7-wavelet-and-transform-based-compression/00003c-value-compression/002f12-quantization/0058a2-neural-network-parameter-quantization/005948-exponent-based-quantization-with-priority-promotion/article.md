# Exponent-based Quantization with Priority Promotion

Also called: Exponent-based Quantization

## Overview
Exponent-based Quantization with Priority Promotion is a lossy compression technique for neural network weights that partitions floating-point numbers into buckets based on their exponent and sign, then reduces the number of buckets using priority promotion to limit indexing bits [S1]. It sets itself apart from siblings like [Global Quantization](15134-global-quantization.md) by grouping weights according to their magnitude scale (exponent) rather than global range, allowing for finer granularity in high-magnitude regions while coarsening low-magnitude ones [S1].

## How it works
**Inputs:** A tensor of floating-point weights $W$ [S1].
**Outputs:** Quantized indices and representative values, ready for entropy coding [S1].

**Data Structures:**
*   **Bucket Map:** A dictionary or hash map where keys are tuples $(s, e)$ representing sign and exponent, and values are lists of weights belonging to that bucket [S1, G].
*   **Bucket Representatives:** A map from $(s, e)$ to a single representative value $v_{\text{rep}}$ [S1].
*   **Priority List:** A sorted list of active bucket keys $(s, e)$, ordered by the magnitude of the exponent $e$ (descending) [S1, G].

**Steps:**

1.  **Partitioning:**
    For each weight $w$ in the tensor $W$, decompose it into its sign $s$, mantissa $m$, and exponent $e$ based on the floating-point representation:
    $$ w = (-1)^s \times m \times 2^e $$
    Assign $w$ to a bucket identified by the key $(s, e)$ [S1].

2.  **Bucket Representation:**
    For each non-empty bucket $B_{(s,e)}$, calculate the representative value $v_{\text{rep}}$ as the average of the maximum and minimum values found in that bucket:
    $$ v_{\text{rep}} = \frac{\max(B_{(s,e)}) + \min(B_{(s,e)})}{2} $$
    All weights in $B_{(s,e)}$ are replaced by $v_{\text{rep}}$ during quantization [S1].

3.  **Priority Promotion (Bucket Reduction):**
    Let $N_b$ be the number of bits allocated for bucket indexing [S1]. The maximum number of distinct buckets that can be indexed is $2^{N_b} - 1$ [S1].
    *   Sort all active buckets by their exponent $e$ in descending order (largest exponents first) [S1, G].
    *   Select the top $K = 2^{N_b} - 1$ buckets with the largest exponents. These buckets retain their specific representatives [S1].
    *   Merge all remaining buckets (those with smaller exponents) into a single "overflow" bucket [S1].
    *   The representative value for this overflow bucket is defined as $0$ [S1].
    *   Assign indices $0 \dots K-1$ to the significant buckets and index $K$ to the overflow bucket, ensuring all indices fit within $N_b$ bits [S1].

4.  **Compression:**
    Replace each weight in the tensor with the index of its assigned bucket and the representative value associated with that bucket. Compress the resulting sequence of indices and values using Huffman coding [S1].

**Parameters:**
*   $N_b$: Number of bits for bucket indexing. Typical range used in experiments is 3 to 5 [S1].

**Pitfalls:**
*   Using fewer than $2^2 - 1 = 3$ buckets (i.e., $N_b < 2$) results in substantial distortion because the bucket resolution is too coarse to represent the weight distribution accurately [S1].
*   The choice of $N_b$ directly trades bitrate for distortion. Values below 3 cause significant quality drops, while values above 5 show stable quality with diminishing returns [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from collections import defaultdict

def exponent_based_quantization(weights, N_b):
    """
    Compresses a list of floating-point weights using exponent-based quantization
    with priority promotion.
    
    Args:
        weights: List of floats.
        N_b: Number of bits for bucket indexing.
        
    Returns:
        bucket_reps: Dict mapping (sign, exponent) to representative value.
        key_to_index: Dict mapping (sign, exponent) to bucket index.
        overflow_index: Index assigned to the merged overflow bucket.
        indices: List of bucket indices for each weight.
    """
    # Step 1: Partition weights into buckets based on sign and exponent
    buckets_data = defaultdict(list)
    
    for w in weights:
        if w == 0.0:
            s, m, e = 0, 0.0, 0
        else:
            s = 0 if w > 0 else 1
            m, e = math.frexp(abs(w))
        buckets_data[(s, e)].append(w)
        
    # Step 2: Calculate representative values for each bucket
    bucket_reps = {}
    for key, vals in buckets_data.items():
        bucket_reps[key] = (max(vals) + min(vals)) / 2.0
        
    # Step 3: Priority Promotion
    active_keys = list(bucket_reps.keys())
    # Sort by exponent descending
    active_keys.sort(key=lambda k: k[1], reverse=True)
    
    max_significant = (1 << N_b) - 1
    significant_keys = active_keys[:max_significant]
    # Remaining keys are merged into the overflow bucket
    
    key_to_index = {key: i for i, key in enumerate(significant_keys)}
    overflow_index = max_significant  # Index for the overflow bucket
    
    # Step 4: Generate indices for each weight
    indices = []
    for w in weights:
        if w == 0.0:
            s, m, e = 0, 0.0, 0
        else:
            s = 0 if w > 0 else 1
            m, e = math.frexp(abs(w))
        key = (s, e)
        indices.append(key_to_index.get(key, overflow_index))
        
    return bucket_reps, key_to_index, overflow_index, indices
```

## Operating characteristics
*   **Performance:** Tested on NeRF architectures, rendered quality remains stable when $N_b > 5$ and drops significantly when $N_b < 3$ [S1].
*   **Strengths:** Adapts quantization granularity to the magnitude of weights, preserving important high-magnitude parameters while aggressively compressing low-magnitude ones [S1].
*   **Limitations:** Lossy compression that requires careful selection of $N_b$ to balance bitrate and distortion [S1].
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.

## Background
This method was proposed in the context of volumetric video compression through neural-based representations, specifically for compressing NeRF (Neural Radiance Field) models [S1]. It builds on the observation that not all neural network parameters are crucial for accurate rendering, allowing for significant size reduction with limited performance impact [S1].

## Sources
[S1] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
