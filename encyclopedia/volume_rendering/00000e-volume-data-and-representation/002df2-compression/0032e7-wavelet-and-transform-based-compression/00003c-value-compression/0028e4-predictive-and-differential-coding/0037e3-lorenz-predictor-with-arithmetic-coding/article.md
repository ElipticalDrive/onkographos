# Lorenz Predictor with Arithmetic Coding

Also called: Lorenzo, Lorenz预测与算术编码, n-dimensional Lorenzo predictor, n-dimensional parallelogram predictor

## Overview
The Lorenz Predictor with Arithmetic Coding is a predictive compression method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression [S1, unofficial]. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An $n$-dimensional scalar field $F$ with dimensions $D_1 \times D_2 \times \dots \times D_n$.
*   **Output:** A compressed bitstream representing the residuals of the prediction.

**Data Structures:**
*   **Footprint Buffer:** A circular FIFO buffer storing the values of the "footprint" (the $2^n - 1$ neighbors required for prediction). The size of the buffer in $R^4$ is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$, where $X_M, Y_M, Z_M$ are the ranges in the respective axes [S1, unofficial]. For general $n$, the buffer stores the values of the hypercube corners excluding the current target vertex.
*   **Arithmetic Encoder:** A context-based arithmetic encoder that uses the predicted value as context for the residual [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the arithmetic encoder. Initialize the footprint buffer.
2.  **Scan Order:** Process the data using a recursive scan order defined by the `Lorenzo` function. For an $n$-dimensional dataset, the starting call is `Lorenzo(n, -1, ..., -1)` [S1, unofficial].
3.  **Prediction:** For each sample $v$ at coordinates $(x_1, \dots, x_n)$, identify the $2^n - 1$ previously visited neighbors $u$ that form the vertices of the unit hypercube $\{0,1\}^n$ excluding $v$ (where $v$ is locally $(1, \dots, 1)$). Compute the prediction $P(v)$ using the formula:
    $$ P(v) = \sum_{u \in \{0,1\}^n \setminus \{v\}} (-1)^{g(u)} F(u) $$
    where $g(u)$ is the number of coordinates of $u$ that equal 0 (i.e., the Hamming distance from $u$ to $v$ in the local $\{0,1\}^n$ frame) [S1, unofficial].
4.  **Residual Calculation:** Compute the residual $d = F(v) - P(v)$ [S1, unofficial].
5.  **Encoding:** Encode the residual $d$ using the context arithmetic encoder, using $P(v)$ as the context [S1, unofficial].
6.  **Buffer Update:** Update the footprint buffer with the new value $F(v)$ (or the reconstructed value if lossy, though the source describes lossless context) to be used for subsequent predictions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lorenzo_predictor(d, coords):
    """
    Computes the Lorenzo prediction for dimension d at the given coordinates.
    coords: tuple of length n, where -1 indicates the current dimension is not yet initialized.
    d: current dimension being processed (1 to n).
    """
    # Local coordinate system: previously visited neighbors have coords in {0,1}^d
    # The current point v is at (1, 1, ..., 1) in this local frame.
    
    prediction = 0.0
    # Iterate over all 2^d - 1 neighbors u in {0,1}^d excluding (1,1,...,1)
    for u in hypercube_neighbors(d):
        # g(u) is the number of coordinates in u that are 0
        g_u = sum(1 for coord in u if coord == 0)
        weight = (-1) ** g_u
        
        # Retrieve F(u) from the footprint buffer
        # The buffer maps the local {0,1}^d coordinates to the actual data indices
        f_u = get_footprint_value(d, u)
        
        prediction += weight * f_u
        
    return prediction

def lorenzo_compress(d, coords, data, buffer, encoder):
    """
    Recursive compression function.
    d: current dimension index (1-based, from n down to 1)
    coords: current coordinates tuple of length n
    data: the n-dimensional scalar field
    buffer: the footprint buffer
    encoder: the arithmetic encoder
    """
    # Check if all coordinates for dimensions >= d are initialized (not -1)
    # In the source, -1 indicates the start of a dimension's sweep
    if all(c != -1 for c in coords):
        # All dimensions are initialized, process this voxel
        # Compute prediction using the footprint buffer
        pred = lorenzo_predictor(d, coords)
        
        # Get actual value
        actual = get_value(data, coords)
        
        # Compute residual
        residual = actual - pred
        
        # Encode residual with context (prediction)
        encoder.encode(residual, context=pred)
        
        # Update footprint buffer with actual value
        update_footprint(buffer, coords, actual)
    else:
        # Find the first dimension index i (from d down to 1) that is -1
        # The source pseudocode uses a specific recursive structure:
        # Lorenzo(d, x1...xn)
        #   if all x1...xn != -1:
        #      predict and encode
        #   else:
        #      Lorenzo(d-1, x1...0...xn)  <-- Initialize the d-th dimension to 0
        #      for i = 1 to dmax:
        #         Lorenzo(d, x1...i...xn)
        
        # Identify the dimension to initialize
        # The source logic implies we go down dimensions until we find a -1
        # Let's implement the exact recursive step from the source:
        
        # Find the highest dimension index k <= d such that coords[k-1] == -1
        # (assuming coords is 0-indexed in code, but d is 1-indexed dimension count)
        # The source uses x_1...x_n. If d=n, we check x_1...x_n.
        
        # Simplified logic matching the source pseudocode structure:
        # We need to find the first -1 in the suffix of coords corresponding to dimensions 1..d
        
        # Let's map the source's recursive call:
        # Lorenzo(d-1, x_1, ..., x_{d-1}, 0, x_{d+1}, ..., x_n)
        # This sets the d-th coordinate to 0 and recurses on d-1.
        
        # However, the source pseudocode is:
        # Lorenzo(d, x_1,..., x_n)
        #   if all x_1,..., x_n differ from -1:
        #      ...
        #   else:
        #      Lorenzo(d - 1, x_1,..., x_{d-1}, 0, x_{d+1}, ..., x_n)
        #      for i = 1 to dmax do
        #         Lorenzo(d, x_1,..., x_{d-1}, i, x_{d+1}, ..., x_n)
        
        # This implies that if not all are set, we first initialize the current dimension d to 0
        # and recurse on d-1. Then we loop through all values for dimension d.
        
        # To implement this cleanly, we need to know which dimension is currently being initialized.
        # The parameter 'd' in the source seems to indicate the "depth" of recursion or the dimension being processed.
        # Let's assume coords is a list of length n.
        
        # Find the index of the first -1 in coords
        try:
            idx = coords.index(-1)
        except ValueError:
            # Should not happen if the logic is correct
            return

        # The source pseudocode uses 'd' as the dimension count being processed.
        # If we are at Lorenzo(d, ...), and not all are set, we set the d-th coordinate (index d-1) to 0?
        # No, the source says: Lorenzo(d-1, x_1, ..., x_{d-1}, 0, x_{d+1}, ..., x_n)
        # This sets the d-th coordinate to 0.
        
        # Let's reconstruct the loop based on the source:
        # 1. If all coords != -1: Process.
        # 2. Else:
        #    a. Set the coordinate corresponding to dimension 'd' to 0.
        #       Wait, the source pseudocode has `Lorenzo(d-1, ..., 0, ...)`
        #       This implies we are initializing the d-th dimension.
        #    b. Recurse with d-1.
        #    c. Loop i from 1 to dmax for the d-th dimension.
        
        # This structure is slightly ambiguous without knowing how 'd' maps to the index.
        # Let's assume 'd' is the dimension index (1-based) that is currently being initialized or processed.
        # If d=1, we are in the innermost loop.
        
        # Revised Pseudocode to match source exactly:
        
        # Helper to get the index of the dimension being initialized
        # In the source, the starting call is Lorenzo(4, -1, -1, -1, -1) for 4D.
        # This suggests 'd' is the dimension index being processed, and coords has length n.
        # The condition "all x_1...x_n differ from -1" checks if we are at a leaf.
        
        # Let's implement the recursion as described:
        
        # Find the first -1 in coords. Let its index be k (0-based).
        # The source uses d to track the dimension.
        # If we call Lorenzo(d, coords), and coords[d-1] is -1, we initialize it.
        
        # Actually, the source pseudocode:
        # Lorenzo(d, x_1,..., x_n)
        #   if all x_1,..., x_n differ from -1:
        #      ...
        #   else:
        #      Lorenzo(d - 1, x_1,..., x_{d-1}, 0, x_{d+1}, ..., x_n)
        #      for i = 1 to dmax do
        #         Lorenzo(d, x_1,..., x_{d-1}, i, x_{d+1}, ..., x_n)
        
        # This implies that if not all are set, we assume the dimension 'd' is the one being initialized.
        # We set x_d = 0, and recurse on d-1.
        # Then we loop i from 1 to dmax for x_d, and recurse on d.
        
        # This means 'd' must be the index of the first -1? Or the highest dimension?
        # The starting call is Lorenzo(4, -1, -1, -1, -1).
        # First call: all are -1. Else branch.
        # Lorenzo(3, -1, -1, 0, -1) -> sets 4th coord to 0.
        # Then loop i=1..dmax for 4th coord: Lorenzo(4, -1, -1, i, -1).
        
        # This suggests 'd' is the dimension index of the coordinate being initialized.
        
        # Let's implement this logic:
        
        # Find the index of the first -1 in coords
        try:
            k = coords.index(-1)
        except ValueError:
            # All coordinates are set
            pred = lorenzo_predictor(len(coords), coords)
            actual = get_value(data, coords)
            residual = actual - pred
            encoder.encode(residual, context=pred)
            update_footprint(buffer, coords, actual)
            return

        # k is the 0-based index of the dimension being initialized.
        # The source uses 1-based 'd'. So d = k + 1.
        d = k + 1
        
        # Create a new coords list for the recursive call
        # Set the k-th coordinate to 0
        coords_init = list(coords)
        coords_init[k] = 0
        
        # Recurse on d-1 (which is k)
        # Note: The source calls Lorenzo(d-1, ...)
        # If d=1, Lorenzo(0, ...) is not defined in the source, but the base case is "all != -1".
        # If d=1, and we set x_1=0, then all coords might still be -1?
        # No, the source says "Lorenzo(d-1, x_1, ..., x_{d-1}, 0, x_{d+1}, ..., x_n)".
        # If d=1, this is Lorenzo(0, 0, x_2, ..., x_n).
        # But the condition is "all x_1...x_n differ from -1".
        # If we are at Lorenzo(1, -1, -1, -1, -1), we set x_1=0.
        # Then we call Lorenzo(0, 0, -1, -1, -1).
        # This will fail the "all != -1" check.
        
        # There is a discrepancy in my interpretation.
        # Let's look at the 2D example:
        # Lorenzo(2, -1, -1)
        # Lorenzo(1, -1, 0)
        # Lorenzo(0, 0, 0) -> Encodes H[0,0]
        # Lorenzo(1, 1, 0) -> Encodes H[1,0]-H[0,0]
        # ...
        
        # It seems 'd' is the dimension index, and the recursion goes down to d=0?
        # Or maybe 'd' is the number of dimensions remaining?
        # In the 2D example, Lorenzo(0, 0, 0) is called.
        # This suggests that when d=0, we are at the base case?
        # But the condition is "all x_1...x_n differ from -1".
        # In Lorenzo(0, 0, 0), all are 0, so they differ from -1.
        # So the condition is met.
        
        # So the logic is:
        # If all coords != -1: Process.
        # Else:
        #   Find the first -1 at index k.
        #   Set coords[k] = 0.
        #   Recurse with d-1? No, the source says Lorenzo(d-1, ...).
        #   If d=2, k=0 (first -1). Set coords[0]=0. Call Lorenzo(1, 0, -1).
        #   In Lorenzo(1, 0, -1), all are not -1 (x_2 is -1).
        #   Find first -1 at index 1. Set coords[1]=0. Call Lorenzo(0, 0, 0).
        #   In Lorenzo(0, 0, 0), all are not -1. Process.
        
        # So 'd' is just a parameter that decreases by 1 each time we initialize a dimension.
        # It doesn't affect the logic other than tracking depth?
        # The source pseudocode uses 'd' in the loop: "for i = 1 to dmax do Lorenzo(d, ...)".
        # This suggests 'd' is the dimension index for the loop.
        
        # Let's implement the exact source logic:
        
        # 1. Check if all coords != -1
        if all(c != -1 for c in coords):
            pred = lorenzo_predictor(len(coords), coords)
            actual = get_value(data, coords)
            residual = actual - pred
            encoder.encode(residual, context=pred)
            update_footprint(buffer, coords, actual)
            return

        # 2. Find the first -1
        k = coords.index(-1)
        d = k + 1  # 1-based dimension index
        
        # 3. Set coords[k] = 0 and recurse on d-1
        coords_init = list(coords)
        coords_init[k] = 0
        lorenzo_compress(d - 1, coords_init, data, buffer, encoder)
        
        # 4. Loop i from 1 to dmax for coords[k]
        dmax = data.shape[k]  # Assuming data is a numpy array or similar
        for i in range(1, dmax + 1):
            coords_loop = list(coords)
            coords_loop[k] = i
            lorenzo_compress(d, coords_loop, data, buffer, encoder)

def get_footprint_value(d, u):
    """
    Retrieves the value F(u) from the footprint buffer.
    u is a tuple of 0s and 1s in the local frame.
    """
    # Map local u to global coordinates using the current state of the buffer
    # This requires maintaining a mapping of local coordinates to global indices
    # For simplicity, we assume the buffer stores values indexed by local coordinates
    return buffer.get_value(u)

def update_footprint(buffer, coords, value):
    """
    Updates the footprint buffer with the new value.
    """
    # Map global coords to local coordinates in the buffer
    # The buffer is a circular FIFO, so we need to manage indices
    buffer.set_value(coords, value)
```

**Parameters:**
*   **Dimension $n$:** The number of dimensions in the scalar field. Typical values are 3 or 4 [S1, unofficial].
*   **Arithmetic Encoder:** Context-based arithmetic encoder is recommended for a 25% gain over adaptive arithmetic encoding [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The predictor is exact for polynomials of degree less than $n$. For a 3D predictor applied to a cubic function, the error is between 3% and 8% (L-infinity sense) [S1, unofficial]. For a 4D predictor applied to a quartic function, the error is less than 1% [S1, unofficial].
*   **Boundaries:** The algorithm assumes a regular grid. Boundary conditions must be handled by the scan order (the recursive structure naturally handles boundaries by initializing dimensions sequentially) [S1, unofficial].
*   **Memory:** The footprint buffer size is critical for out-of-core implementations. The buffer size in $R^4$ is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$ [S1, unofficial].

## Subtopics

### [Lorenzo Predictor](10120-lorenzo-predictor.md)
The Lorenzo predictor is an isotropic predictive coding method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The prediction for each sample involves $2^n - 1$ lookups and additions. The complexity per sample is $O(2^n)$. The total time complexity is $O(N \cdot 2^n)$ where $N$ is the total number of samples [S1, unofficial].
*   **Memory:** The memory footprint is limited to the footprint buffer. For $R^4$, the buffer size is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$ [S1, unofficial]. This allows for out-of-core processing where the total dataset size exceeds available memory [S1, unofficial].

**Measured Performance:**
*   On a 3D model of a house on fire (43,202,395 bytes uncompressed), 4D Lorenzo compression reduced the size to 524,768 bytes (1.21% of total), outperforming 3D slice compression (1,076,587 bytes, 2.49%) by 50% [S1, unofficial].
*   On a fluid mixing simulation dataset, 3D time slice predictor produced 304,937,058 bytes (1.77 bits/symbol) and 4D Lorenzo produced 318,871,620 bytes (1.85 bits/symbol) [S1, unofficial].
*   Context arithmetic encoder gives a 25% gain over adaptive arithmetic encoder for encoding residuals [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Isotropic prediction exploits data coherence in all dimensions. Exact reconstruction of polynomials of degree less than $n$. Small memory footprint suitable for out-of-core processing.
*   **Limitations:** For data with higher-degree polynomials, prediction error increases. In lossless compression, cubic wavelets are slightly better than the Lorenzo predictor in terms of residual entropy for 4D datasets [S1, unofficial].

**When to Use:**
*   Use for large $n$-dimensional scalar fields where data coherence exists in all dimensions and out-of-core processing is required.
*   Use when isotropic prediction is beneficial over anisotropic scanline prediction.

**When Not to Use:**
*   Use wavelets if lossless compression of 4D data is required and cubic wavelets are available, as they are slightly better in terms of residual entropy [S1, unofficial].
*   Use if the data has high gradients or discontinuities that are not well-captured by polynomial prediction.

## Background
The Lorenz Predictor was introduced by S. Mennon and M. Rizk in the paper "Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields" [S1, unofficial]. It extends the simple two-dimensional parallelogram predictor to higher dimensions. The method is designed for large datasets that exceed available memory, allowing for out-of-core compression and decompression. It has been tested on synthetic data and real datasets such as fluid mixing simulations and fire models [S1, unofficial].

## Sources
[S1] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1404787/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
