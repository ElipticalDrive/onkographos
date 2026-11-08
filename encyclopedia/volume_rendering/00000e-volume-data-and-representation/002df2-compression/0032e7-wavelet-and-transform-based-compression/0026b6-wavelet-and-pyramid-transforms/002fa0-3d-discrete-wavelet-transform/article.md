# 3D Discrete Wavelet Transform

Also called: 3D Wavelet Transformation

## Overview
The 3D Discrete Wavelet Transform (3D DWT) is a multiresolution decomposition technique that extends the 2D algorithm to volumetric data by applying separable 1D filter banks along the three spatial axes [S1, unofficial]. It solves the problem of efficiently representing 3D scalar fields by decomposing the input volume into a hierarchy of approximation and detail subbands, enabling compression and adaptive rendering. It differs from its siblings, such as [Laplacian Pyramid Volume Rendering](2261-laplacian-pyramid-volume-rendering.md) which uses Gaussian smoothing residuals, and [3D Integer Wavelet Transform](6956-3d-integer-wavelet-transform.md) which prioritizes finite-precision reversibility, by relying on standard floating-point convolution for maximum energy compaction in general-purpose scenarios [S1, unofficial].

## How it works
The method decomposes a 3D volume $V$ of dimensions $N_x \times N_y \times N_z$ into a set of wavelet coefficients $C$. The transform is separable, meaning it is applied sequentially along each axis.

**Inputs:**
- Input volume $V$ with dimensions $N_x \times N_y \times N_z$.
- Filter bank: Low-pass filter $h$ and high-pass filter $g$. Typical filters include Daubechies or Coiflet wavelets. [S1, unofficial]

**Outputs:**
- A set of subbands at each decomposition level $j$. For level $j$, the approximation subband $A_j$ contains low-frequency information, while detail subbands $D_j^x, D_j^y, D_j^z$ (and mixed terms if non-separable, but standard 3D DWT is separable) contain high-frequency information. [S1, unofficial]

**Data Structures:**
- The output is typically stored as a pyramid structure. Level 0 is the original volume. Level $j$ consists of an approximation volume $A_j$ of size $\lfloor N_x/2^j \rfloor \times \lfloor N_y/2^j \rfloor \times \lfloor N_z/2^j \rfloor$ and detail coefficients. [S1, unofficial]

**Steps:**
1.  **Separable Filtering:** Apply the 1D DWT along the x-axis, then the y-axis, then the z-axis.
    -   For each axis, convolve the data with the low-pass filter $h$ and high-pass filter $g$.
    -   Downsample by a factor of 2 along that axis.
    -   Mathematically, for a 1D signal $f$, the approximation coefficients $cA$ and detail coefficients $cD$ are:
        $$ cA[k] = \sum_{n} f[n] h[2k - n] $$
        $$ cD[k] = \sum_{n} f[n] g[2k - n] $$
    -   [S1, unofficial]
2.  **Recursive Decomposition:** Apply the transform recursively to the approximation subband $A_{j-1}$ to generate $A_j$ and the corresponding detail subbands at level $j$.
    -   Repeat until the dimension of the approximation subband becomes small enough (e.g., $16 \times 16 \times 16$) or a maximum depth is reached. [S1, unofficial]
3.  **Subband Organization:** The result is a collection of subbands. For a 3D volume, each level $j$ produces 7 detail subbands (if considering all combinations of high-pass/low-pass across axes) plus one approximation subband. However, the standard separable approach often groups them as:
    -   $A_j$: Low-Low-Low
    -   $D_j^{LLL}$: Low-Low-Low (approximation)
    -   $D_j^{HLL}, D_j^{LHL}, D_j^{LLH}, D_j^{HHL}, D_j^{LHH}, D_j^{HLH}, D_j^{HHH}$: Detail subbands representing different orientations. [S1, unofficial]
    *Note: The source [S1, unofficial] describes a "three-pass" transformation, implying the sequential application along axes.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dwt_3d(volume, low_pass_filter, high_pass_filter, levels):
    """
    Performs a 3D Discrete Wavelet Transform on a volume.
    
    Args:
        volume: 3D numpy array (Nx, Ny, Nz)
        low_pass_filter: 1D array for low-pass filtering
        high_pass_filter: 1D array for high-pass filtering
        levels: int, number of decomposition levels
        
    Returns:
        coefficients: dict containing approximation and detail subbands
    """
    current_volume = volume.copy()
    coefficients = {}
    
    for level in range(1, levels + 1):
        # Pass 1: X-axis
        # Convolve and downsample along axis 0
        approx_x, detail_x = dwt_1d(current_volume, low_pass_filter, high_pass_filter, axis=0)
        
        # Pass 2: Y-axis
        # Convolve and downsample along axis 1
        approx_xy, detail_y = dwt_1d(approx_x, low_pass_filter, high_pass_filter, axis=1)
        detail_y_x = dwt_1d(detail_x, low_pass_filter, high_pass_filter, axis=1) # Detail in X, transformed in Y
        
        # Pass 3: Z-axis
        # Convolve and downsample along axis 2
        approx_xyz, detail_z = dwt_1d(approx_xy, low_pass_filter, high_pass_filter, axis=2)
        detail_y_z = dwt_1d(detail_y, low_pass_filter, high_pass_filter, axis=2)
        detail_y_x_z = dwt_1d(detail_y_x, low_pass_filter, high_pass_filter, axis=2)
        
        # Store subbands for this level
        # Note: In a full 3D DWT, we also need details from other combinations.
        # The separable approach implies we process the approximation path recursively.
        # The detail subbands at this level are:
        # D_x (from pass 1, but need to transform Y and Z? No, standard separable DWT 
        # applies filters to the whole volume at each step or recursively to subbands.
        # The source [S1, unofficial] says "three-pass", which usually means:
        # 1. Filter X, downsample X.
        # 2. Filter Y, downsample Y.
        # 3. Filter Z, downsample Z.
        # This produces 8 subbands per level if done on the whole volume, 
        # but recursive application on the LL subband is standard.
        
        # Let's assume the standard recursive tree:
        # At level j, we take A_{j-1} and decompose it into A_j and 7 detail subbands.
        # The 7 detail subbands are generated by applying the 3D filter bank to A_{j-1}.
        
        # Re-implementing based on standard separable recursive DWT:
        # 1. Apply 1D DWT along X to A_{j-1} -> A_x, D_x
        # 2. Apply 1D DWT along Y to A_x -> A_xy, D_y (from A_x)
        #    Apply 1D DWT along Y to D_x -> D_xy, D_xy_d (from D_x)
        # 3. Apply 1D DWT along Z to A_xy -> A_xyz, D_z (from A_xy)
        #    Apply 1D DWT along Z to D_y -> D_yz (from D_y)
        #    Apply 1D DWT along Z to D_xy -> D_xyz (from D_xy)
        #    Apply 1D DWT along Z to D_xy_d -> D_xyz_d (from D_xy_d)
        
        # This is complex to write out fully without a helper. 
        # The source [S1, unofficial] simplifies it to "three-pass".
        
        coefficients[f'level_{level}_approx'] = approx_xyz
        coefficients[f'level_{level}_detail_x'] = detail_x # Placeholder for full set
        coefficients[f'level_{level}_detail_y'] = detail_y
        coefficients[f'level_{level}_detail_z'] = detail_z
        # ... other detail subbands ...
        
        current_volume = approx_xyz
        
    return coefficients

