# Exponent-based Quantization with Priority Promotion

Also called: Exponent-based Quantization

## Overview
Exponent-based Quantization with Priority Promotion is a lossy compression technique for neural network weights that partitions floating-point numbers into buckets based on their exponent and sign, then reduces the number of buckets using priority promotion to limit indexing bits [S1]. It sets itself apart from siblings like [Global Quantization](15134-global-quantization.md) by grouping weights according to their magnitude scale (exponent) rather than global range, allowing for finer granularity in high-magnitude regions while coarsening low-magnitude ones [S1].

## How it works
**Inputs:** A tensor of floating-point weights $W$.
**Outputs:** A compressed bitstream of quantized indices and values, ready for entropy coding.

**Data Structures:**
*   **Bucket Map:** A dictionary or hash map where keys are tuples $(s, e)$ representing sign and exponent, and values are lists of weights belonging to that bucket.
*   **Bucket Representatives:** A map from $(s, e)$ to a single representative value $v_{rep}$.
*   **Priority List:** A sorted list of active bucket keys $(s, e)$, ordered by the magnitude of the exponent $e$ (descending).

**Steps:**

1.  **Partitioning:**
    For each weight $w$ in the tensor $W$, decompose it into its sign $s$, mantissa $m$, and exponent $e$ based on the floating-point representation:
    $$ w = (-1)^s \times m \times 2^e $$
    Assign $w$ to a bucket identified by the key $(s, e)$ [S1].

2.  **Bucket Representation:**
    For each non-empty bucket $B_{(s,e)}$, calculate the representative value $v_{rep}$ as the average of the maximum and minimum values found in that bucket:
    $$ v_{rep} = \frac{\max(B_{(s,e)}) + \min(B_{(s,e)})}{2} $$
    All weights in $B_{(s,e)}$ are replaced by $v_{rep}$ during quantization [S1].

3.  **Priority Promotion (Bucket Reduction):**
    Let $N_b$ be the number of bits allocated for bucket indexing. The maximum number of distinct buckets that can be indexed is $2^{N_b} - 1$ [S1].
    *   Sort all active buckets by their exponent $e$ in descending order (largest exponents first).
    *   Select the top $K = 2^{N_b} - 1$ buckets with the largest exponents. These buckets retain their specific representatives.
    *   Merge all remaining buckets (those with smaller exponents) into a single "overflow" bucket.
    *   The representative value for this overflow bucket is defined as $0$ [S1].
    *   This ensures that only $N_b$ bits are required to index any bucket [S1].

4.  **Compression:**
    Replace each weight in the tensor with the index of its assigned bucket (using $N_b$ bits) and the representative value associated with that bucket. Compress the resulting sequence of indices and values using Huffman coding [S1].

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
        buckets: Dict mapping (sign, exponent) to representative value.
        indices: List of bucket indices for each weight.
    """
    # Step 1: Partition weights into buckets based on sign and exponent
    # Bucket key: (sign, exponent)
    # Bucket value: list of weights
    buckets_data = defaultdict(list)
    
    for w in weights:
        if w == 0:
            # Special case for zero, typically exponent is handled as a specific bucket or merged
            # Based on [S1], 0 is likely merged into the overflow bucket or handled by sign/exp
            # Assuming standard float decomposition:
            sign = 0 if w >= 0 else 1
            # Note: Exact float decomposition logic depends on hardware/standard.
            # For the purpose of this algorithm, we assume access to s and e.
            # In Python, math.frexp returns (m, e) where m is in [0.5, 1).
            # Sign is separate.
            m, e = math.frexp(abs(w))
            # Adjust e if w is 0
            if w == 0:
                e = 0 
                m = 0
            
            # Map to integer exponent for bucketing if necessary, 
            # but [S1] implies using the raw exponent value for sorting.
            # We store the exponent as is.
            key = (sign, e)
            buckets_data[key].append(w)
            
    # Step 2: Calculate representative values for each bucket
    bucket_reps = {}
    for key, vals in buckets_data.items():
        if not vals:
            continue
        max_val = max(vals)
        min_val = min(vals)
        bucket_reps[key] = (max_val + min_val) / 2.0
        
    # Step 3: Priority Promotion
    # Get all active keys
    active_keys = list(bucket_reps.keys())
    
    # Sort by exponent (descending)
    # Key is (sign, exponent). We sort by exponent [1]
    active_keys.sort(key=lambda k: k[1], reverse=True)
    
    # Max number of buckets we can index
    max_buckets = (1 << N_b) - 1
    
    # Select top buckets
    selected_keys = active_keys[:max_buckets]
    remaining_keys = active_keys[max_buckets:]
    
    # Create a mapping from key to index
    key_to_index = {}
    for i, key in enumerate(selected_keys):
        key_to_index[key] = i
        
    # The remaining keys are merged into a single bucket represented by 0
    # We assign them a special index, e.g., max_buckets (if we had N_b+1 bits)
    # But [S1] says "only N_b bits are required". 
    # This implies the overflow bucket is one of the 2^N_b - 1 slots?
    # No, [S1] says "keep 2^N_b - 1 buckets... The rest buckets are merged into one bucket, 
    # which is represented by 0."
    # It does not explicitly say the overflow bucket gets an index. 
    # However, to index it, we need an index. 
    # If we have 2^N_b - 1 slots for "significant" buckets, we have 1 slot left 
    # if we use N_b bits for 2^N_b values? 
    # Or does the overflow bucket not need an index because it's implicit?
    # [S1] says "only N_b bits are required to index buckets".
    # If we have 2^N_b - 1 significant buckets, we can index them with N_b bits 
    # if we reserve one index for the overflow? 
    # 2^N_b total indices. 2^N_b - 1 used for significant. 1 used for overflow.
    # This fits perfectly.
    
    overflow_index = (1 << N_b) - 1 # The last index
    
    # Assign indices
    indices = []
    for w in weights:
        if w == 0:
             # Handle zero explicitly if needed, or let it fall into a bucket
             # Assuming 0 is in a bucket with exp 0 or similar
             pass
             
        # Re-decompose to find key
        sign = 0 if w >= 0 else 1
        m, e = math.frexp(abs(w))
        if w == 0: e = 0
        
        key = (sign, e)
        
        if key in key_to_index:
            idx = key_to_index[key]
        else:
            idx = overflow_index
            
        indices.append(idx)
        
    # The representative for the overflow bucket is 0
    # The representatives for selected keys are in bucket_reps
    
    return bucket_reps, key_to_index, overflow_index, indices
```

## Operating characteristics
*   **Time Complexity:** Partitioning is $O(N)$ where $N$ is the number of weights. Sorting buckets is $O(B \log B)$ where $B$ is the number of unique exponent/sign combinations. In practice, $B$ is small relative to $N$.
*   **Memory Complexity:** Requires storing the bucket representatives and the mapping. The output size is $N \times N_b$ bits for indices plus the storage for representatives.
*   **Performance:** Tested on NeRF architectures. Quality remains stable for $N_b > 5$ and drops significantly for $N_b < 3$ [S1].
*   **Strengths:** Adapts quantization granularity to the magnitude of weights, preserving important high-magnitude parameters while aggressively compressing low-magnitude ones.
*   **Limitations:** Lossy. Requires careful selection of $N_b$ to balance bitrate and distortion.

## Background
This method was proposed in the context of volumetric video compression through neural-based representations, specifically for compressing NeRF (Neural Radiance Field) models [S1]. It builds on the observation that not all neural network parameters are crucial for accurate rendering, allowing for significant size reduction with limited performance impact [S1].

## Sources
[S1] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
