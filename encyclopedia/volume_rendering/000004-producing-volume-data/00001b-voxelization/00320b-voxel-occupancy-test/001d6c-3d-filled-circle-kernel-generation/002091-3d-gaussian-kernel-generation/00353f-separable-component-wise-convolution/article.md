# Separable Component-Wise Convolution

Also called: Separable component-wise convolution on decomposed low-rank tensor

## Overview
Separable component-wise convolution is a computationally efficient method for applying 3D Gaussian smoothing to volumes represented as decomposed low-rank tensors [S1, unofficial]. It solves the problem of high computational cost and memory usage associated with naively convolving a 3D kernel on a reconstructed tensor volume [S1, unofficial]. Unlike traditional separated kernel convolution which only exploits kernel separability, this method distributes the convolution across individual tensor components by leveraging the separability of the input signal itself, thereby preserving the space compression advantage of the low-rank representation [S1, unofficial].

## How it works
**Inputs:**
- A decomposed low-rank tensor volume $T_\sigma$ (for density) or $T_c$ (for color), represented as a sum of $R$ rank-1 components.
- A 3D Gaussian kernel $N_{3d}$, defined as the outer product of 1D and 2D Gaussian kernels: $N_{3d} = N_{1d} \otimes N_{2d}$ [S1, unofficial].
- For color fields: Basis vectors $b_r$ associated with each component [S1, unofficial].

**Outputs:**
- The convoluted tensor volume $\tilde{T}_\sigma$ or $\tilde{T}_c$ [S1, unofficial].
- The final spectral restricted fields $F_\sigma$ and $\vec{F}_c$ derived from the convoluted tensors [S1, unofficial].

**Data Structures:**
The input tensor is stored in a decomposed low-rank format. For each rank $r \in \{1, \dots, R\}$, the tensor consists of:
- Vector components: $\tilde{v}_{\sigma,r}^X, \tilde{v}_{\sigma,r}^Y, \tilde{v}_{\sigma,r}^Z$ (and similarly for color $\tilde{v}_{c,r}$) [S1, unofficial].
- Matrix components: $\tilde{M}_{\sigma,r}^{Y,Z}, \tilde{M}_{\sigma,r}^{X,Z}, \tilde{M}_{\sigma,r}^{X,Y}$ (and similarly for color $\tilde{M}_{c,r}$) [S1, unofficial].
- Basis vectors: $b_r$ for color fields [S1, unofficial].

The dimensions of the volume are $I \times J \times K$ [S1, unofficial]. The kernel size in each dimension is $L_N$ [S1, unofficial].

**Steps:**

1. **Decompose the Kernel:** Recognize that the 3D Gaussian kernel $N_{3d}$ is separable into $N_{1d}$ and $N_{2d}$ [S1, unofficial].

2. **Compute 1D Convolutions on Vector Components:** For each rank $r$ and each axis ($X, Y, Z$), apply a 1D Gaussian convolution ($N_{1d}$) to the corresponding vector component.
   - Formula: $\tilde{v}_{\sigma,r} = (N_{1d} *_{1d} \tilde{v}_{\sigma,r})$ [S1, unofficial].
   - Note: The notation $\tilde{v}_{\sigma,r}$ in the result represents the convoluted vector. The specific axis ($X, Y, Z$) determines which dimension the 1D convolution operates on.

3. **Compute 2D Convolutions on Matrix Components:** For each rank $r$ and each pair of axes ($Y,Z$), ($X,Z$), ($X,Y$), apply a 2D Gaussian convolution ($N_{2d}$) to the corresponding matrix component. The 2D convolution is further separated into two sequential 1D convolutions for efficiency [S1, unofficial].
   - Formula: $\tilde{M}_{\sigma,r} = (N_{2d} *_{2d} \tilde{M}_{\sigma,r})$ [S1, unofficial].