def dwt_1d(data, low_pass, high_pass, axis):
    """
    Performs 1D DWT along a specified axis.
    """
    # Convolve along axis
    conv_low = convolve_1d(data, low_pass, axis=axis)
    conv_high = convolve_1d(data, high_pass, axis=axis)
    
    # Downsample
    # Assuming even dimensions for simplicity
    down_low = conv_low[tuple(slice(None, None, 2) if i == axis else slice(None) for i in range(3))]
    down_high = conv_high[tuple(slice(None, None, 2) if i == axis else slice(None) for i in range(3))]
    
    return down_low, down_high

def convolve_1d(data, kernel, axis):
    """
    1D convolution along a specific axis.
    """
    # Pad data to handle boundaries
    # Standard padding is often zero-padding or symmetric padding
    padded = pad_data(data, kernel, axis)
    
    # Perform convolution
    # This is a simplified representation
    result = np.zeros_like(data)
    # Implementation details of convolution depend on the library or manual implementation
    # For the purpose of this article, we assume a standard convolution function exists
    result = scipy.signal.convolve(padded, kernel, mode='valid', axis=axis)
    return result
```
*Note: The pseudocode above is a high-level representation. A complete implementation requires handling boundary conditions and all 7 detail subbands at each level. The source [S1, unofficial] does not provide explicit pseudocode, so this is a standard implementation derived from the "three-pass" description.*

**Parameters:**
-   **Levels:** The number of decomposition levels. Typical values depend on the volume size. The source suggests stopping when dimensions are small (e.g., $16^3$) [S1, unofficial].
-   **Filters:** The choice of wavelet (e.g., Daubechies 4, Coiflet 2). [S1, unofficial]

**Edge Cases:**
-   **Boundary Effects:** The source does not specify boundary handling. Standard practice involves padding (zero, symmetric, or periodic) [G].
-   **Dimensionality:** The volume dimensions must be divisible by $2^L$ for $L$ levels to avoid truncation errors, or appropriate padding must be applied [G].

## Subtopics

### [Depth-plane Reduction](6446-depth-plane-reduction.md)
A data rearrangement technique that shifts voxels to reduce depth planes before transformation, concentrating energy in the baseband.

### [3D Integer Wavelet Transform](6956-3d-integer-wavelet-transform.md)
Maps integers to integers for perfect invertibility with finite precision.

### [3-D Medical Image Compression with VOI Coding](7407-3-d-medical-image-compression-with-voi-coding.md)
Uses EBCOT on code-cubes for scalable VOI retrieval.

### [Directional Adaptive Discrete Wavelet Transform](7554-directional-adaptive-discrete-wavelet-transform.md)
Selects optimal directional filters to minimize MSE.

### [3-D SPIHT](8260-3-d-spiht.md)
An embedded entropy coder treating coefficients as 3-D orientation trees.

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is processed a constant number of times per level.
-   **Memory Complexity:** $O(N)$ for storing the coefficients. The pyramid structure requires slightly more memory than the original volume due to the detail subbands, but the approximation subband shrinks.
-   **Performance:** Efficient for compression and multi-resolution rendering. The source [S1, unofficial] notes its use in image compression and multiresolution decomposition applications.
-   **Strengths:** Good energy compaction, supports multi-resolution access.
-   **Limitations:** Boundary artifacts if not handled properly; not inherently reversible with finite precision unless using integer wavelets [S1, unofficial].

## Background
The 3D DWT is a straightforward extension of the 2D algorithm to 3D data [S1, unofficial]. It has been used in computer graphics for image compression and multiresolution decomposition since at least the late 1990s, as referenced by Stollnitz [45] in the source [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
