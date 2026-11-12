# Lorenzo Predictor

Also called: Lorenzo compression, n-dimensional Lorenzo predictor, n-dimensional parallelogram predictor

## Overview
The Lorenzo predictor is an isotropic predictive coding method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression [S1, unofficial]. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An $n$-dimensional scalar field $F$ defined on a grid with dimensions $D_1 \times D_2 \times \dots \times D_n$.
*   **Output:** A stream of encoded residuals (differences between actual values and predictions).

**Data Structures:**
*   **Footprint Buffer:** A circular FIFO queue storing necessary neighbor values. For a dataset of size $D^n$, the footprint size is $D^{n-1} + D^{n-2} + \dots + D + 1$ [S1, unofficial]. This allows out-of-core processing where only a single $(n-1)$-dimensional slice plus smaller sub-slices are kept in memory [S1, unofficial].
*   **Coordinates:** Traversal uses a recursive state $(d, x_1, \dots, x_n)$ where $d$ is the current recursion depth/dimension, $x_i$ are coordinates, and $-1$ indicates an unassigned coordinate [S1, unofficial].

**Prediction Formula:**
For a target vertex $v$ (represented as the all-ones vector $(1, 1, \dots, 1)$ in the local unit cube relative to the traversal order), the predicted value $E(v)$ is:
$$E(v) = \sum_{u \in U} (-1)^{c_0(u)+1} F(u)$$
Where:
*   $U = \mathbb{Z}_2^n - \{v\}$ is the set of $2^n - 1$ previously processed neighbors (vertices of the unit cube excluding $v$) [S1, unofficial].
*   $c_0(u)$ is the number of coordinates in $u$ that equal zero [S1, unofficial].
*   $c_0(u)$ can be computed as $n - c_1(u) = n - (u \cdot v)$, where $n$ is the dimension, $c_1(u)$ is the number of coordinates in $u$ equal to one, and $u \cdot v$ is the dot product of $u$ and $v=(1,1,\dots,1)$ [S1, unofficial].

