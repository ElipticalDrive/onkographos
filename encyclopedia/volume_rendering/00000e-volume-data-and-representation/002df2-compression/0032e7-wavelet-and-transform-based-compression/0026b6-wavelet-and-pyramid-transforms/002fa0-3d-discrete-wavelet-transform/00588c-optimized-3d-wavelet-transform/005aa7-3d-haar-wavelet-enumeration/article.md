# 3D Haar Wavelet Enumeration

Also called: 3D Haar Wavelet Decomposition for Virtual Sculpture, 3D wavelet enumeration, Discrete multiresolution model based on 3D wavelets

## Overview
3D Haar Wavelet Enumeration is a multiresolution representation method that encodes 3D objects as a discrete set of voxels using 3D Haar wavelet decomposition to enable accelerated display and virtual sculpture operations [S1, unofficial]. It solves the problem of high processing and display times associated with uniform spatial enumerations by allowing the extraction of specific levels of detail (LOD) from a most detailed level down to coarser resolutions [S1, unofficial]. Unlike bounding box optimization methods (AABB/OBB) which are expensive in processing time for overlap reduction, this method prioritizes speed through uniform spatial enumeration or octree structures [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D discrete object defined by a uniform spatial enumeration (voxel grid). Each voxel contains a density value coded on one byte (0 for empty, 255 for full) [S1, unofficial]. The input sequence length $p$ in each dimension must be a power of two (e.g., $p=128=2^7$) [S1, unofficial].
*   **Output:** A hierarchical structure storing wavelet coefficients for each level of detail, allowing reconstruction of the object at any specified resolution [S1, unofficial].

**Data Structures**
The method uses a hierarchical structure proposed by Pinnamaneni [Pin02] (cited in [S1, unofficial]). For each transformation step, the data is split into two blocks:
1.  **Block 'L':** Contains low-resolution coefficients obtained by a low-pass filter.
2.  **Block 'H':** Contains detail coefficients obtained by a high-pass filter [S1, unofficial].

The final sequence of coefficients for a level consists of the low-resolution coefficients of the last level of detail and all accumulated detail coefficients. The total size of the coefficient sequence equals the size of the original sequence [S1, unofficial].

**Formulas**
The 1D Haar Wavelet transform maps an input sequence $X^0$ to a low-resolution sequence $X^1$ and a detail sequence $Y^1$ using the following formulas for index $i$:

$$
X^1_i = \frac{X^0_{2i} + X^0_{2i+1}}{2}
$$

$$
Y^1_i = \frac{X^0_{2i} - X^0_{2i+1}}{2}
$$

Where:
*   $X^0_{2i}$ and $X^0_{2i+1}$ are adjacent elements in the input sequence.
*   $X^1_i$ is the low-resolution coefficient.
*   $Y^1_i$ is the detail coefficient [S1, unofficial].

The inverse transform reconstructs the original sequence:
$$
X^0_{2i} = X^1_i + Y^1_i
$$
$$
X^0_{2i+1} = X^1_i - Y^1_i
$$
[S1, unofficial]

**Algorithm Steps**
1.  **Define Object:** Represent the 3D object as a uniform spatial enumeration of voxels with byte-coded density values [S1, unofficial].
2.  **X-Direction Transform:** Apply the 1D Haar Wavelet transformation in the x-direction to the input data. This splits the data into low-resolution (L) and detail (H) blocks along the x-axis [S1, unofficial].
3.  **Y-Direction Transform:** Apply the 1D Haar Wavelet transformation in the y-direction to the result of the x-step [S1, unofficial].
4.  **Z-Direction Transform:** Apply the 1D Haar Wavelet transformation in the z-direction to the result of the y-step. This completes one level of 3D decomposition [S1, unofficial].
5.  **Recursive Decomposition:** Repeat steps 2-4 recursively on the low-resolution 'L' block to generate subsequent levels of detail [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def haar_1d_forward(input_seq):
    """
    Performs one level of 1D Haar Wavelet decomposition.
    Returns low-resolution coefficients and detail coefficients.
    """
    n = len(input_seq)
    # Input length must be even (power of 2)
    low_res = []
    detail = []
    for i in range(0, n, 2):
        # Formula: X^1_i = (X^0_{2i} + X^0_{2i+1}) / 2
        low_res.append((input_seq[i] + input_seq[i+1]) / 2.0)
        # Formula: Y^1_i = (X^0_{2i} - X^0_{2i+1}) / 2
        detail.append((input_seq[i] - input_seq[i+1]) / 2.0)
    return low_res, detail

def haar_1d_inverse(low_res, detail):
    """
    Reconstructs the original sequence from low-resolution and detail coefficients.
    """
    n = len(low_res)
    output = [0.0] * (2 * n)
    for i in range(n):
        # Formula: X^0_{2i} = X^1_i + Y^1_i
        output[2*i] = low_res[i] + detail[i]
        # Formula: X^0_{2i+1} = X^1_i - Y^1_i
        output[2*i+1] = low_res[i] - detail[i]
    return output

def decompose_3d_haar(voxel_grid, max_levels):
    """
    Performs 3D Haar Wavelet decomposition on a 3D voxel grid.
    
    Args:
        voxel_grid: 3D array of density values (bytes 0-255).
        max_levels: Number of decomposition levels.
        
    Returns:
        A hierarchical structure of coefficients.
    """
    # Initialize structure to store coefficients for each level
    # Structure contains blocks L (low) and H (detail)
    coefficients = []
    
    # Current working grid starts as the input
    current_grid = voxel_grid
    
    for level in range(max_levels):
        # Step 1: Transform in X-direction
        # Apply 1D Haar transform along the first axis
        # Note: Implementation requires iterating over Y and Z dimensions
        # For each (y, z), transform the x-axis slice
        new_grid_x = apply_transform_axis(current_grid, axis=0)
        
        # Step 2: Transform in Y-direction
        # Apply 1D Haar transform along the second axis
        # For each (x, z), transform the y-axis slice
        new_grid_y = apply_transform_axis(new_grid_x, axis=1)
        
        # Step 3: Transform in Z-direction
        # Apply 1D Haar transform along the third axis
        # For each (x, y), transform the z-axis slice
        new_grid_z = apply_transform_axis(new_grid_y, axis=2)
        
        # Separate Low (L) and High (H) blocks
        # L contains the low-resolution coefficients (first half of each dimension)
        # H contains the detail coefficients (second half or mixed based on filter)
        # The source describes blocks 'L' and 'H' resulting from each step.
        # Typically, L is the low-pass output, H is the high-pass output.
        # In 3D, this results in 8 sub-bands, but the recursive step focuses on L.
        
        low_block = get_low_resolution_block(new_grid_z)
        detail_block = get_detail_block(new_grid_z)
        
        coefficients.append({
            'level': level,
            'low': low_block,
            'detail': detail_block
        })
        
        # Prepare for next level: use the low-resolution block
        current_grid = low_block
        
    return coefficients

def apply_transform_axis(grid, axis):
    """
    Applies 1D Haar transform along a specific axis of a 3D grid.
    """
    # Determine dimensions
    dims = grid.shape
    # Create output grid of same size
    output = [0.0] * dims[0] * dims[1] * dims[2] # Flattened for simplicity in pseudocode logic
    
    # Iterate over all indices not in the transform axis
    # This is a placeholder for the nested loops required to cover all slices
    # The core operation is calling haar_1d_forward on each slice
    
    # Example for axis=0 (X):
    # for y in range(dims[1]):
    #     for z in range(dims[2]):
    #         slice = get_slice(grid, axis=0, y=y, z=z)
    #         low, high = haar_1d_forward(slice)
    #         put_slice(output, low, axis=0, y=y, z=z, type='low')
    #         put_slice(output, high, axis=0, y=y, z=z, type='high')
        
    # Return the transformed grid
    return output

def get_low_resolution_block(grid):
    """
    Extracts the low-resolution block (L) from the transformed grid.
    """
    # Returns the portion of the grid corresponding to low-pass filter outputs
    # In Haar, this is typically the first half of the data along each transformed axis
    pass

def get_detail_block(grid):
    """
    Extracts the detail block (H) from the transformed grid.
    """
    # Returns the portion of the grid corresponding to high-pass filter outputs
    pass
```

**Parameters and Edge Cases**
*   **Resolution:** Typical example is $128 \times 128 \times 128$ voxels with 6 levels of detail (levels 0 to 5) [S1, unofficial].
*   **Voxel Intensity:** 0 for empty, 255 for full [S1, unofficial].
*   **Constraint:** The input sequence length in each dimension must be a power of two. If not, padding or truncation is required (not specified in sources, but implied by the power-of-two requirement) [G].
*   **Display:** The discrete object can be displayed using the marching cubes algorithm to provide a smooth surface, selecting the appropriate LOD based on distance and frame rate requirements [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Building and extracting times depend only on the number of voxels in the initial uniform spatial enumeration and the desired levels of detail, not on the kind of 3D object [S1, unofficial].
*   **Measured Performance:** Measured on a PC with AMD 3GHz, 1GB RAM, NVIDIA Geforce FX 5200 (128MB VRAM) [S1, unofficial].
    *   Building time (decomposition of $128^3$ voxels): 0.6975 s [S1, unofficial].
    *   Extracting times:
        *   Level 0 ($128^3$): 0.7529 s [S1, unofficial].
        *   Level 1 ($64^3$): 0.0627 s [S1, unofficial].
        *   Level 2 ($32^3$): 0.0019 s [S1, unofficial].
        *   Level 3 ($16^3$): 0.0002 s [S1, unofficial].
*   **Strengths:** Faster than AABB/OBB methods for modeling; suitable for virtual sculpture operations involving addition/subtraction of material; allows efficient LOD management [S1, unofficial].
*   **Limitations:** Relies on uniform spatial enumeration which can be memory-intensive for large sparse volumes compared to sparse octrees (though octrees are noted as faster than AABB/OBB) [S1, unofficial]. The source mentions an improvement in section 2.3 to reduce times, but details are not provided in the notes [S1, unofficial].

## Background
*   **Origin:** The method was proposed in the context of "Discrete Tools for Virtual Sculpture" [S1, unofficial].
*   **History:** It builds upon the 1D Haar wavelet decomposition explained by Liu (1988) and Hubbard (1995) in the context of spatial structures, extending it to 3D for multiresolution modeling [S1, unofficial]. The hierarchical structure is based on Pinnamaneni [Pin02] [S1, unofficial].
*   **Usage:** Used for representing 3D objects to be sculpted, allowing easy handling of material addition/subtraction and accelerated display via LOD [S1, unofficial].

## Sources
[S1] Discrete Tools for Virtual Sculpture, https://www.scitepress.org/PublishedPapers/2006/13546/pdf/index.html, fetched 2026-10-27 (unofficial)