4. **Reconstruct the Convoluted Tensor:** Combine the individually convoluted components using the identity from Theorem 4 [S1, unofficial].
   - For the density field $\tilde{T}_\sigma$:
     $$ \tilde{T}_\sigma = \sum_{r=1}^{R} \left( \tilde{v}_{\sigma,r}^X \otimes \tilde{M}_{\sigma,r}^{Y,Z} + \tilde{v}_{\sigma,r}^Y \otimes \tilde{M}_{\sigma,r}^{X,Z} + \tilde{v}_{\sigma,r}^Z \otimes \tilde{M}_{\sigma,r}^{X,Y} \right) $$
     where $\tilde{T}_\sigma = (N_{3d} *_{3d} T_\sigma)$ [S1, unofficial].
   - For the color field $\tilde{T}_c$:
     $$ \tilde{T}_c = \sum_{r=1}^{R} \left( \tilde{v}_{c,r}^X \otimes \tilde{M}_{c,r}^{Y,Z} \otimes b_r + \tilde{v}_{c,r}^Y \otimes \tilde{M}_{c,r}^{X,Z} \otimes b_r + \tilde{v}_{c,r}^Z \otimes \tilde{M}_{c,r}^{X,Y} \otimes b_r \right) $$
     [S1, unofficial].

