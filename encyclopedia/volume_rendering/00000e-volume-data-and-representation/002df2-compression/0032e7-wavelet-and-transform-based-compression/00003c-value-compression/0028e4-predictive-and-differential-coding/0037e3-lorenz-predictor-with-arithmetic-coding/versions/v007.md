# Lorenz Predictor with Arithmetic Coding

Also called: Lorenzo, Lorenz预测与算术编码, n-dimensional Lorenzo predictor, n-dimensional parallelogram predictor

## Overview
The Lorenz Predictor with Arithmetic Coding is a lossless compression method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals, which are then entropy-encoded using arithmetic coding [S1, unofficial]. It is distinguished from anisotropic scanline predictors by being isotropic, exploiting data coherence in all dimensions, and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An $n$-dimensional scalar field $F$ defined on a grid with dimensions $X_M \times Y_M \times \dots \times T_M$.
*   **Output:** A compressed bitstream representing the residuals of the prediction.

**Data Structures:**
*   **Footprint Buffer:** A circular FIFO buffer storing the values of previously visited neighbors required for prediction. For a 4D dataset, the buffer size is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$ [S1, unofficial]. The buffer must support random access to the specific neighbors defined by the predictor's geometry.

**Prediction Model:**
The predictor $\hat{v}$ estimates the value of a sample $v$ based on the values at the $2^n - 1$ previously visited neighbors forming the other corners of the $n$-dimensional unit hypercube of which $v$ is the vertex $(1, \dots, 1)$ [S1, unofficial].

Let $u \in \{0, 1\}^n$ be the coordinates of a neighbor relative to $v$ in a local coordinate system where $v$ is at $(1, \dots, 1)$. The weight for each neighbor is determined by $g(u)$, the number of coordinates of $u$ that equal 0 (equivalently, the Hamming distance from the all-ones vector, or $n - u \cdot v$) [S1, unofficial].

The predicted value $P(v)$ is:
$$ P(v) = \sum_{u \in \{0,1\}^n \setminus \{(1,\dots,1)\}} (-1)^{g(u)} F(u) $$

**Residual Calculation:**
The residual $d$ is calculated as:
$$ d = F(v) - P(v) $$

**Entropy Encoding:**
The residuals are encoded using a context arithmetic encoder. The context for the encoder is the actual prediction value $P(v)$ [S1, unofficial]. This context-based approach provides a 25% gain over an adaptive arithmetic encoder [S1, unofficial].

**Scan Order and Recursion:**
The data is processed in a specific recursive scan order to ensure that all neighbors required for prediction have already been encoded [S1, unofficial]. The algorithm `Lorenzo(d, x_1, \dots, x_n)` processes the data as follows:

1.  If all coordinates $x_1, \dots, x_n$ differ from $-1$ (indicating a valid sample point in the current dimension $d$):
    *   Compute prediction $p = \text{LorenzoPredictor}(d, x_1, \dots, x_n)$.
    *   Encode the residual $F[x_1, \dots, x_n] - p$.
2.  Else (indicating a boundary or dimension transition):
    *   Recursively call `Lorenzo(d - 1, x_1, \dots, x_{d-1}, 0, x_{d+1}, \dots, x_n)` to initialize the next lower dimension.
    *   Loop $i$ from 1 to $d_{max}$ (the size of the $d$-th dimension):
        *   Call `Lorenzo(d, x_1, \dots, x_{d-1}, i, x_{d+1}, \dots, x_n)`.

The starting call for a 4D dataset is `Lorenzo(4, -1, -1, -1, -1)` [S1, unofficial].

**Pseudocode:**

