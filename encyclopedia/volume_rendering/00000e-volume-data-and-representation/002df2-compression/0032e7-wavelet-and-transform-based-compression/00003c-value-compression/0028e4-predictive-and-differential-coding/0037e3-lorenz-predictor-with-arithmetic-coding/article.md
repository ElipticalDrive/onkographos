# Lorenz Predictor with Arithmetic Coding

Also called: Lorenzo, Lorenz预测与算术编码, n-dimensional Lorenzo predictor, n-dimensional parallelogram predictor

## Overview
The Lorenz Predictor with Arithmetic Coding is a predictive compression method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression [S1, unofficial]. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An $n$-dimensional scalar field $F$ with dimensions $D_1 \times D_2 \times \dots \times D_n$.
*   **Output:** A compressed bitstream representing the residuals between actual values and predictions, encoded via arithmetic coding.

**Data Structures:**
*   **Footprint Buffer:** A circular FIFO buffer storing the values of the "footprint" (the $2^n - 1$ neighbors required for prediction).
    *   **Size:** For $R^n$, the buffer size is $\sum_{k=0}^{n-1} \prod_{i=1}^{k} D_i$ (with $\prod_{i=1}^{0} D_i = 1$). Specifically, for $R^4$, the size is $X_M Y_M Z_M + X_M Y_M + X_M + 1$ [S1, unofficial].
    *   **Layout:** The buffer must support random access to the specific neighbor indices required by the prediction formula.

**Prediction Formula:**
Let $v$ be the current sample point with local coordinates $(1, 1, \dots, 1)$ in a system where previously visited neighbors have coordinates in $\{0, 1\}^n$. The predicted value $\hat{F}(v)$ is:

$$ \hat{F}(v) = \sum_{u \in \{0,1\}^n \setminus \{v\}} (-1)^{g(u)} F(u) $$

Where:
*   $u$ iterates over all $2^n - 1$ previously visited neighbors.
*   $g(u)$ is the number of coordinates of $u$ that equal 0 (Hamming weight of the complement of $u$ relative to the all-ones vector). Equivalently, $g(u) = n - (u \cdot v)$ where $v=(1,\dots,1)$ and $u \cdot v$ is the dot product [S1, unofficial].
*   Weights are $+1$ if $g(u)$ is even, and $-1$ if $g(u)$ is odd.

**Algorithm Steps:**
1.  **Initialization:** Initialize the arithmetic encoder. Initialize the footprint buffer with zeros or appropriate boundary handling.
2.  **Traversal Order:** Process the data using the recursive `Lorenzo` function defined below, which defines a specific scan order ensuring that all $2^n - 1$ neighbors of the current point $v$ have already been processed and stored in the footprint.
3.  **Prediction:** For each sample $F[x_1, \dots, x_n]$, retrieve the $2^n - 1$ neighbor values from the footprint buffer. Compute $\hat{F}(v)$ using the formula above.
4.  **Residual Calculation:** Compute the residual $d = F[x_1, \dots, x_n] - \hat{F}(v)$.
5.  **Encoding:** Encode $d$ using an arithmetic encoder. A context arithmetic encoder (using the prediction $\hat{F}(v)$ as context) provides a ~25% gain over an adaptive arithmetic encoder [S1, unofficial].
6.  **Footprint Update:** Update the footprint buffer with the newly processed value $F[x_1, \dots, x_n]$ at the appropriate position corresponding to its coordinates.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class ArithmeticEncoder:
    """Placeholder for arithmetic encoding logic."""
    def __init__(self):
        pass
    def encode(self, value, context=None):
        # Implementation details of arithmetic encoding
        pass