5. **Sample the Final Fields:**
   - Density field: $F_\sigma(x) = \tilde{T}_\sigma(x)$ [S1, unofficial].
   - Color field: $\vec{F}_c(x, \vec{d}) = S(\tilde{T}_c(x), \vec{d})$, where $S$ is a spectral function and $\vec{d}$ is the viewing direction [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def separable_component_wise_convolution(tensor_components, kernel_1d, kernel_2d, basis_vectors=None):
    """
    Performs separable component-wise convolution on a decomposed low-rank tensor.
    
    Args:
        tensor_components: A list of R components. Each component is a dict with keys:
            'v_X': 1D array of size I (vector along X)
            'v_Y': 1D array of size J (vector along Y)
            'v_Z': 1D array of size K (vector along Z)
            'M_YZ': 2D array of size JxK (matrix for Y,Z axes)
            'M_XZ': 2D array of size IxK (matrix for X,Z axes)
            'M_XY': 2D array of size IxJ (matrix for X,Y axes)
        kernel_1d: 1D Gaussian kernel array of size L_N.
        kernel_2d: 2D Gaussian kernel array of size L_N x L_N.
        basis_vectors: Optional list of R basis vectors (for color fields).
        
    Returns:
        convoluted_components: A list of R components with convoluted vectors and matrices.
    """
    R = len(tensor_components)
    convoluted_components = []
    
    for r in range(R):
        comp = tensor_components[r]
        conv_comp = {}
        
        # Step 2: 1D Convolution on Vector Components
        # Apply 1D Gaussian convolution along the respective dimension
        conv_comp['v_X'] = convolve_1d(comp['v_X'], kernel_1d)
        conv_comp['v_Y'] = convolve_1d(comp['v_Y'], kernel_1d)
        conv_comp['v_Z'] = convolve_1d(comp['v_Z'], kernel_1d)
        
        # Step 3: 2D Convolution on Matrix Components
        # Apply 2D Gaussian convolution, separated into two 1D convolutions
        conv_comp['M_YZ'] = convolve_2d_separable(comp['M_YZ'], kernel_2d)
        conv_comp['M_XZ'] = convolve_2d_separable(comp['M_XZ'], kernel_2d)
        conv_comp['M_XY'] = convolve_2d_separable(comp['M_XY'], kernel_2d)
        
        convoluted_components.append(conv_comp)
        
    return convoluted_components

def convolve_1d(signal, kernel):
    """
    Performs 1D convolution of signal with kernel.
    Handles boundary conditions by padding with zeros or replicating edges.
    """
    # Implementation detail: Standard 1D convolution
    # Output size depends on padding strategy. Assuming 'same' padding for simplicity.
    # In practice, this is a dot product over the sliding window.
    padded_signal = pad_signal(signal, len(kernel) // 2)
    output = []
    for i in range(len(signal)):
        # Dot product of kernel and signal window
        val = sum(kernel[k] * padded_signal[i + k - len(kernel) // 2] for k in range(len(kernel)))
        output.append(val)
    return output

def convolve_2d_separable(matrix, kernel_2d):
    """
    Performs 2D convolution using separable 1D kernels.
    kernel_2d is assumed to be separable into k_x and k_y.
    If kernel_2d is not explicitly separated, it is decomposed.
    """
    # Decompose 2D kernel into two 1D kernels if not already done
    # For a Gaussian, k_x and k_y are identical 1D Gaussian profiles.
    k_x = extract_1d_from_2d(kernel_2d, axis=0)
    k_y = extract_1d_from_2d(kernel_2d, axis=1)
    
    # Convolve along first axis (rows)
    # Matrix shape: (H, W)
    # Convolve each row along W? No, convolve along columns (axis 1) first?
    # Standard separable: Convolve along rows, then along columns.
    
    # Convolve along axis 1 (columns) using k_y
    # Each row is a 1D signal
    temp_matrix = []
    for row in matrix:
        temp_matrix.append(convolve_1d(row, k_y))
    
    # Convolve along axis 0 (rows) using k_x
    # Transpose, convolve rows, transpose back
    temp_matrix_T = transpose(temp_matrix)
    convolved_rows = [convolve_1d(row, k_x) for row in temp_matrix_T]
    result = transpose(convolved_rows)
    
    return result

def pad_signal(signal, padding):
    """Pads signal with zeros."""
    return [0] * padding + signal + [0] * padding

def extract_1d_from_2d(kernel_2d, axis):
    """Extracts 1D kernel from 2D kernel. Assumes separability."""
    # For a Gaussian, any row or column is proportional to the 1D kernel.
    # Return the first row or column.
    if axis == 0:
        return kernel_2d[0]
    else:
        return [row[0] for row in kernel_2d]

def transpose(matrix):
    """Transposes a 2D matrix."""
    return [[matrix[j][i] for j in range(len(matrix))] for i in range(len(matrix[0]))]
```

**Edge Cases and Pitfalls:**
- **Naive Convolution Cost:** Naively convolving the 3D Gaussian kernel with the reconstructed 3D tensor volume destroys the space compression advantage. The time complexity becomes $O(I \cdot J \cdot K \cdot L_N^3)$ and space complexity $O(I \cdot J \cdot K)$ [S1, unofficial].
- **Traditional Separated Kernel:** Traditional techniques only separate the 3D kernel without utilizing input signal separability. This requires three sequential 1D convolutions on the full 3D volume, resulting in time complexity $O(I \cdot J \cdot K \cdot L_N)$ and space complexity $O(I \cdot J \cdot K)$ [S1, unofficial].
- **Precision:** The method relies on the correctness of Theorem 4, which allows the distribution of convolution across components. Any deviation from the low-rank structure or incorrect decomposition may lead to inaccuracies [S1, unofficial].

## Operating characteristics
**Time Complexity:**
- Computing convoluted components: $O(I \cdot J \cdot L_N + J \cdot K \cdot L_N + K \cdot I \cdot L_N)$, assuming 2D Gaussian convolution on matrix components is separated into 1D Gaussian convolutions [S1, unofficial].
- Each query sample: $O(R)$, same as the original decomposed tensor [S1, unofficial].

**Space Complexity:**
- The method preserves the space compression advantage of the decomposed low-rank tensor. It does not require storing the full 3D volume during convolution [S1, unofficial].

**Strengths:**
- Drastically reduces computation required for filtering 3D radiance fields compared to naive or traditional separated kernel methods [S1, unofficial].
- Maintains the efficiency of low-rank tensor representations [S1, unofficial].

**Limitations:**
- Requires the tensor to be in a decomposed low-rank format [S1, unofficial].
- Relies on the separability of the Gaussian kernel and the specific structure of the tensor decomposition [S1, unofficial].

**When to Use:**
- When working with decomposed low-rank tensor volumes and needing to apply 3D Gaussian smoothing efficiently [S1, unofficial].
- When memory and computational resources are constrained, and the low-rank structure can be maintained [S1, unofficial].

**When Not to Use:**
- When the volume is not in a decomposed low-rank format [S1, unofficial].
- When high precision is required and the low-rank approximation introduces significant error [S1, unofficial].

## Background
The method was proposed to improve the robustness of joint optimization of camera poses and decomposed low-rank tensorial radiance fields [S1, unofficial]. It addresses the issue that naively convolving the 3D Gaussian kernel with the 3D volume destroys the space compression advantage of the decomposed low-rank tensor [S1, unofficial]. The technique is based on Theorem 4, which proves that the 3D convoluted tensor can be expressed as the composition of individually convoluted components [S1, unofficial]. This allows the distribution of the 3D Gaussian convolution across the individual components of the decomposed low-rank tensor [S1, unofficial]. The method is part of a coarse-to-fine training schedule to limit the spectrum of the radiance field [S1, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
