# Nonstandard 2D Wavelet Decomposition

Also called: Alternating Row-Column Wavelet, Lifted 2D Wavelet

## Overview
Nonstandard 2D Wavelet Decomposition is a multi-dimensional extension strategy for wavelet transforms where 1D transforms are alternated between rows and columns at each resolution level [S1, unofficial]. It solves the problem of mixed-resolution basis functions found in standard decomposition, providing direct control of the resolution level $m$ and ensuring all basis functions have square support [S1, unofficial]. It is more computationally efficient than the standard row-then-column approach [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 2D data array $f(x,y)$.
*   Output: Scaling coefficients $c_{mpq}$ and detail coefficients $d^{type}_{mpq}$ for types 1, 2, and 3, organized by resolution level $m$.

**Data Structures:**
The decomposition generates coefficients arranged by resolution level. For a given level $m$, the basis functions are defined on a grid. The coefficients are stored such that the scaling coefficients $c_{mpq}$ represent the approximation at level $m$, while $d^{type}_{mpq}$ represent the details added at level $m$.

**Basis Functions:**
The 2D basis functions are constructed as tensor products of 1D scaling functions $\phi$ and wavelets $\psi$ [S1, unofficial]:
$$
\begin{aligned}
\phi_{mpq}^2(x, y) &= \phi_{mp}(x)\phi_{mq}(y) \\
\psi_{mpq}^{2,1}(x, y) &= \psi_{mp}(x)\phi_{mq}(y) \\
\psi_{mpq}^{2,2}(x, y) &= \phi_{mp}(x)\psi_{mq}(y) \\
\psi_{mpq}^{2,3}(x, y) &= \psi_{mp}(x)\psi_{mq}(y)
\end{aligned}
$$
where type 1 corresponds to horizontal details, type 2 to vertical details, and type 3 to diagonal details [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the function approximation with the coarsest level scaling coefficients $c_{m_0 pq}$ [S1, unofficial].
2.  For each resolution level $m$ from $m_0 + 1$ upwards:
    a. Apply the 1D wavelet transform to all rows of the current approximation. This separates row-wise details from row-wise approximations.
    b. Apply the 1D wavelet transform to the relevant columns of the result. This separates column-wise details from column-wise approximations.
    c. Extract the detail coefficients $d^{type}_{mpq}$ corresponding to the wavelet components generated in steps (a) and (b).
    d. Update the scaling coefficients $c_{mpq}$ with the approximation components.
3.  The final representation is the sum of the coarsest scaling coefficients and all detail coefficients from all levels [S1, unofficial]:
    $$
    f(x, y) = \sum_{p, q} c_{m_0 pq} \phi_{m_0 pq}^2(x, y) + \sum_{m = m_0 + 1}^{\infty} \sum_{\text{type}=1}^{3} \sum_{p, q} d_{mpq}^{\text{type}} \psi_{mpq}^{2, \text{type}}(x, y)
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nonstandard_2d_wavelet_decomposition(data, transform_1d):
    """
    Performs nonstandard 2D wavelet decomposition.
    
    Args:
        data: 2D array of input values.
        transform_1d: Function performing one iteration of forward 1D wavelet transform.
                      Returns (approximation, details).
    
    Returns:
        coefficients: Dictionary or structure containing scaling and detail coefficients
                      organized by resolution level.
    """
    # Initialize with the input data
    current_data = data.copy()
    coefficients = {}
    
    # Assume we decompose down to a single coarse level or a specified min_level
    # In a full multi-resolution setup, this loop continues until the desired coarsest level is reached.
    # For the purpose of this algorithm description, we iterate through levels.
    
    level = 0
    while current_data.shape[0] > 1: # Stop when 1x1 scaling coefficient remains
        row_approximations = []
        row_details = []
        
        # Step 1: Transform rows
        for r in range(current_data.shape[0]):
            approx, detail = transform_1d(current_data[r, :])
            row_approximations.append(approx)
            row_details.append(detail)
            
        # Stack row approximations to form the intermediate matrix for column transform
        # Note: row_details contains the horizontal details (Type 1)
        # We need to process the column approximations to get vertical (Type 2) and diagonal (Type 3)
        
        intermediate_matrix = []
        for r in range(len(row_approximations)):
            intermediate_matrix.append(row_approximations[r])
            
        col_approximations = []
        col_details_v = [] # Vertical details (Type 2)
        col_details_d = [] # Diagonal details (Type 3)
        
        # Step 2: Transform columns of the row-approximated matrix
        for c in range(intermediate_matrix[0].shape[0]):
            col_data = [row[c] for row in intermediate_matrix]
            approx, detail = transform_1d(col_data)
            col_approximations.append(approx)
            col_details_v.append(detail)
            
            # The diagonal details come from the interaction of row and column wavelets.
            # In the nonstandard scheme, the diagonal details are often derived from 
            # the wavelet coefficients of the columns applied to the row wavelet coefficients,
            # or explicitly stored. Based on the basis function definition:
            # Type 3 = psi(x) * psi(y).
            # The standard nonstandard algorithm extracts these during the column transform 
            # if the 1D transform returns all components.
            # However, the source [S1, unofficial] implies d^{type} are extracted.
            # Let's assume transform_1d returns (approx, detail).
            # The diagonal details are typically the wavelet coefficients of the columns 
            # applied to the row wavelet coefficients? No, the nonstandard scheme alternates.
            # Actually, the standard nonstandard algorithm (LJFIPACK style) processes rows then columns.
            # The details from the row transform are Type 1.
            # The details from the column transform on the *approximations* are Type 2.
            # The details from the column transform on the *details* are Type 3.
            
            # Re-evaluating based on [S1, unofficial] formula:
            # f = sum c phi + sum d^1 psi_phi + sum d^2 phi_psi + sum d^3 psi_psi
            # Row transform: f -> approx (phi) + detail (psi).
            # So Row Transform produces Type 1 (psi_phi) and leaves phi_phi.
            # Column Transform on phi_phi produces Type 2 (phi_psi) and leaves phi_phi.
            # Column Transform on psi_phi produces Type 3 (psi_psi) and leaves psi_phi.
            
            # Therefore, we must transform columns of BOTH the row-approximations and row-details.
            
        # Corrected Step 2: Transform columns of row-approximations AND row-details
        final_approximations = []
        final_details_v = []
        final_details_d = []
        
        # Process columns of row approximations (produces Type 2)
        for c in range(intermediate_matrix[0].shape[0]):
            col_data = [row[c] for row in intermediate_matrix]
            approx, detail = transform_1d(col_data)
            final_approximations.append(approx)
            final_details_v.append(detail)
            
        # Process columns of row details (produces Type 3)
        # Note: row_details is a list of arrays. We need to transpose it to get columns.
        if row_details:
            num_cols = row_details[0].shape[0]
            for c in range(num_cols):
                col_data = [row[c] for row in row_details]
                approx, detail = transform_1d(col_data)
                # The 'approx' part of this transform corresponds to the Type 3 coefficient?
                # No, the 'detail' part corresponds to the wavelet component.
                # The basis function for Type 3 is psi(x)psi(y).
                # The input to the column transform is psi(x) (from row transform).
                # The output 'detail' is psi(y). So the product is psi(x)psi(y).
                final_details_d.append(detail)
                
        # Store coefficients for this level
        coefficients[level] = {
            'scaling': final_approximations, # This is the input for the next level
            'type1': row_details,            # Horizontal details
            'type2': final_details_v,        # Vertical details
            'type3': final_details_d         # Diagonal details
        }
        
        # Prepare for next level
        current_data = final_approximations
        level += 1
        
    return coefficients
```

**Parameters:**
*   `transform_1d`: The specific 1D wavelet transform kernel (e.g., B-spline wavelet). Typical values depend on the chosen wavelet order.
*   Resolution levels: Determined by the data size and desired granularity.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The 1D transform must handle boundaries correctly. Padding or extension strategies must be consistent with the basis function support [S1, unofficial].
*   **Precision:** Accumulation of errors in multi-level decomposition can affect reconstruction quality. Use of high-precision arithmetic may be necessary for critical applications [G].
*   **Degenerate Input:** If the data size is not a power of 2, the decomposition may require padding or truncation at the finest/coarsest levels [G].

## Subtopics

### Optimized Non-Standard 2D Wavelet Decomposition
A modification to the standard non-standard technique that reduces computational requirements by skipping columns that contain only wavelet coefficients [S2, unofficial]. In the standard algorithm, the 1D transform is applied to all columns after the row transform. In the optimized version, the 1D transform is applied only to columns comprising data values (i.e., every alternate column is skipped) [S2, unofficial]. For a data set size of $s \times s$, the number of data elements processed by the 1D transform reduces from $2s^2$ to $1.5s^2$, a 25% reduction in computational cost [S2, unofficial]. This modification generates the same multiresolution data sets as the original algorithm [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The nonstandard decomposition is slightly more efficient than the standard decomposition [S1, unofficial]. The optimized variant reduces the number of 1D transforms by 25% [S2, unofficial].
*   **Memory Complexity:** Requires storage for scaling and detail coefficients at all levels. The structure allows for efficient compression of detail coefficients due to their sparsity and decreasing importance [S1, unofficial].
*   **Strengths:** Direct control of resolution level $m$; square support for all basis functions; computational efficiency; compatibility with lifting schemes [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires careful handling of boundary conditions; the optimization assumes specific data structures (skipping columns) which may not apply to all data layouts [S2, unofficial].
*   **When to use:** When direct resolution control and square support are required, and computational efficiency is a priority. Particularly suitable for B-spline wavelet compression [S1, unofficial].

## Background
The nonstandard decomposition was introduced as an alternative to the standard row-then-column decomposition to overcome the limitations of mixed-resolution basis functions [S1, unofficial]. It is widely used in multiresolution representation and compression of surfaces and volumes [S1, unofficial]. The optimized variant was developed to further reduce computational requirements in implementations like LJFIPACK [S2, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
[S2] Seamless Multiresolution Isosurfaces Using Wavelets, https://digital.library.unt.edu/ark:/67531/metadc702274/, fetched 2026-10-26 (unofficial)