class LorenzoPredictor:
    def __init__(self, dimensions):
        self.n = len(dimensions)
        self.dims = dimensions
        # Calculate footprint size: sum of products of dimensions for lower dimensions
        # Size = D1*D2*...*D(n-1) + D1*...*D(n-2) + ... + 1
        self.footprint_size = 0
        prod = 1
        for i in range(self.n - 1, -1, -1):
            self.footprint_size += prod
            if i > 0:
                prod *= dimensions[i-1]
        
        # Circular buffer for footprint
        self.footprint = [0.0] * self.footprint_size
        
    def get_footprint_index(self, coords):
        """
        Map n-dimensional coordinates (where each coord is 0 or 1)
        to a linear index in the footprint buffer.
        The buffer stores slices of dimension n-1, then n-2, etc.
        """
        # This mapping must match the traversal order.
        # Based on the recursive structure, the footprint is organized
        # such that the last dimension varies fastest in the innermost slice.
        # However, the exact linear indexing depends on the specific buffer layout.
        # A standard row-major mapping for the n-1 dimensional hypercube:
        index = 0
        multiplier = 1
        for i in range(self.n - 1):
            index += coords[i] * multiplier
            multiplier *= 2 # Since coords are 0 or 1
        return index

    def predict(self, coords):
        """
        Predict value at coords (1,1,...,1) using neighbors in {0,1}^n \ {(1,...,1)}.
        """
        prediction = 0.0
        # Iterate over all 2^n - 1 neighbors
        for i in range(1, 1 << self.n):
            # Skip the point itself (all 1s)
            if i == (1 << self.n) - 1:
                continue
            
            # Construct neighbor coordinate vector u
            u = []
            temp = i
            for _ in range(self.n):
                u.append(temp & 1)
                temp >>= 1
            
            # Calculate g(u): number of zeros in u
            # Since u is a list of 0s and 1s, g(u) = n - sum(u)
            g_u = self.n - sum(u)
            
            # Weight is (-1)^g(u)
            weight = 1 if (g_u % 2 == 0) else -1
            
            # Retrieve value from footprint
            # The neighbor u has coordinates in {0,1}^n.
            # We need to map u to the footprint buffer.
            # The footprint buffer stores the "history" of the last processed slice.
            # The mapping depends on the specific traversal.
            # For the standard Lorenzo traversal, the footprint contains the 
            # values of the cube {0,1}^n excluding the current (1,...,1).
            
            # Note: The exact linear index calculation for the footprint buffer
            # requires aligning with the recursive traversal order.
            # A simplified mapping for a 2D case (n=2):
            # Neighbors of (1,1) are (0,1), (1,0), (0,0).
            # If footprint is [F(0,0), F(1,0), F(0,1)] (row major of 1D slice + previous slice)
            # This requires careful implementation of get_footprint_index.
            
            # For the purpose of this article, we assume a helper `get_footprint_value(u)`
            # correctly retrieves F(u) from the circular buffer.
            val = self.get_footprint_value(u)
            prediction += weight * val
            
        return prediction

    def get_footprint_value(self, u):
        """
        Retrieve F(u) from the footprint buffer.
        u is a list of 0s and 1s.
        """
        # This function must be implemented to match the buffer layout.
        # For R^4, footprint size is XMYMZM + XMYM + XM + 1.
        # The buffer is a circular FIFO.
        # The index calculation is complex and depends on the current position.
        # [G] Placeholder for the specific linear indexing logic derived from the 
        # recursive traversal order and buffer layout.
        pass

    def encode_residual(self, residual):
        self.encoder.encode(residual)

    def lorenzo_recursive(self, d, coords):
        """
        Recursive traversal function.
        d: current dimension index (1-based, from 1 to n)
        coords: current coordinates for dimensions 1 to d-1 (0-based index in list)
        """
        # This is a high-level representation of the recursive structure.
        # The actual implementation iterates through the data.
        pass

    def compress(self, data):
        """
        Compress the n-dimensional data array.
        data: list of lists ... (n-dimensional)
        """
        # The recursive algorithm processes the data in a specific order.
        # Starting call for n-dimensional dataset: Lorenzo(n, -1, ..., -1)
        # The -1 indicates "not yet started" for that dimension.
        
        # We implement the iterative equivalent of the recursive traversal
        # to avoid stack overflow and manage the footprint buffer.
        
        # The traversal order is defined by the recursive function:
        # Lorenzo(d, x1...xn):
        #   if all xi != -1:
        #     predict and encode
        #   else:
        #     Lorenzo(d-1, x1...0...xn)
        #     for i = 1 to dmax:
        #       Lorenzo(d, x1...i...xn)
        
        # This corresponds to a depth-first traversal where we fill dimensions
        # from 1 to n.
        
        # Implementation of the traversal loop:
        # We use a stack to simulate the recursion.
        stack = [(self.n, [-1] * self.n)]
        
        while stack:
            d, current_coords = stack.pop()
            
            # Check if all coordinates are set (not -1)
            if all(c != -1 for c in current_coords):
                # Predict and Encode
                pred = self.predict(current_coords)
                actual = self.get_data_value(current_coords)
                residual = actual - pred
                self.encode_residual(residual)
                
                # Update footprint
                self.update_footprint(current_coords, actual)
            else:
                # Find the first dimension that is -1
                dim_idx = -1
                for i in range(self.n):
                    if current_coords[i] == -1:
                        dim_idx = i
                        break
                
                # If we are at the base case (d=1, and it's -1), we need to start from 0
                # The recursive logic:
                # Lorenzo(d, ..., -1, ...) -> Lorenzo(d-1, ..., 0, ...)
                # Then loop i=1 to dmax: Lorenzo(d, ..., i, ...)
                
                # Adjust coordinates for the recursive call
                new_coords = current_coords.copy()
                new_coords[dim_idx] = 0
                
                # Push the loop part first (so it's processed last, maintaining order)
                # Actually, recursion pushes the loop body.
                # To simulate:
                # 1. Call Lorenzo(d-1, ..., 0, ...)
                # 2. Loop i=1 to dmax: Lorenzo(d, ..., i, ...)
                
                # We push the loop iterations in reverse order so they are popped in order
                for i in range(self.dims[dim_idx], 0, -1):
                    loop_coords = current_coords.copy()
                    loop_coords[dim_idx] = i
                    stack.append((d, loop_coords))
                
                # Push the recursive call for the lower dimension
                # Note: the coordinate at dim_idx is set to 0
                stack.append((d - 1, new_coords))

    def get_data_value(self, coords):
        # Access the n-dimensional data array
        pass

    def update_footprint(self, coords, value):
        # Update the circular buffer
        pass
