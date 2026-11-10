# Lorenz Predictor with Arithmetic Coding

Also called: Lorenzo, Lorenz预测与算术编码, n-dimensional Lorenzo predictor, n-dimensional parallelogram predictor

## Overview
The Lorenz Predictor with Arithmetic Coding is a lossless compression method for $n$-dimensional scalar fields that estimates the value of a sample point using its $2^n - 1$ previously visited neighbors in a regular grid to generate residuals, which are then encoded using arithmetic coding [S1, unofficial]. It is distinguished from anisotropic scanline predictors by being isotropic, exploiting data coherence in all dimensions simultaneously, and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An $n$-dimensional scalar field $F$ defined on a grid with dimensions $X_{max}, Y_{max}, \dots, T_{max}$.
*   **Output:** A compressed bitstream representing the residuals between actual values and predicted values.

**Data Structures:**
*   **Footprint Buffer:** A circular FIFO vector used to store the immediate neighbors required for prediction. For an $n$-dimensional dataset, the buffer size is determined by the sum of the sizes of slices in lower dimensions. Specifically, for $R^4$, the size is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$, where $X_M, Y_M, Z_M$ are the ranges in the respective axes [S1, unofficial]. This small footprint allows for out-of-core processing [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Start the recursive traversal with the dimension $d=n$ and all coordinates set to $-1$ (indicating unvisited). Call `Lorenzo(n, -1, ..., -1)` [S1, unofficial].
2.  **Recursive Traversal:** The function `Lorenzo(d, x_1, ..., x_n)` processes the data in a specific order:
    *   If all coordinates $x_1, \dots, x_n$ are not $-1$ (i.e., a valid sample point is reached):
        *   Compute the prediction $P(v)$ for the sample $v$ at coordinates $(x_1, \dots, x_n)$ using the Lorenz Predictor formula.
        *   Calculate the residual $R = F(v) - P(v)$.
        *   Encode $R$ using an arithmetic encoder. A context arithmetic encoder, using the prediction $P(v)$ as context, provides a 25% gain over an adaptive arithmetic encoder [S1, unofficial].
    *   If any coordinate is $-1$:
        *   Find the first index $d$ such that $x_d = -1$.
        *   Recursively call `Lorenzo(d-1, x_1, ..., x_{d-1}, 0, x_{d+1}, ..., x_n)` to initialize the $d$-th dimension at 0.
        *   Loop $i$ from 1 to $d_{max}$ (the size of the $d$-th dimension):
            *   Call `Lorenzo(d, x_1, ..., x_{d-1}, i, x_{d+1}, ..., x_n)` [S1, unofficial].
3.  **Prediction Formula:** For a sample $v$ with local coordinates $(1, 1, \dots, 1)$ relative to the unit hypercube of previously visited neighbors $\{0, 1\}^n$, the predicted value $P(v)$ is:
    $$ P(v) = \sum_{u \in \{0,1\}^n \setminus \{v\}} (-1)^{g(u)} F(u) $$
    where $g(u)$ is the number of coordinates of $u$ that equal 0 (equivalently, the Hamming distance from $u$ to $v$ in the hypercube, or $n - u \cdot v$ where $u \cdot v$ is the dot product) [S1, unofficial]. The weights are $+1$ if $g(u)$ is even and $-1$ if $g(u)$ is odd.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lorenzo_predictor(dim, coords):
    """
    Computes the Lorenz prediction for the sample at coords in a hypercube
    of previously visited neighbors.
    coords: tuple of length 'dim', values are 0 or 1.
    """
    prediction = 0.0
    for u in product([0, 1], repeat=dim):
        if u == coords:
            continue
        # g(u) is the number of zeros in u
        g_u = u.count(0)
        weight = 1 if g_u % 2 == 0 else -1
        # Retrieve F(u) from the footprint buffer
        # Note: In a real implementation, coords map to buffer indices
        f_u = get_footprint_value(u) 
        prediction += weight * f_u
    return prediction

def encode_residuals(dim, coords, max_dims):
    """
    Recursive function to traverse the n-dimensional grid and encode residuals.
    dim: current dimension being initialized (starts at n)
    coords: current coordinates tuple
    max_dims: tuple of max sizes for each dimension
    """
    # Find the first dimension that is -1 (uninitialized)
    try:
        d = coords.index(-1)
    except ValueError:
        # All coordinates are initialized, process the sample
        # Convert -1s to actual indices if necessary, but here coords 
        # should be fully populated 0..max-1 when reaching this block
        # The logic in source uses -1 as a marker for "not yet visited in this recursion path"
        # When all are != -1, we are at a valid point.
        
        # Note: The source pseudocode implies coords are passed as is.
        # If we reach here, coords has no -1.
        
        # Compute prediction
        # The source defines local coords (1,1,1) for the current point.
        # We need to map global coords to the local {0,1}^n cube of neighbors.
        # The neighbors are the 2^n - 1 points in the hypercube defined by 
        # the previous indices in each dimension.
        
        # Simplified: Assume get_footprint_value handles the mapping from 
        # local relative coordinates to the buffer.
        # The 'coords' here are the global indices.
        # The 'dim' parameter in the source recursive call suggests we are 
        # building up the coordinate tuple.
        
        # Re-interpreting source logic:
        # The source passes 'd' (dimension index) and 'x_1...x_n'.
        # When all x_i != -1, we compute prediction for F[x_1...x_n].
        
        # To implement strictly from source:
        # We need access to F. In a streaming context, F is read from disk/memory.
        # We need access to neighbors. They are in the footprint buffer.
        
        # Let's assume 'coords' are the global indices.
        # The neighbors are the 2^n - 1 points in the hypercube ending at coords.
        # Specifically, if current is (x,y,z), neighbors are (x-1,y-1,z-1) etc.
        
        # However, the source uses a specific recursion:
        # Lorenzo(d, x1...xn)
        # If all != -1: Predict and Encode.
        # Else: Initialize x_d to 0, then loop x_d from 1 to dmax.
        
        # This recursion order ensures that when we are at (x1...xn),
        # all points in the hypercube {0,1}^n relative to (1,1...1) 
        # (which corresponds to the previous indices) have been visited.
        
        # Implementation detail:
        # The 'coords' in the final call are the global indices.
        # The prediction uses the 2^n - 1 neighbors.
        
        # For the pseudocode to be self-contained, we assume a 'data' structure
        # and a 'buffer' structure.
        
        # Since the source recursion is complex to map directly to a simple loop
        # without the state machine of 'd', we will write the recursive function.
        
        # Get the actual value F(v)
        f_v = get_data_value(coords)
        
        # Get prediction
        # The neighbors are the 2^n - 1 points in the hypercube defined by
        # coords - (1,1,...,1) to coords - (0,0,...,0) excluding coords itself.
        # In the source's local system, v is (1,1,...,1) and neighbors are {0,1}^n \ {v}.
        
        # We need to map global coords to the local hypercube indices.
        # The neighbors are at indices:
        # For each bit mask b in 0..2^n-1, if b != (2^n-1):
        #   neighbor_index = coords - (1,1,...,1) + b
        #   Wait, the source says "previously visited neighbors forming the other corners".
        #   The recursion order guarantees these are in the buffer.
        
        # Let's use the source's local coordinate definition for the formula.
        # P(v) = sum_{u in {0,1}^n \ {v}} (-1)^g(u) F(u)
        # Here u are local coordinates.
        
        prediction = 0.0
        n = len(coords)
        for i in range(1 << n):
            u = tuple(1 if (i >> j) & 1 else 0 for j in range(n))
            if u == tuple(1 for _ in range(n)):
                continue
            g_u = u.count(0)
            weight = 1 if g_u % 2 == 0 else -1
            
            # Map local u to global neighbor index
            # The current point is coords.
            # The hypercube of neighbors is defined by the previous indices.
            # If current is (x,y,z), the neighbors are (x-1,y-1,z-1) to (x,y,z) excluding (x,y,z).
            # Actually, the source says "previously visited".
            # The recursion visits (0,0,0), then (1,0,0)...
            # When at (1,1,1), the neighbors are (0,0,0), (1,0,0), (0,1,0), (0,0,1), (1,1,0), (1,0,1), (0,1,1).
            # These are exactly coords - (1,1,1) + u for u in {0,1}^n \ {1,1,1}.
            
            neighbor_global = tuple(coords[k] - 1 + u[k] for k in range(n))
            f_u = get_footprint_value(neighbor_global)
            prediction += weight * f_u
            
        residual = f_v - prediction
        arithmetic_encode(residual, context=prediction)
        return

    # If we are here, coords has -1s.
    # Find the first -1
    d = coords.index(-1)
    
    # Initialize this dimension to 0
    new_coords = list(coords)
    new_coords[d] = 0
    new_coords = tuple(new_coords)
    
    # Recurse to initialize lower dimensions if necessary
    # The source calls Lorenzo(d-1, ..., 0, ...)
    # This effectively sets the d-th dim to 0 and recurses to set d-1 to 0, etc.
    # But wait, the source code:
    # Lorenzo(d-1, x1...xd-1, 0, xd+1...xn)
    # This sets the d-th coordinate to 0, and then recurses on d-1.
    # This ensures that when we start the loop for d, the lower dimensions are initialized.
    
    lorenzo_predictor_with_arithmetic(d - 1, new_coords, max_dims)
    
    # Loop for the current dimension d
    for i in range(1, max_dims[d]):
        new_coords = list(coords)
        new_coords[d] = i
        new_coords = tuple(new_coords)
        lorenzo_predictor_with_arithmetic(d, new_coords, max_dims)

# Wrapper
def compress_volume(data, max_dims):
    n = len(max_dims)
    start_coords = tuple([-1] * n)
    lorenzo_predictor_with_arithmetic(n, start_coords, max_dims)
```

**Parameters:**
*   $n$: The dimension of the data (e.g., 3 for 3D volumes, 4 for 4D spatiotemporal data) [S1, unofficial].
*   $d_{max}$: The size of each dimension [S1, unofficial].
*   **Arithmetic Encoder:** A context arithmetic encoder is recommended over an adaptive one for a 25% gain [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Polynomial Degree:** The predictor is exact for polynomials of degree $< n$. For a 3D predictor applied to a cubic function, the error is between 3% and 8% (L-infinity sense) [S1, unofficial]. For a 4D predictor applied to a quartic function, the error is less than 1% [S1, unofficial].
*   **Memory Footprint:** The buffer size grows exponentially with dimension if not managed as a circular FIFO. The specific formula for $R^4$ is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$ [S1, unofficial].
*   **Boundary Conditions:** The recursion handles boundaries by initializing dimensions to 0 and looping up to $d_{max}$. The predictor requires $2^n - 1$ neighbors, so it cannot predict the first sample in any dimension if not handled by the initialization logic (which sets the first sample to be encoded without prediction or uses a different base case, but the source implies the recursion handles the "first" points by initializing them to 0 and then processing). Note: The source pseudocode encodes the first point (0,0,0) after initializing lower dims. The prediction for (0,0,0) would require neighbors with negative indices, which don't exist. The source example trace shows `Encodes H[0,0]` for the first point, implying no prediction or a special case. The pseudocode above needs a guard: if any coordinate is 0, prediction might be invalid. The source trace: `Lorenzo(0,0,0); Encodes H[0,0]`. This suggests that for the very first point, or points on the boundary where neighbors are missing, the residual is just the value itself (or a different predictor is used). The source states "previously visited neighbors". For (0,0,0), there are no previously visited neighbors in the hypercube sense. Thus, the residual is $F(0,0,0) - 0$ or simply encoded directly. The source does not explicitly define the prediction for boundary points where neighbors are missing, but the trace implies direct encoding. [G: For points where any coordinate is 0, the residual is the value itself, as no neighbors exist in the positive direction within the processed set.]

## Subtopics

### [Lorenzo Predictor](10120-lorenzo-predictor.md)
The Lorenzo predictor is an isotropic predictive coding method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

### [SZ-L/R](11010-sz-l-r.md)
SZ-L/R is a block-based lossless/lossy compression algorithm for volumetric data that employs Lorenzo and linear regression predictors. It partitions input data into independent 6×6×6 blocks, allowing for cache-friendly processing and random access without inter-block dependencies. Unlike global interpolation methods such as [SZ-Interp](52-sz-interp.md), SZ-L/R prioritizes local feature preservation, making it suitable for unsmooth and irregular datasets, though it may introduce block-wise artifacts.

## Operating characteristics
*   **Time Complexity:** The predictor is of the highest possible order among all predictors attempting to predict the value at one corner of a cube from the other $2^n - 1$ corners [S1, unofficial]. The computational cost per sample is $O(2^n)$ for the prediction sum.
*   **Memory Complexity:** The memory footprint is small, proportional to the size of one slice plus the footprint of the lower-dimensional data. For $R^4$, it is $X_M Y_M Z_M + X_M Y_M + X_M + 1$ [S1, unofficial]. This makes it suitable for out-of-core implementations [S1, unofficial].
*   **Measured Performance:**
    *   On a 3D model of a house on fire (43,202,395 bytes uncompressed), 4D Lorenzo compression reduced the size to 524,768 bytes (1.21% of total), outperforming 3D slice compression (1,076,587 bytes, 2.49%) by 50% [S1, unofficial].
    *   On a fluid mixing simulation dataset, 3D time slice predictor produced 304,937,058 bytes (1.77 bits/symbol) and 4D Lorenzo produced 318,871,620 bytes (1.85 bits/symbol) [S1, unofficial].
    *   In lossless compression of 4D datasets, cubic wavelets are slightly better than the Lorenzo predictor in terms of residual entropy (e.g., 0.16 vs 0.20 bits/symbol for smooth data, 3.28 vs 3.73 bits/symbol for harsh data) [S1, unofficial].
    *   In lossy L-infinity compression, the Lorenzo predictor performs better than SPIHT on tested 2D datasets [S1, unofficial].
*   **Strengths:** Isotropic prediction, exact reconstruction of polynomials of degree $< n$, small memory footprint, suitable for out-of-core processing.
*   **Limitations:** For data with high-frequency components or polynomials of degree $\ge n$, prediction errors are non-zero. In lossless 4D compression, cubic wavelets may achieve slightly lower entropy.

## Background
The Lorenz Predictor (also known as the Lorenzo predictor) was introduced by S. Mennon and M. Rizk in the paper "Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields" [S1, unofficial]. It extends the simple two-dimensional parallelogram predictor to higher dimensions. The method is particularly noted for its application in large-scale scientific simulations, such as fluid dynamics and climate modeling, where data coherence exists in all dimensions. It is used in conjunction with arithmetic coding to achieve high compression ratios for large n-dimensional scalar fields.

## Sources
[S1] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1404787/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
