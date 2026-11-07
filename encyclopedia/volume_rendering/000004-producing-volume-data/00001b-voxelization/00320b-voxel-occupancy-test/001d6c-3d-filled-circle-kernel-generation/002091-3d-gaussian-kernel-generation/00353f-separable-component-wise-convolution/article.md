# Separable Component-Wise Convolution

Also called: Separable component-wise convolution on decomposed low-rank tensor

## Overview
Separable component-wise convolution is a computationally efficient method for applying 3D Gaussian smoothing to volumes represented as decomposed low-rank tensors. It solves the problem of high computational cost and memory usage associated with naively convolving a 3D kernel on a reconstructed full tensor volume. Unlike traditional separated kernel convolution which only exploits kernel separability, this method distributes the convolution across individual vector and matrix components of the decomposed tensor, leveraging the input signal's inherent separability to maintain space compression and reduce time complexity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A decomposed low-rank tensor volume $T$ of rank $R$, with dimensions $I \times J \times K$. The tensor is decomposed into $R$ components, each consisting of vector components ($\tilde{v}$) and matrix components ($\tilde{M}$) along different axes.
*   **Kernel:** A 3D Gaussian kernel $N_{3d}$, defined as the outer product of a 1D Gaussian kernel $N_{1d}$ and a 2D Gaussian kernel $N_{2d}$ ($N_{3d} = N_{1d} \otimes N_{2d}$).
*   **Output:** The convoluted tensor volume $\tilde{T}_\sigma$ (for density) or $\tilde{T}_c$ (for color), represented in the same decomposed low-rank format.

**Data Structures:**
The input tensor is stored as a list of $R$ components. For each component $r \in \{1, \dots, R\}$:
*   $\tilde{v}_{\sigma,r}^X$: A vector of size $I$ (or $J, K$ depending on axis).
*   $\tilde{M}_{\sigma,r}^{Y,Z}$: A matrix of size $J \times K$ (or other axis pairs).
*   $b_r$: A basis vector (for color fields).

**Numbered Steps:**
1.  **Decompose:** Ensure the input volume $T_\sigma$ is represented in its decomposed low-rank form with $R$ components.
2.  **Component-wise 1D Convolution:** For each vector component $\tilde{v}_{\sigma,r}$ along a specific axis (e.g., X), apply a 1D Gaussian convolution with kernel $N_{1d}$.
    $$ \tilde{v}_{\sigma,r} = (N_{1d} *_{1d} \tilde{v}_{\sigma,r}) $$
3.  **Component-wise 2D Convolution:** For each matrix component $\tilde{M}_{\sigma,r}^{Y,Z}$, apply a 2D Gaussian convolution with kernel $N_{2d}$. This is typically implemented as two sequential 1D convolutions (one along Y, one along Z).
    $$ \tilde{M}_{\sigma,r} = (N_{2d} *_{2d} \tilde{M}_{\sigma,r}) $$
4.  **Reconstruct:** Combine the individually convoluted components using the identity from Theorem 4 to form the final convoluted tensor $\tilde{T}_\sigma$.
    $$ \tilde{T}_\sigma = \sum_{r=1}^{R} \tilde{v}_{\sigma,r}^X \otimes \tilde{M}_{\sigma,r}^{Y,Z} + \tilde{v}_{\sigma,r}^Y \otimes \tilde{M}_{\sigma,r}^{X,Z} + \tilde{v}_{\sigma,r}^Z \otimes \tilde{M}_{\sigma,r}^{X,Y} $$
5.  **Color Field Processing (Optional):** For color fields, combine the convoluted components with basis vectors $b_r$ to form $\tilde{T}_c$, then apply the spectral function $S$ to obtain the spectral restricted color field $\vec{F}_c(x, \vec{d}) = S(\tilde{T}_c(x), \vec{d})$.
    $$ \tilde{T}_c = \sum_{r=1}^{R} \tilde{v}_{c,r}^X \otimes \tilde{M}_{c,r}^{Y,Z} \otimes b_r + \tilde{v}_{c,r}^Y \otimes \tilde{M}_{c,r}^{X,Z} \otimes b_r + \tilde{v}_{c,r}^Z \otimes \tilde{M}_{c,r}^{X,Y} \otimes b_r $$

