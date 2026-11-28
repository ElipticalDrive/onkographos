# Colorspace Transformations for Compression

Also called: Colorspace Optimization for Wavelet Compression, PCA colorspace, RGB to optimized colorspace, RGB to optimized colorspace transform, YIQ colorspace

## Overview
This method transforms RGB volume data into an optimized colorspace to decorrelate channels and assign significance weights for encoding and quantization [S1, unofficial]. It sets itself apart from sibling methods by explicitly leveraging perceptual or statistical channel importance to prioritize wavelet coefficients during compression, rather than treating all color channels equally [S1, unofficial].

## How it works
**Inputs:** RGB volume data (vectors $\mathbf{R}, \mathbf{G}, \mathbf{B}$).
**Outputs:** Transformed volume data in an optimized colorspace with associated significance weights.

**Data Structures:**
The optimized colorspace is spanned by three basis vectors $\mathbf{C}_1, \mathbf{C}_2, \mathbf{C}_3$. Each coefficient in the wavelet domain is represented by coordinates in this space.

**Steps:**
1.  **Transform RGB to Optimized Colorspace:** Convert the input RGB volume data into the space spanned by $\mathbf{C}_1, \mathbf{C}_2, \mathbf{C}_3$ [S1, unofficial].
2.  **Assign Significance Weights:** For each wavelet coefficient, calculate two independent significance weights:
    *   One based on the energy of the associated wavelet function [S1, unofficial].
    *   One based on the global significance of the colorspace coordinates [S1, unofficial].
3.  **Calculate Significance Score:** Compute the significance score $S$ for each coefficient using the formula:
    $$ S(w, \mathbf{C}) = E(w(\mathbf{C})) \cdot I(\mathbf{C}) $$
    Where:
    *   $w$ is the wavelet coefficient.
    *   $\mathbf{C}$ represents the colorspace coordinates.
    *   $E(w(\mathbf{C}))$ is the wavelet energy [S1, unofficial].
    *   $I(\mathbf{C})$ is the color channel importance [S1, unofficial].
4.  **Encode/Quantize:** Use the significance score to order coefficients for encoding and determine quantization precision [S1, unofficial].

**Parameters and Specific Colorspaces:**
*   **YIQ Colorspace:** A data-independent transform. $Y$ encodes luminance; $I$ and $Q$ encode chromaticity. Importance factors based on NTSC bandwidth conventions are:
    *   $Y$: 4
    *   $I$: 1.5
    *   $Q$: 0.6 [S1, unofficial]
*   **PCA/Karhunen-Loève Expansion:** A data-dependent transform. The transformation matrix is computed individually for each dataset. The significance of each eigenvector is described by the absolute values of the corresponding eigenvalues [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Intensity Volumes:** The transformation can be skipped for intensity volumes such as raw CT or MRI data sets [S1, unofficial].
*   **Computational Cost:** PCA computation is computationally more challenging for offline forward compression but provides better results [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def colorspace_transform(rgb_volume, method='YIQ'):
    """
    Transforms RGB volume data into an optimized colorspace.
    
    Args:
        rgb_volume: 3D array of RGB values.
        method: 'YIQ' or 'PCA'.
        
    Returns:
        transformed_volume: 3D array in optimized colorspace.
        importance_factors: Dictionary mapping channel names to importance weights.
    """
    if method == 'YIQ':
        # Simple matrix transform to YIQ
        # Y = 0.299*R + 0.587*G + 0.114*B
        # I = 0.596*R - 0.274*G - 0.322*B
        # Q = 0.211*R - 0.523*G + 0.312*B
        # Note: Exact matrix coefficients for YIQ may vary; standard NTSC is used here.
        y = 0.299 * rgb_volume[:,:,0] + 0.587 * rgb_volume[:,:,1] + 0.114 * rgb_volume[:,:,2]
        i = 0.596 * rgb_volume[:,:,0] - 0.274 * rgb_volume[:,:,1] - 0.322 * rgb_volume[:,:,2]
        q = 0.211 * rgb_volume[:,:,0] - 0.523 * rgb_volume[:,:,1] + 0.312 * rgb_volume[:,:,2]
        
        transformed_volume = np.stack((y, i, q), axis=-1)
        importance_factors = {'Y': 4, 'I': 1.5, 'Q': 0.6}
        
    elif method == 'PCA':
        # Compute PCA matrix for the dataset
        # Flatten volume to 2D array (pixels x channels)
        data = rgb_volume.reshape(-1, 3)
        mean = np.mean(data, axis=0)
        centered_data = data - mean
        cov_matrix = np.cov(centered_data, rowvar=False)
        
        # Eigen decomposition
        eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
        
        # Sort eigenvalues and eigenvectors in descending order
        sorted_indices = np.argsort(eigenvalues)[::-1]
        eigenvalues = eigenvalues[sorted_indices]
        eigenvectors = eigenvectors[:, sorted_indices]
        
        # Transform data
        transformed_data = np.dot(centered_data, eigenvectors)
        transformed_volume = transformed_data.reshape(rgb_volume.shape)
        
        # Importance factors are absolute values of eigenvalues
        importance_factors = {
            'C1': abs(eigenvalues[0]),
            'C2': abs(eigenvalues[1]),
            'C3': abs(eigenvalues[2])
        }
        
    else:
        raise ValueError("Unsupported method")
        
    return transformed_volume, importance_factors

def calculate_significance_score(wavelet_energy, importance_factor):
    """
    Calculates the significance score for a coefficient.
    
    Args:
        wavelet_energy: Energy of the wavelet coefficient.
        importance_factor: Importance weight of the color channel.
        
    Returns:
        significance_score: Product of energy and importance.
    """
    return wavelet_energy * importance_factor
```

## Subtopics

### YIQ Colorspace
A data-independent transform using luminance/chrominance separation. Choose when computational simplicity is preferred and NTSC-like progressive refinement is desired. [Link](2090-quantization-with-noise-injection.md)

### PCA Colorspace
A data-dependent transform using statistical optimization. Choose when better compression results are needed and offline computation cost is acceptable. [Link](5722-thresholding-and-quantization.md)

## Operating characteristics
*   **Time Complexity:** YIQ transform is $O(N)$ where $N$ is the number of voxels. PCA transform is $O(N^3)$ for covariance matrix computation (or $O(N^2)$ if optimized) plus $O(N)$ for transformation, making it significantly more expensive for large datasets [S1, unofficial].
*   **Memory Complexity:** Minimal overhead for YIQ (same size as RGB). PCA requires storing the transformation matrix (9 floats for 3x3) and potentially intermediate data [S1, unofficial].
*   **Performance:** YIQ allows progressive refinement in both spatial and color domains [S1, unofficial]. PCA provides better compression results due to optimal decorrelation [S1, unofficial].
*   **Strengths:** Improves compression efficiency by prioritizing perceptually or statistically important information. Enables progressive transmission [S1, unofficial].
*   **Limitations:** PCA is computationally intensive for offline compression [S1, unofficial]. Not applicable to intensity volumes [S1, unofficial].

## Background
Originated in the context of compression domain volume rendering for distributed systems [S1, unofficial]. The method was developed to balance CPU, network, and graphics performance, allowing scalability and low client storage requirements (e.g., less than 5KB for mother-wavelet splats and Huffman tables) [S1, unofficial]. It builds on wavelet compression techniques and significance encoding [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering for Distributed ..., https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