```

**Parameters:**
*   $n$: The dimension of the data (e.g., 3 for 3D volume, 4 for 3D+time).
*   $D_i$: The size of the data in dimension $i$.
*   **Encoder Type:** Context arithmetic encoder is recommended for a 25% gain over adaptive arithmetic coding [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The predictor is exact for polynomials of degree $< n$. For a 3D predictor applied to a cubic function, the error is between 3% and 8% (L-infinity sense). For a 4D predictor applied to a quartic function, the error is less than 1% [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes a regular grid. Boundary handling must ensure that the footprint buffer is correctly initialized and that the traversal order respects the data boundaries.
*   **Memory Footprint:** The footprint buffer size grows exponentially with $n$ if not managed carefully. For $R^4$, the buffer size is $X_M Y_M Z_M + X_M Y_M + X_M + 1$. This is significantly smaller than the full dataset, enabling out-of-core processing [S1, unofficial].

## Subtopics

### [Lorenzo Predictor](10120-lorenzo-predictor.md)
The Lorenzo predictor is an isotropic predictive coding method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$.

### [SZ-L/R](11010-sz-l-r.md)
SZ-L/R is a block-based lossless/lossy compression algorithm for volumetric data that employs Lorenzo and linear regression predictors. It partitions input data into independent 6×6×6 blocks, allowing for cache-friendly processing and random access without inter-block dependencies. Unlike global interpolation methods such as [SZ-Interp](52-sz-interp.md), SZ-L/R prioritizes local feature preservation, making it suitable for unsmooth and irregular datasets, though it may introduce block-wise artifacts.

## Operating characteristics
*   **Time Complexity:** The prediction for each sample involves $2^n - 1$ lookups and additions. The traversal is $O(N)$ where $N$ is the total number of samples.
*   **Memory Complexity:** The memory footprint is $O(\prod_{i=1}^{n-1} D_i)$, which is the size of the largest slice plus lower-dimensional footprints. This is much smaller than the full dataset $O(\prod_{i=1}^{n} D_i)$, making it suitable for out-of-core processing [S1, unofficial].
*   **Measured Performance:**
    *   On a 4D dataset of a house on fire (43,202,395 bytes uncompressed), 4D Lorenzo compression reduced the size to 524,768 bytes (1.21% of total), outperforming 3D slice compression (1,076,587 bytes, 2.49%) by 50% [S1, unofficial].
    *   On a fluid mixing simulation dataset, 3D time slice predictor produced 304,937,058 bytes (1.77 bits/symbol) and 4D Lorenzo produced 318,871,620 bytes (1.85 bits/symbol) [S1, unofficial].
    *   In lossless compression, cubic wavelets are slightly better than the Lorenzo predictor in terms of residual entropy for 4D datasets (e.g., 0.16 vs 0.20 bits/symbol for smooth data) [S1, unofficial].
    *   In lossy L-infinity compression, the Lorenzo predictor performs better than SPIHT (a wavelet implementation using S+P transform) on tested 2D datasets [S1, unofficial].
*   **Strengths:** Isotropic prediction exploits coherence in all dimensions. Small memory footprint enables out-of-core processing. Exact reconstruction of polynomials of degree $< n$.
*   **Limitations:** Anisotropic data (e.g., strong gradients in one direction) may not benefit as much. Random access is limited due to the sequential dependency of the scan order [1096-delta-encoding.md].

## Background
The Lorenz Predictor was introduced by S. Mennon and M. Rizk in the paper "Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields" [S1, unofficial]. It extends the simple two-dimensional parallelogram predictor to higher dimensions. The method is designed to handle large $n$-dimensional scalar fields where the total size exceeds available memory, by minimizing the memory footprint during compression and decompression. It has been applied to scientific datasets such as fluid mixing simulations and fire models [S1, unofficial].

## Sources
[S1] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1404787/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