**Formulas:**
*   **Naive 3D Convolution (for comparison):**
    $$ F_\sigma(x,y,z) = (N_{3d} *_{3d} T_\sigma)(x,y,z) $$
    Where $*_{3d}$ denotes 3D convolution.
*   **Separable Component-Wise Convolution Identity (Theorem 4):**
    $$ \tilde{T}_\sigma = \sum_{r=1}^{R} \tilde{v}_{\sigma,r}^X \otimes \tilde{M}_{\sigma,r}^{Y,Z} + \tilde{v}_{\sigma,r}^Y \otimes \tilde{M}_{\sigma,r}^{X,Z} + \tilde{v}_{\sigma,r}^Z \otimes \tilde{M}_{\sigma,r}^{X,Y} $$
    Where:
    *   $\tilde{T}_\sigma = (N_{3d} *_{3d} T_\sigma)$ is the convoluted tensor.
    *   $\tilde{v}_{\sigma,r} = (N_{1d} *_{1d} \tilde{v}_{\sigma,r})$ is the 1D Gaussian convoluted vector component.
    *   $\tilde{M}_{\sigma,r} = (N_{2d} *_{2d} \tilde{M}_{\sigma,r})$ is the 2D Gaussian convoluted matrix component.

**Parameters:**
*   $R$: Rank of the decomposed tensor. Typical values depend on the desired accuracy vs. compression trade-off. Not given in the sources.
*   $L_N$: Size of the 1D Gaussian kernel in each dimension. Not given in the sources.
*   $I, J, K$: Dimensions of the volume along X, Y, Z axes.

**Edge Cases and Pitfalls:**
*   **Naive Implementation:** Naively convolving the 3D Gaussian kernel with the 3D volume requires reconstructing the whole 3D tensor before applying convolution, destroying the space compression advantage of the decomposed low-rank tensor [S1, unofficial].
*   **Traditional Separated Kernel:** Traditional techniques only separate the 3D kernel without utilizing the input signal's separability, requiring three sequential 1D convolutions on the full 3D volume. This results in higher time complexity $O(I \cdot J \cdot K \cdot L_N)$ and space complexity $O(I \cdot J \cdot K)$ to store intermediate results [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def separable_component_wise_convolution(tensor_decomposition, kernel_1d, kernel_2d):
    """
    Performs separable component-wise convolution on a decomposed low-rank tensor.
    
    Args:
        tensor_decomposition: List of R components. Each component is a dict with keys:
            'v_X': 1D array of size I
            'v_Y': 1D array of size J
            'v_Z': 1D array of size K
            'M_YZ': 2D array of size J x K
            'M_XZ': 2D array of size I x K
            'M_XY': 2D array of size I x J
        kernel_1d: 1D Gaussian kernel array.
        kernel_2d: 2D Gaussian kernel array.
        
    Returns:
        convoluted_decomposition: List of R components with convoluted vectors and matrices.
    """
    R = len(tensor_decomposition)
    convoluted_decomposition = []
    
    for r in range(R):
        comp = tensor_decomposition[r]
        
        # Step 2: Apply 1D Gaussian convolution to vector components
        # Assuming a standard 1D convolution function that handles boundary conditions
        v_X_conv = convolve_1d(comp['v_X'], kernel_1d)
        v_Y_conv = convolve_1d(comp['v_Y'], kernel_1d)
        v_Z_conv = convolve_1d(comp['v_Z'], kernel_1d)
        
        # Step 3: Apply 2D Gaussian convolution to matrix components
        # 2D convolution is separated into two 1D convolutions
        # First along Y, then along Z (or vice versa)
        M_YZ_conv = convolve_2d_separable(comp['M_YZ'], kernel_1d, kernel_1d)
        M_XZ_conv = convolve_2d_separable(comp['M_XZ'], kernel_1d, kernel_1d)
        M_XY_conv = convolve_2d_separable(comp['M_XY'], kernel_1d, kernel_1d)
        
        convoluted_decomposition.append({
            'v_X': v_X_conv,
            'v_Y': v_Y_conv,
            'v_Z': v_Z_conv,
            'M_YZ': M_YZ_conv,
            'M_XZ': M_XZ_conv,
            'M_XY': M_XY_conv
        })
        
    return convoluted_decomposition