**Weight Assignment:**
*   Immediate neighbors (distance 1 in the cube graph) have weight $+1$ [S1, unofficial].
*   Second-degree neighbors (distance 2) have weight $-1$ [S1, unofficial].
*   Third-degree neighbors have weight $+1$, alternating based on the minimal number of cube edges between the neighbor and $v$ [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the recursion with `Lorenzo(n, -1, -1, ..., -1)` [S1, unofficial].
2.  In `Lorenzo(d, x_1, ..., x_n)`:
    *   If all $x_i \neq -1$:
        *   Compute prediction $E$ using the formula above based on the $2^n - 1$ neighbors already in the footprint [S1, unofficial].
        *   Compute residual $R = F[x_1, \dots, x_n] - E$ [S1, unofficial].
        *   Encode $R$ (losslessly or lossily) [S1, unofficial].
        *   Update the footprint with the actual value $F[x_1, \dots, x_n]$ to be used by future predictions [S1, unofficial].
    *   Else (some $x_i = -1$):
        *   Identify $k$, the greatest index such that $x_k = -1$ [S1, unofficial].
        *   Recursively call `Lorenzo(d-1, x_1, ..., x_{k-1}, 0, x_{k+1}, ..., x_n)` to initialize the current dimension's start [S1, unofficial].
        *   Loop $i$ from $1$ to $d_{max}$ (size of dimension $d$):
            *   Recursively call `Lorenzo(d, x_1, ..., x_{k-1}, i, x_{k+1}, ..., x_n)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import itertools

def lorenzo_predictor(coords):
    """Compute prediction E(v) for a point given its relative 0/1 coordinates in the unit cube."""
    n = len(coords)
    prediction = 0.0
    # Iterate over all 2^n - 1 neighbors u in Z_2^n excluding v=(1,1,...,1)
    for u in itertools.product([0, 1], repeat=n):
        if u == coords:
            continue
        # c0(u) is number of zeros in u
        c0_u = u.count(0)
        weight = (-1) ** (c0_u + 1)
        # Retrieve F(u) from the footprint buffer using absolute coordinates derived from u
        # This mapping depends on the traversal order and current absolute indices.
        value_u = get_footprint_value(u)
        prediction += weight * value_u
    return prediction

def lorenzo_traverse(d, coords, dmax_list, footprint):
    """Recursive traversal to compress the n-dimensional array."""
    n = len(dmax_list)
    # Base case: all coordinates assigned
    if all(c != -1 for c in coords):
        # Map relative 0/1 coords to absolute indices to fetch from footprint
        neighbor_values = get_neighbor_values_from_footprint(coords, footprint)
        prediction = sum((-1) ** (u.count(0) + 1) * val for u, val in zip(itertools.product([0, 1], repeat=n), neighbor_values) if u != tuple(coords))
        residual = get_value_at_coords(coords) - prediction
        # Encode residual losslessly or lossily
        encode_residual(residual)
        # Update footprint with actual value
        update_footprint(coords, get_value_at_coords(coords))
        return

    # Recursive case: find k, the greatest index where coords[k] == -1
    k = -1
    for i in range(n - 1, -1, -1):
        if coords[i] == -1:
            k = i
            break
            
    # Initialize the k-th dimension to 0
    current_dim_idx = k  # 0-based
    dim_size = dmax_list[current_dim_idx]
    
    # Set x_k = 0
    coords[k] = 0
    
    # Recursive call to initialize lower dimensions (as per paper pseudocode)
    lorenzo_traverse(d - 1, coords, dmax_list, footprint)
    
    # Loop over the current dimension
    for i in range(1, dim_size + 1):
        coords[k] = i
        lorenzo_traverse(d, coords, dmax_list, footprint)
```
*Note: Helper functions `get_footprint_value`, `get_neighbor_values_from_footprint`, `get_value_at_coords`, `encode_residual`, and `update_footprint` implement standard buffer indexing, value retrieval, arithmetic encoding, and FIFO queue updates respectively [G].*

**Parameters:**
*   $n$: Dimension of the predictor (and dataset) [S1, unofficial].
*   $d_{max}$: Number of samples in the current dimension [S1, unofficial].
*   $k$: Greatest index where $x_k = -1$, identifying the current dimension being iterated [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Polynomial Degree Limit:** The predictor cannot recover the value of an $n$-th degree polynomial at the corner $v$ from the other corners because it cannot differentiate the monomial $x_1 x_2 \dots x_n$ (which is 1 at $v$ and 0 elsewhere on the cube) from the zero polynomial [S1, unofficial].
*   **Dimension Lengths:** If dimensions have different lengths, the footprint size depends on the traversal order. The order should be chosen to minimize the footprint size [S1, unofficial].
*   **Memory Scarcity:** If memory is scarce for the arithmetic encoder's probability table, only frequently occurring small corrections around zero need to be compressed. Occasional large corrections can be flagged and transmitted verbatim [S1, unofficial].

## Subtopics

### Lorenzo Lossy Compression with Residual Encoding
Lossy compression using the Lorenzo predictor allows a small discrepancy between compressed and real data to improve the compression ratio, with a 5% tolerance reducing file size by half [S2, unofficial]. It supports two error metrics: $L_\infty$ (maximum error) and $L_2$ (root mean square error) [S2, unofficial]. To guarantee $L_\infty$ error does not exceed a threshold, residuals are quantized and values at the scalar field are readjusted to compensate for error accumulation, with adjusted corrections encoded losslessly [S2, unofficial]. Using a context arithmetic encoder (using the actual prediction as context) provides a 25% gain over an adaptive arithmetic encoder [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The footprint size is $D^{n-1} + D^{n-2} + \dots + D + 1$ for a dataset of size $D^n$, allowing out-of-core processing with only a small buffer in core memory [S1, unofficial].
*   **Time:** Linear in the number of samples, with $O(2^n)$ operations per sample for prediction [S1, unofficial].

**Measured Performance:**
*   On a 4D house fire dataset (43,202,395 bytes uncompressed), 4D Lorenzo compression achieved 524,768 bytes (1.21% of total), outperforming 3D slice compression (1,076,587 bytes, 2.49%) by 50% [S1, unofficial].
*   On a LLNL fluid mixing dataset, 3D Lorenzo predictor on time slices produced 304,937,058 bytes (1.77 bits/symbol), while 4D prediction produced 318,871,620 bytes (1.85 bits/symbol). 3D was better due to temporal subsampling reducing coherence [S1, unofficial].
*   For a 3D predictor applied to a cubic function, the relative error is between 3% and 8% ($L_\infty$ sense). For a 4D predictor applied to a quartic function, the relative error is less than 1% [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Isotropic prediction exploits data coherence in all dimensions. Exact reconstruction of polynomials of degree $< n$. Out-of-core capability.
*   **Limitations:** Cannot recover $n$-th degree polynomials. Performance degrades if data coherence is low (e.g., due to subsampling) [S1, unofficial].

## Background
The Lorenzo predictor was introduced in the paper "Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields" [S1, unofficial]. It extends a simple two-dimensional parallelogram predictor to higher dimensions [S1, unofficial]. It is named after Lorenzo, likely referring to the author or a colleague, though the specific origin of the name is not detailed in the provided sources. It is used in volume rendering and scientific data visualization for compressing large $n$-dimensional scalar fields [S1, unofficial].

## Sources
[S1] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1406792/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1404787/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