```python
import math

class ArithmeticEncoder:
    """
    Simplified context arithmetic encoder.
    In a real implementation, this would handle bit-stream I/O and probability modeling.
    """
    def encode_residual(self, residual, context_prediction):
        # Context is the predicted value.
        # The encoder models the probability distribution of residuals given the prediction.
        # This is a placeholder for the actual arithmetic coding logic.
        pass

class LorenzPredictor:
    def __init__(self, data, dimensions):
        self.data = data
        self.dimensions = dimensions # List of sizes [X, Y, Z, T]
        self.n = len(dimensions)
        self.encoder = ArithmeticEncoder()
        # Footprint buffer for out-of-core processing
        # Size calculation for R^4: XM*YM*ZM + XM*YM + XM + 1
        # Implemented as a circular FIFO vector
        self.buffer_size = self._calculate_buffer_size()
        self.buffer = [0.0] * self.buffer_size

    def _calculate_buffer_size(self):
        # Example for 4D: X*Y*Z + X*Y + X + 1
        # Generalizing for n-dimensions is complex; the source specifies R^4 explicitly.
        # For implementation, one must allocate sufficient memory for the footprint.
        # This function returns the size for the specific dimensionality.
        if self.n == 4:
            return self.dimensions[0] * self.dimensions[1] * self.dimensions[2] + \
                   self.dimensions[0] * self.dimensions[1] + \
                   self.dimensions[0] + 1
        else:
            # Not given in the sources for other dimensions.
            return 0

    def _get_neighbor_value(self, coords):
        """
        Retrieves the value of a neighbor from the data or buffer.
        In an out-of-core implementation, this accesses the circular buffer.
        For simplicity, we assume direct access to the data array here,
        but the buffer management is critical for the method's memory efficiency.
        """
        # In a real implementation, this maps logical coordinates to buffer indices
        # handling the circular nature and dimension wrapping.
        return self.data[coords]

    def predict(self, coords):
        """
        Computes the Lorenzo prediction for the sample at coords.
        coords is a tuple (x, y, z, t) for 4D.
        """
        n = self.n
        prediction = 0.0
        
        # Iterate over all 2^n corners of the hypercube
        # The current sample is at (1, ..., 1) in local coords relative to the cube.
        # Neighbors are at u in {0, 1}^n excluding (1, ..., 1).
        
        # Generate all binary combinations for the neighbors
        for i in range(1, 1 << n): # 1 to 2^n - 1
            # Construct the neighbor coordinate relative to the current sample
            # If current sample is (x, y, z, t), neighbor u corresponds to
            # bits of i indicating whether to subtract 1 from the coordinate.
            # However, the source defines u in {0,1}^n local coords.
            # We need to map local u to global coordinates.
            
            # Local u is derived from the binary representation of i
            local_u = []
            temp_i = i
            for dim in range(n):
                local_u.append(temp_i & 1)
                temp_i >>= 1
            
            # Calculate g(u): number of zeros in local_u
            # Or equivalently n - sum(local_u)
            g_u = n - sum(local_u)
            
            # Weight is (-1)^g_u
            weight = 1 if g_u % 2 == 0 else -1
            
            # Map local_u to global coordinates
            # If local_u[dim] is 0, global coord is current - 1
            # If local_u[dim] is 1, global coord is current (but this is the sample itself, excluded)
            # Wait, the source says u in {0,1}^n \ {(1,...,1)}.
            # The sample v is at (1,...,1).
            # The neighbors are the other corners.
            # If local_u[dim] == 0, the neighbor is at coord - 1.
            # If local_u[dim] == 1, the neighbor is at coord.
            
            global_coords = list(coords)
            for dim in range(n):
                if local_u[dim] == 0:
                    global_coords[dim] -= 1
            
            # Retrieve value
            val = self._get_neighbor_value(tuple(global_coords))
            prediction += weight * val
            
        return prediction

    def compress(self):
        """
        Compresses the n-dimensional scalar field using the Lorenzo predictor.
        """
        # Recursive function to traverse the data in the correct order
        def lorenzo_recursive(d, coords):
            # coords is a list of current coordinates for dimensions 0 to d-1
            # d is the current dimension being processed (1-indexed in source logic, but let's adapt)
            # Source: Lorenzo(d, x1...xn). d is the dimension index.
            # If d > n, we are done.
            
            if d > self.n:
                return

            # Determine the range for the current dimension d
            # Source uses 1-based indexing for dimensions in the recursion description
            # Let's map to 0-based: dim_idx = d - 1
            dim_idx = d - 1
            dim_size = self.dimensions[dim_idx]
            
            # Check if all coordinates are -1 (initial state)
            # The source uses -1 as a sentinel for "not yet visited/initialized"
            # We need to handle the initialization of the scan order.
            
            # Simplified iterative approach for clarity, matching the recursive logic:
            # The recursion effectively performs a nested loop structure.
            pass

        # Implementing the recursive scan order explicitly
        # Lorenzo(d, x1...xn)
        # If all xi != -1: predict and encode
        # Else: recurse to lower dimension, then loop current dimension
        
        # We will use a stack or explicit recursion to mimic the source's Lorenzo function
        self._lorenzo_step(self.n, [-1] * self.n)

    def _lorenzo_step(self, d, coords):
        """
        Recursive step for the Lorenzo scan order.
        d: current dimension index (1 to n)
        coords: list of coordinates for dimensions 1 to n
        """
        if d == 0:
            return

        # Check if all coordinates from 1 to d are not -1
        # In the source, the check is "if all x1...xn differ from -1"
        # But the recursion only passes x1...xn.
        # The condition effectively checks if we are at a valid leaf node in the recursion tree
        # where all dimensions up to d have been initialized.
        
        # Actually, the source logic:
        # Lorenzo(d, x1...xn)
        # if all x1...xn differ from -1:
        #    predict and encode
        # else:
        #    Lorenzo(d-1, x1...0...xn)
        #    for i = 1 to dmax: Lorenzo(d, x1...i...xn)
        
        # This implies that if any coordinate is -1, we are not at a valid sample point yet.
        # We must initialize the lower dimensions.
        
        # Check if any coordinate in coords[0:d] is -1
        if any(c == -1 for c in coords[:d]):
            # Find the first -1 coordinate
            for i in range(d):
                if coords[i] == -1:
                    # Initialize this dimension to 0
                    coords[i] = 0
                    # Recurse to lower dimension? No, the source says Lorenzo(d-1, ...)
                    # But we are at dimension d. We need to fill x_d first?
                    # The source recursion: Lorenzo(d-1, x1...xd-1, 0, xd+1...xn)
                    # This sets xd to 0 and recurses on d-1.
                    
                    # Let's follow the source exactly:
                    # Lorenzo(d, x1...xn)
                    # else Lorenzo(d-1, x1...xd-1, 0, xd+1...xn)
                    
                    # So we set x_d = 0 and call Lorenzo(d-1, ...)
                    # But wait, the source passes x1...xn to Lorenzo(d-1, ...).
                    # This means the list of coordinates is fixed size n.
                    
                    # Let's adjust coords to have x_d = 0
                    coords[d-1] = 0 # d is 1-based index, so d-1 is 0-based
                    
                    # Call Lorenzo(d-1, ...)
                    self._lorenzo_step(d - 1, coords)
                    
                    # Then loop i from 1 to dmax for dimension d
                    # Note: the source says "for i = 1 to dmax do Lorenzo(d, x1...i...xn)"
                    # This implies we iterate through the current dimension d, keeping lower dims fixed?
                    # No, the lower dims are already initialized by the recursive call.
                    
                    # Actually, the structure is:
                    # To process dimension d, we first ensure dimensions 1..d-1 are initialized (via recursion)
                    # Then we iterate through dimension d.
                    
                    # But the source recursion is:
                    # Lorenzo(d, ...)
                    # if valid: encode
                    # else:
                    #    Lorenzo(d-1, ... with xd=0 ...)
                    #    for i=1 to dmax: Lorenzo(d, ... with xd=i ...)
                    
                    # This means if we are at dimension d, and not all coords are valid,
                    # we first initialize the lower dimensions (by setting xd=0 and recursing d-1)
                    # Then we iterate xd from 1 to dmax.
                    
                    # However, the "all x1...xn differ from -1" check is global.
                    # If we are at Lorenzo(4, -1, -1, -1, -1), it calls Lorenzo(3, -1, -1, -1, 0).
                    # Then Lorenzo(3, -1, -1, -1, 0) calls Lorenzo(2, -1, -1, 0, 0).
                    # Then Lorenzo(2, -1, -1, 0, 0) calls Lorenzo(1, -1, 0, 0, 0).
                    # Then Lorenzo(1, -1, 0, 0, 0) calls Lorenzo(0, 0, 0, 0, 0).
                    # Then Lorenzo(0, ...) returns.
                    # Then Lorenzo(1, ...) loops i=1 to dmax for x1.
                    
                    # So the logic is:
                    # If we are at dimension d, and x_d is -1, we set x_d=0 and recurse d-1.
                    # After returning, we loop x_d from 1 to dmax.
                    
                    # But what about x_{d+1}...x_n? They are also -1.
                    # The source says "Lorenzo(d-1, x1...xd-1, 0, xd+1...xn)".
                    # This sets x_d to 0 and recurses on d-1.
                    # The coordinates x_{d+1}...x_n remain -1.
                    # This seems to imply that the recursion goes down to dimension 1,
                    # initializing all dimensions to 0, then back up.
                    
                    # Let's re-read carefully:
                    # Lorenzo(d, x1...xn)
                    # if all x1...xn differ from -1:
                    #    encode
                    # else:
                    #    Lorenzo(d-1, x1...xd-1, 0, xd+1...xn)
                    #    for i = 1 to dmax: Lorenzo(d, x1...xd-1, i, xd+1...xn)
                    
                    # This means if ANY coordinate is -1, we don't encode.
                    # We first initialize the current dimension d to 0, and recurse to d-1.
                    # This will eventually initialize all dimensions to 0.
                    # Then, after the recursion returns, we loop through dimension d.
                    # But wait, if x_{d+1} is still -1, the recursive call Lorenzo(d, ...) inside the loop
                    # will again hit the "else" branch and recurse to d-1?
                    # No, because inside the loop, we call Lorenzo(d, ... with xd=i ...).
                    # If x_{d+1} is -1, then "all x1...xn differ from -1" is false.
                    # So it will recurse to d-1 again?
                    # This would lead to infinite recursion or redundant calls.
                    
                    # The key is that the recursion structure ensures that when we are at Lorenzo(d, ...),
                    # the dimensions d+1...n are already "processed" or "initialized" in the context of the caller?
                    # No, the caller is Lorenzo(d+1, ...).
                    
                    # Let's trace 2D: Lorenzo(2, -1, -1)
                    # Else: Lorenzo(1, -1, 0)
                    #    Else: Lorenzo(0, 0, 0) -> returns
                    #    Loop i=1 to Xmax: Lorenzo(1, i, 0)
                    #        If i=1: Lorenzo(1, 1, 0). All coords != -1? Yes (1, 0). Encode H[0,0].
                    #        If i=2: Lorenzo(1, 2, 0). Encode H[1,0].
                    # Loop i=1 to Ymax: Lorenzo(2, -1, i)
                    #    Else: Lorenzo(1, 0, i)
                    #        Else: Lorenzo(0, 0, i) -> returns
                    #        Loop j=1 to Xmax: Lorenzo(1, j, i)
                    #            Encode H[j-1, i-1]
                    
                    # This matches the trace in the source:
                    # Lorenzo(2,-1,-1); Lorenzo(1,-1,0); Lorenzo(0,0,0); Encodes H[0,0]
                    # Lorenzo(1,1,0); Encodes H[1,0]-H[0,0]
                    # Lorenzo(1,2,0); Encodes H[2,0]-H[1,0]
                    # Lorenzo(2,-1,1); Lorenzo(1,0,1); Encodes H[0,1]-H[0,0]
                    
                    # So the logic is:
                    # When calling Lorenzo(d, coords), if any coord is -1, we must initialize the lower dimensions.
                    # We set the current dimension d to 0, and recurse to d-1.
                    # This ensures that when we return to the loop for dimension d, the lower dimensions are initialized.
                    # But wait, in the 2D trace, Lorenzo(1, -1, 0) calls Lorenzo(0, 0, 0).
                    # Then it loops i=1 to Xmax for Lorenzo(1, i, 0).
                    # Then Lorenzo(2, -1, -1) loops i=1 to Ymax for Lorenzo(2, -1, i).
                    # Inside Lorenzo(2, -1, 1), it calls Lorenzo(1, 0, 1).
                    # Lorenzo(1, 0, 1) has all coords != -1? Yes (0, 1). Encode H[0,1].
                    # Then it loops j=1 to Xmax for Lorenzo(1, j, 1).
                    
                    # So the "else" branch handles the initialization of the current dimension d to 0,
                    # and recurses to d-1 to initialize d-1...1.
                    # Then the loop iterates d from 1 to dmax.
                    
                    # Implementation:
                    
                    # Set current dimension d to 0
                    coords[d-1] = 0
                    
                    # Recurse to d-1
                    self._lorenzo_step(d - 1, coords)
                    
                    # Loop i from 1 to dmax for dimension d
                    for i in range(1, self.dimensions[d-1] + 1):
                        coords[d-1] = i
                        self._lorenzo_step(d, coords)
        else:
            # All coordinates are valid
            # Predict and encode
            pred = self.predict(tuple(coords))
            actual = self._get_neighbor_value(tuple(coords))
            residual = actual - pred
            self.encoder.encode_residual(residual, pred)

## Subtopics

### [Lorenzo Predictor](10120-lorenzo-predictor.md)
The Lorenzo predictor is an isotropic predictive coding method for $n$-dimensional scalar fields that estimates a sample value from its $2^n - 1$ immediate neighbors in a regular grid to generate residuals for compression [S1, unofficial]. It is distinguished from anisotropic scanline predictors by exploiting data coherence in all dimensions and is the simplest isotropic predictor capable of exactly reconstructing all polynomials of degree less than $n$ [S1, unofficial].

### [SZ-L/R](11010-sz-l-r.md)
SZ-L/R is a block-based lossless/lossy compression algorithm for volumetric data that employs Lorenzo and linear regression predictors [S1, unofficial]. It partitions input data into independent 6×6×6 blocks, allowing for cache-friendly processing and random access without inter-block dependencies [S1, unofficial]. Unlike global interpolation methods such as [SZ-Interp](52-sz-interp.md), SZ-L/R prioritizes local feature preservation, making it suitable for unsmooth and irregular datasets, though it may introduce block-wise artifacts [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The method has a small memory footprint, making it suitable for out-of-core implementations. For a 4D dataset, the buffer size is $X_M \cdot Y_M \cdot Z_M + X_M \cdot Y_M + X_M + 1$ [S1, unofficial]. The buffer is implemented as a circular FIFO vector [S1, unofficial].
*   **Time:** The predictor is of the highest possible order among all predictors attempting to predict the value at one corner of a cube from the other $2^n - 1$ corners [S1, unofficial].

**Measured Performance:**
*   On a 3D model of a house on fire (43,202,395 bytes uncompressed), 4D Lorenzo compression reduced the size to 524,768 bytes (1.21% of total), outperforming 3D slice compression (1,076,587 bytes, 2.49%) by 50% [S1, unofficial].
*   On a fluid mixing simulation dataset, 3D time slice predictor produced 304,937,058 bytes (1.77 bits/symbol) and 4D Lorenzo produced 318,871,620 bytes (1.85 bits/symbol) [S1, unofficial].
*   Context arithmetic encoder gives a 25% gain over adaptive arithmetic encoder for encoding residuals [S1, unofficial].
*   In lossless compression, cubic wavelets are slightly better than the Lorenzo predictor in terms of residual entropy for 4D datasets [S1, unofficial].
*   In lossy L-infinity compression, the Lorenzo predictor performs better than SPIHT (a wavelet implementation using S+P transform) on the tested 2D dataset [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Isotropic prediction exploits data coherence in all dimensions. Exact for polynomials of degree less than $n$. Small memory footprint suitable for out-of-core processing.
*   **Limitations:** For data with higher degree polynomials, errors occur (3-8% for 3D predictor on cubic functions, <1% for 4D predictor on quartic functions) [S1, unofficial]. In lossless compression of 4D data, cubic wavelets may yield slightly better entropy [S1, unofficial].

## Background
The Lorenz Predictor was introduced by S. Mennon and M. Rizk in the paper "Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields" [S1, unofficial]. It extends a simple two-dimensional parallelogram predictor to higher dimensions. The method is designed for large $n$-dimensional scalar fields, particularly in scientific visualization and simulation data, where out-of-core processing is necessary due to the size of the data exceeding available memory [S1, unofficial].

## Sources
[S1] Out-of-Core Compression and Decompression of Large n-Dimensional Scalar Fields, https://digital.library.unt.edu/ark:/67531/metadc1404787/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