def convolve_1d(signal, kernel):
    """
    Performs 1D convolution of signal with kernel.
    Returns the convolved signal.
    """
    # Implementation details for 1D convolution (e.g., using FFT or direct summation)
    # This is a standard operation, assumed to be available.
    # For direct implementation:
    n = len(signal)
    k_len = len(kernel)
    result = [0.0] * n
    for i in range(n):
        for j in range(k_len):
            idx = i + j - k_len // 2
            if 0 <= idx < n:
                result[i] += signal[idx] * kernel[j]
    return result

def convolve_2d_separable(matrix, kernel_1d_axis1, kernel_1d_axis2):
    """
    Performs 2D convolution on matrix using two 1D kernels.
    """
    # Convolve along axis 1 (rows)
    rows, cols = matrix.shape
    temp_matrix = [[0.0] * cols for _ in range(rows)]
    for i in range(rows):
        temp_matrix[i] = convolve_1d(matrix[i], kernel_1d_axis1)
        
    # Convolve along axis 2 (columns)
    result_matrix = [[0.0] * cols for _ in range(rows)]
    for j in range(cols):
        col = [temp_matrix[i][j] for i in range(rows)]
        convolved_col = convolve_1d(col, kernel_1d_axis2)
        for i in range(rows):
            result_matrix[i][j] = convolved_col[i]
            
    return result_matrix
```

## Subtopics

### [Dr. TVAM Convolution](8425-dr-tvam-convolution.md)
A separable 3D convolution method using Gaussian diffusion kernels, implemented via Dr.Jit's `dr.convolve` function for tomographic volumetric additive manufacturing simulations. It sets itself apart from non-separable 3D convolutions by leveraging axis-aligned separability to reduce computational complexity, allowing for interactive rates on CUDA hardware. It builds on [Separable Component-Wise Convolution](7446-separable-component-wise-convolution.md) by applying specific Gaussian diffusion kernels rather than generic low-rank tensor decompositions.

## Operating characteristics
*   **Time Complexity:** The time complexity for computing the convoluted components is $O(I \cdot J \cdot L_N + J \cdot K \cdot L_N + K \cdot I \cdot L_N)$, assuming the 2D Gaussian convolution on matrix components is separated into 1D Gaussian convolutions. The time complexity for each query sample is $O(R)$ [S1, unofficial].
*   **Space Complexity:** The space complexity is maintained at the level of the decomposed tensor, avoiding the $O(I \cdot J \cdot K)$ space required to store the full 3D volume during convolution [S1, unofficial].
*   **Comparison with Naive Approach:** Naive 3D convolution has a time complexity of $O(I \cdot J \cdot K \cdot L_N^3)$ and space complexity of $O(I \cdot J \cdot K)$ [S1, unofficial].
*   **Comparison with Traditional Separated Kernel:** Traditional separated kernel convolution has a time complexity of $O(I \cdot J \cdot K \cdot L_N)$ and space complexity of $O(I \cdot J \cdot K)$ [S1, unofficial].
*   **Strengths:** Drastically reduces computation required for filtering 3D radiance fields while maintaining the space compression advantage of decomposed low-rank tensors [S1, unofficial].
*   **Limitations:** Requires the input volume to be in a decomposed low-rank tensor format. The method relies on the specific identity (Theorem 4) which may not apply to all tensor decomposition schemes.

## Background
The method was proposed in the context of improving robustness for joint optimization of camera poses and decomposed low-rank tensorial radiance fields. It addresses the issue that naively convolving a 3D Gaussian kernel with a 3D volume destroys the space compression advantage of decomposed low-rank tensors. The technique leverages an identity (Theorem 4) to distribute the 3D Gaussian convolution across individual components of the decomposed tensor. This allows for efficient spectrum limiting of the radiance field with a coarse-to-fine training schedule [S1, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
