# Radix-k Compositing

Also called: Radix-k

## Overview
Radix-k Compositing is a distributed memory compositing strategy that combines the flexibility of direct send with the performance of binary swap [S1]. It solves the limitation of binary-swap methods, which require the number of processors to be a power of two, by allowing any number of processors $N$ to participate in the compositing phase [S1]. It sets itself apart from its parent [2-3 Swap Compositing](12223-2-3-swap-compositing.md) by generalizing the group size per round, whereas 2-3 swap is restricted to groups of two or three.

## How it works
**Inputs:**
- $N$: Total number of processors involved in compositing.
- $P$: Set of processors, indexed $0 \dots N-1$.
- Image data: Each processor holds a region of the current image (partial image) to be composited.

**Outputs:**
- A fully composited image distributed across the processors (or gathered to a specific node, depending on the higher-level scheme).

**Data Structures:**
- The image is partitioned among processors. Each processor owns a specific region of the final image.
- No specific bit-packing is defined in the sources; standard image buffers are assumed.

**Steps:**
1. **Factorization:** Determine a sequence of group sizes $k_1, k_2, \dots, k_m$ such that:
   $$ \prod_{i=1}^{m} k_i = N $$
   The factors $k_i$ are typically small integers (e.g., 2s and 3s) to balance communication overhead and parallelism [S1].
2. **Round Execution:** For each round $i$ from $1$ to $m$:
   a. Divide the $N$ processors into groups of size $k_i$.
   b. Within each group, processors exchange their current image regions using a **direct send** method [S1].
   c. Each processor updates its local image data by compositing the received data with its own [S1].
3. **Completion:** After $m$ rounds, the compositing is complete.

**Parameters:**
- $k_i$: Group size for round $i$. The choice of factors affects the number of communication rounds ($m$) and the volume of data sent per round.
- $N$: Number of processors. Must be a composite number or prime (if $k_1=N$).

**Edge Cases and Pitfalls:**
- If $N$ is prime, the only factorization is $k_1 = N$, resulting in a single round of all-to-all communication. This may be inefficient for large $N$ compared to binary swap [S1].
- Load imbalance can occur if image regions are not evenly distributed or if some processors have more complex geometry to composite [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radix_k_compositing(N, my_id, my_image, all_images):
    """
    Perform Radix-k compositing.
    N: Total number of processors.
    my_id: ID of this processor (0 to N-1).
    my_image: The partial image held by this processor.
    all_images: A list of images held by all processors (for simulation/distributed context).
    """
    # Step 1: Factorize N into group sizes k_i
    # Note: The sources do not specify the exact factorization algorithm.
    # A simple greedy approach using small primes is implied.
    factors = []
    n_temp = N
    for k in [2, 3, 5, 7]:  # Small primes
        while n_temp % k == 0:
            factors.append(k)
            n_temp //= k
    if n_temp > 1:
        factors.append(n_temp)  # Remaining prime factor

    # Step 2: Execute rounds
    current_images = list(all_images) # Copy of all images for simulation
    current_images[my_id] = my_image

    for k in factors:
        # Divide processors into groups of size k
        # Group index for this processor
        group_id = my_id // k
        # Start index of this group
        group_start = group_id * k
        # End index of this group
        group_end = group_start + k

        # Collect images from all processors in this group
        group_images = []
        for p in range(group_start, group_end):
            group_images.append(current_images[p])

        # Composite all images in the group
        # The order of compositing matters for opacity.
        # Assuming standard front-to-back compositing.
        # The sources state "each processor owning a region of the current image"
        # exchanges data. This implies all processors in the group update their
        # image with the combined result of the group.
        
        combined_image = group_images[0]
        for img in group_images[1:]:
            combined_image = composite_front_to_back(combined_image, img)

        # Update local image
        current_images[my_id] = combined_image

    return current_images[my_id]

def composite_front_to_back(img1, img2):
    """
    Standard front-to-back compositing.
    Returns a new image.
    """
    # Implementation details of compositing are assumed to be standard.
    # This is a placeholder for the actual compositing math.
    pass
```

## Subtopics

### Binary-Tree Compositing
Binary-Tree Compositing is a simpler compositing strategy initially implemented as a basic binary-tree step [S2, unofficial]. It combines partial images from processing nodes in a hierarchical manner [S2, unofficial]. It sets itself apart from Radix-k by using a fixed tree structure rather than configurable group sizes per round [S2, unofficial]. It may be replaced later with more elaborate algorithms like binary-swap or Radix-k for better performance [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The number of communication rounds is equal to the number of factors in the factorization of $N$. For $N=2^p$, this is $p$ rounds (similar to binary swap). For $N$ with many small factors, it remains efficient.
- **Memory Complexity:** Each processor stores the full image at each step, as it receives data from all other processors in its group.
- **Performance:** Combines the flexibility of direct send (any $N$) with the performance of binary swap (logarithmic rounds for composite $N$) [S1].
- **Strengths:** Scalable to any number of processors; reduces load imbalance compared to pure object-order or image-order approaches when used in a hybrid scheme [S1].
- **Limitations:** Performance depends on the factorization of $N$; if $N$ is prime, it degrades to all-to-all communication [S1].

## Background
Radix-k Compositing was proposed by Peterka et al. [S1]. It was developed to address the limitations of binary-swap compositing, which requires the number of processors to be a power of two [S1]. The method is part of a hybrid distributed-memory parallel volume rendering scheme that combines object-order and image-order approaches to reduce load imbalance [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S2] Volume Visualization of AMR Data, https://graphics.cs.ucdavis.edu/~okreylos/Research/AMRVis/MainPage.html, fetched 2026-10-26 (unofficial)
