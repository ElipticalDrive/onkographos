# Incremental Subrange Integration


## Overview
Incremental Subrange Integration is an algorithm for computing the exact $n \times n$ pre-integration lookup table in $O(n^2)$ time, avoiding the $O(n^3)$ cost of brute-force integration [S1]. It solves the problem that standard incremental composition fails because the non-linear opacity correction depends on the number of composited intervals $|j-i|$, which varies across the lookup table [S1]. Unlike brute-force methods, it reuses previously computed subrange integrals by organizing them along diagonals where $|j-i|$ is constant [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A transfer function with $n_t$ entries (sample values).
*   **Output:** An $n \times n$ pre-integration lookup table, where $n$ is typically equal to or a small constant factor of $n_t$ [S1].

**Data Structures:**
*   **Lookup Table:** An $n \times n$ matrix storing pre-integrated color and opacity values for all pairs of transfer function endpoints $(i, j)$ [S1].
*   **Subrange-Integral Table:** For each diagonal of length $n_r$ (where $n_r = |j-i|$), a table is constructed to store intermediate integration results [S1].
    *   The transfer function (size $n_t$) is separated into bins of length $n_r$ [S1].
    *   Each bin stores $2n_r$ integration values: computed incrementally from left-to-right and right-to-left [S1].
    *   Total entries per sub-integral table: $O(n_t)$ [S1].

**Formulas:**
The core challenge is the non-linear opacity correction required when compositing intervals of different lengths. The opacity value from the transfer function ($\alpha_{\text{from xfer func}}$) represents opacity per unit in world space. To composite it correctly over a range of $|j-i|$ intervals with world-space sampling slab width $\Delta$, the opacity must be corrected before compositing [S1]:

$$
\alpha_{\text{for compositing}} = 1 - (1 - \alpha_{\text{from xfer func}})^{\frac{\Delta}{|j - i|}}
$$

This correction must be applied to the $\alpha$ values before compositing; it cannot be applied to a composited sequence of color values [S1].

**Steps:**
1.  **Diagonal Processing:** Iterate through all $2n$ diagonals of the $n \times n$ lookup table. Each diagonal corresponds to a fixed difference $n_r = |j - i|$ [S1].
2.  **Bin Construction:** For a diagonal of length $n_r$, divide the transfer function into bins of length $n_r$ [S1].
3.  **Subrange Computation:** For each bin, compute subrange integrals incrementally:
    *   **Left-hand subrange:** Start with a subrange $i \rightarrow x$. Expand it to $i \rightarrow (x + 1)$ by compositing a new value behind it (using the corrected opacity) [S1].
    *   **Right-hand subrange:** Start with a subrange $x \rightarrow j$. Expand it to $(x - 1) \rightarrow j$ by compositing one value in the front (using the corrected opacity) [S1].
    *   Store the $2n_r$ intermediate results in the sub-integral table for this bin [S1].
4.  **Bin Composition:** Composite two subrange-integral pairs from adjacent bins to construct the final integral of length $n_r$ for the diagonal [S1].
5.  **Reverse Direction:** For entries where $i > j$ (below the main diagonal), reverse the view direction. Compute integrals and composite operations in the opposite direction (back-to-front) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_preintegration_table(transfer_function, n, delta):
    """
    Computes the n x n pre-integration lookup table using incremental subrange integration.
    
    Args:
        transfer_function: List of (color, opacity) tuples, size n_t.
        n: Dimension of the lookup table.
        delta: World-space sampling slab width.
        
    Returns:
        lookup_table: n x n matrix of (color, opacity) tuples.
    """
    n_t = len(transfer_function)
    lookup_table = [[None for _ in range(n)] for _ in range(n)]
    
    # Iterate over all diagonals defined by the difference dr = j - i
    # dr ranges from -(n-1) to (n-1)
    for dr in range(-(n - 1), n):
        # Determine the length of the integration range for this diagonal
        # The number of intervals is |j - i| = |dr|
        num_intervals = abs(dr)
        
        # Skip if num_intervals is 0 (diagonal elements are trivial or handled separately)
        # In standard pre-integration, diagonal entries might be identity or single-sample
        if num_intervals == 0:
            for i in range(n):
                j = i
                lookup_table[i][j] = transfer_function[i] # Or identity, depending on convention
            continue
            
        # Create subrange-integral table for this diagonal
        # Bins of length num_intervals
        num_bins = (n_t + num_intervals - 1) // num_intervals # Ceiling division
        # Each bin stores 2 * num_intervals values
        # We can use a flat list or dictionary keyed by bin_index and direction
        subrange_tables = []
        
        for b in range(num_bins):
            bin_start = b * num_intervals
            bin_end = min(bin_start + num_intervals, n_t)
            bin_size = bin_end - bin_start
            
            # Initialize storage for left-to-right and right-to-left integrals
            # We need to store results for subranges within this bin
            # Left-to-right: integrals starting at bin_start, ending at bin_start + k
            # Right-to-left: integrals ending at bin_end - 1, starting at bin_end - 1 - k
            
            left_integrals = []
            right_integrals = []
            
            # Compute left-to-right integrals for the bin
            # Start with the first sample in the bin
            current_color, current_alpha = transfer_function[bin_start]
            # Apply opacity correction for 1 interval
            alpha_corrected = 1 - (1 - current_alpha) ** (delta / 1.0)
            left_integrals.append((current_color, alpha_corrected))
            
            for k in range(1, bin_size):
                # Expand to the right by compositing the next sample
                next_color, next_alpha = transfer_function[bin_start + k]
                # Correct opacity for 1 interval
                next_alpha_corrected = 1 - (1 - next_alpha) ** (delta / 1.0)
                # Composite: new = old + (1 - old_alpha) * next
                old_color, old_alpha = left_integrals[-1]
                new_alpha = old_alpha + (1 - old_alpha) * next_alpha_corrected
                new_color = old_color + (1 - old_alpha) * next_color * next_alpha_corrected
                left_integrals.append((new_color, new_alpha))
                
            # Compute right-to-left integrals for the bin
            # Start with the last sample in the bin
            current_color, current_alpha = transfer_function[bin_end - 1]
            alpha_corrected = 1 - (1 - current_alpha) ** (delta / 1.0)
            right_integrals.append((current_color, alpha_corrected))
            
            for k in range(1, bin_size):
                # Expand to the left by compositing the previous sample
                prev_color, prev_alpha = transfer_function[bin_end - 1 - k]
                prev_alpha_corrected = 1 - (1 - prev_alpha) ** (delta / 1.0)
                # Composite: new = prev + (1 - prev_alpha) * old
                old_color, old_alpha = right_integrals[-1]
                new_alpha = prev_alpha_corrected + (1 - prev_alpha_corrected) * old_alpha
                new_color = prev_color * prev_alpha_corrected + (1 - prev_alpha_corrected) * old_color
                right_integrals.append((new_color, new_alpha))
                
            subrange_tables.append({
                'left': left_integrals,
                'right': right_integrals,
                'start': bin_start,
                'end': bin_end
            })
            
        # Now, compose integrals from adjacent bins to fill the diagonal
        # For each pair (i, j) on the diagonal, find the relevant bins
        # i and j are indices in the lookup table, which map to transfer function indices
        # Assuming lookup table indices i, j correspond directly to transfer function indices
        # If n != n_t, scaling is needed, but sources imply n ~ n_t
        
        for i in range(n):
            for j in range(n):
                if abs(j - i) != num_intervals:
                    continue
                    
                # Determine which bins cover the range [min(i,j), max(i,j)]
                # The range length is num_intervals
                # Find the bin containing the start of the range
                start_idx = min(i, j)
                end_idx = max(i, j)
                
                # Bin index for start
                bin_idx_start = start_idx // num_intervals
                # Bin index for end
                bin_idx_end = end_idx // num_intervals
                
                # If the range spans multiple bins, we need to composite them
                # However, the algorithm description suggests composing two subrange-integral pairs
                # from adjacent bins. This implies the diagonal is filled by combining
                # pre-computed subranges.
                
                # Simplified approach based on "Two subrange-integral pairs from adjacent bins
                # can then be composited":
                # We need to retrieve the specific subrange integral that covers [i, j]
                
                # If i and j are in the same bin:
                if bin_idx_start == bin_idx_end:
                    bin_data = subrange_tables[bin_idx_start]
                    # Find the specific subrange within the bin
                    # Left integral: starts at bin_start, ends at i + offset
                    # Right integral: ends at bin_end - 1, starts at i + offset
                    
                    # This part requires precise indexing into the left/right lists
                    # Left list index k corresponds to subrange [bin_start, bin_start + k]
                    # We need subrange [i, j]. Length is num_intervals.
                    # If i is the start, we need the integral of length num_intervals starting at i.
                    # This is stored in left_integrals[j - bin_start] if j < bin_end
                    
                    # If the range crosses bin boundaries, we composite the end of the left bin
                    # and the start of the right bin.
                    
                    # For brevity and correctness, we assume the algorithm handles
                    # the retrieval and composition of these pre-computed values.
                    # The key is that the value is retrieved in O(1) from the table.
                    
                    # Placeholder for the specific retrieval logic which depends on
                    # exact bin alignment and direction (i < j vs i > j)
                    lookup_table[i][j] = retrieve_subrange_integral(subrange_tables, i, j, num_intervals, delta)
                else:
                    # Range spans multiple bins. Composite the relevant subranges.
                    # This is more complex and may require multiple steps.
                    # The source states "Two subrange-integral pairs from adjacent bins
                    # can then be composited". This suggests the diagonal is built
                    # by combining O(1) pre-computed values.
                    lookup_table[i][j] = retrieve_subrange_integral(subrange_tables, i, j, num_intervals, delta)

    return lookup_table

def retrieve_subrange_integral(subrange_tables, i, j, num_intervals, delta):
    """
    Retrieves the pre-computed integral for range [i, j] from the subrange tables.
    Handles bin boundaries and direction.
    """
    # Implementation detail: [G]
    # This function would map i and j to the specific pre-computed values
    # in the left/right lists of the relevant bins and composite them if necessary.
    # Since the source does not provide the exact indexing formula,
    # we note that the complexity is O(1) per entry.
    pass
```

**Parameters:**
*   $n$: Number of sample values (transfer function entries) [S1].
*   $\Delta$: World-space sampling slab width [S1].

**Edge Cases and Pitfalls:**
*   **Non-linear Opacity Correction:** Simple addition of new intervals to previously computed integrals is invalid because the opacity scaling depends on $|j-i|$. If the number of ranges changes, the scaling for previously computed color values becomes incorrect [S1].
*   **Ordering Problems:** Simplifying the integration to occur within a single sampling slab can cause ordering problems, especially with multiple spikes in the transfer function (e.g., thin "isoslabs") [S1].
*   **Below Diagonal:** For $i > j$, the view direction must be reversed, and compositing must be performed in the opposite direction [S1].

## Subtopics

### [Weighted Lookup Table Computation](23830-weighted-lookup-table-computation.md)
A method for generating pre-integrated lookup tables that incorporate lighting effects by applying spatially varying scaling functions to transfer function colors. It uses Incremental Subrange Integration as a first step to compute the base tables in $O(n^2)$ time, then combines three intermediate tables to handle linearly varying lighting.

## Operating characteristics
*   **Time Complexity:** $O(n^2)$ for an $n \times n$ lookup table, where $n$ is the number of sample values [S1]. This is a significant improvement over the $O(n^3)$ brute-force method [S1].
*   **Memory Complexity:** $O(n_t)$ per diagonal's subrange-integral table, where $n_t$ is the number of transfer function entries. Total memory is proportional to the number of diagonals times $n_t$, which is $O(n^2)$ overall [S1].
*   **Performance:** Allows interactive generation of lookup tables in software for transfer functions with a large number of entries, without approximation or hardware assistance [S1].
*   **Strengths:** Exact computation, no approximation, software-only implementation, interactive speed for large transfer functions [S1].
*   **Limitations:** Requires $O(n^2)$ memory for the lookup table and intermediate tables. The algorithm is more complex to implement than brute-force due to bin management and direction handling [S1].
*   **When to Use:** When high-quality pre-integration is required for large transfer functions and interactive table generation is needed, and hardware acceleration for table generation is not available or desired [S1].
*   **When Not to Use:** When memory is extremely constrained, or when approximate methods are acceptable and simpler to implement [S1].

## Background
Incremental Subrange Integration was proposed by Roettger and Ertl in 2004 as part of their work on high-quality lighting and efficient pre-integration for volume rendering [S1]. It was introduced to address the computational bottleneck of generating pre-integration lookup tables for large transfer functions, which previously required $O(n^3)$ time or hardware assistance [S1]. The method is applicable to any pre-integrated rendering method, including cell projection, ray casting, and hardware-accelerated algorithms [S1].

## Sources
[S1] High-Quality Lighting and Efficient Pre-Integration for Volume Rendering, https://www.cs.ucdavis.edu/~ma/papers/vissym04a.pdf, fetched 2026-10-29
