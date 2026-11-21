# B-spline Wavelet Compression

Also called: B-spline FWT, B-spline Wavelet Transform, Fast Wavelet Transform, Matrix-based Wavelet Transform

## Overview
B-spline Wavelet Compression is a method for compressing volumetric data by decomposing it into a hierarchy of scaling and wavelet coefficients using a semi-orthogonal B-spline basis [S1, unofficial]. It solves the problem of efficient storage and transmission of volume data by exploiting the decay of coefficients across scales, specifically addressing the lack of compact support in B-spline dual functions through a matrix-based projection scheme [S1, unofficial]. It sets itself apart from standard orthogonal wavelet transforms by allowing linear complexity construction and smooth basis functions, and from Haar wavelets by providing better approximation properties for smooth data [S1, unofficial]. It builds on [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md) by implementing the specific B-spline filter bank and matrix operations.

## How it works
**Inputs:** A 1D data vector $\mathbf{c}_m$ of length $N = 2^{M-m} + d$, where $M$ is the maximum decomposition level, $m$ is the current level, and $d$ is the B-spline degree [S1, unofficial]. For 3D volumes, this transform is applied separably along each axis [S1, unofficial].
**Outputs:** Scaling coefficients $\mathbf{c}_{M}$ and wavelet coefficients $\mathbf{d}_{k}$ for $k = m+1 \dots M$ [S1, unofficial].

**Data Structures:**
*   $\mathbf{c}_m$: Input coefficient vector.
*   $\mathbf{C}_m$: Band-diagonal inner-product matrix for primal-to-dual projection.
*   $\mathbf{D}_{m+1}$: Band-diagonal inner-product matrix for wavelet space, defined as $\mathbf{D}_{m+1} = \mathbf{Q}_{m+1}^T \mathbf{C}_m \mathbf{Q}_{m+1}$ [S1, unofficial].
*   $\mathbf{P}_{m+1}, \mathbf{Q}_{m+1}$: Primal and dual projection matrices (implicitly defined or precomputed) [S1, unofficial].
*   $\tilde{\mathbf{c}}_m, \tilde{\mathbf{d}}_m$: Dual space coefficient vectors.

**Parameters:**
*   $d$: Degree of the B-spline. For cubic B-splines, $d=3$ [S1, unofficial].
*   $M$: Maximum decomposition level [S1, unofficial].

**Steps:**
1.  **Expansion:** Expand the original data vector $\mathbf{c}_m$ by repeating the last data value to yield the required length $N = 2^{M-m} + d$ [S1, unofficial].
2.  **Projection to Dual Space:** Transform the primal coefficients to dual space to handle the non-compact support of B-spline dual functions. Compute $\tilde{\mathbf{c}}_m = \mathbf{C}_m \mathbf{c}_m$ [S1, unofficial]. Since $\mathbf{C}_m$ is band-diagonal, this is $O(N)$ [S1, unofficial].
3.  **Decomposition:** Apply the transpose of the projection matrices to generate dual-space approximation and detail coefficients:
    $$ \tilde{\mathbf{c}}_{m+1} = (\mathbf{P}_{m+1})^T \tilde{\mathbf{c}}_m $$
    $$ \tilde{\mathbf{d}}_{m+1} = (\mathbf{Q}_{m+1})^T \tilde{\mathbf{c}}_m $$
    The lengths of $\tilde{\mathbf{c}}_{m+1}$ and $\tilde{\mathbf{d}}_{m+1}$ are $2^{M-m-1}+1$ and $2^{M-m-1}+d$ respectively, allowing space-efficient in-place implementation [S1, unofficial].
4.  **Recovery of Detail Signals:** Transform the dual-space detail coefficients back to primal space by solving the linear system:
    $$ \mathbf{D}_{m+1} \mathbf{d}_{m+1} = \tilde{\mathbf{d}}_{m+1} $$
    The matrix $\mathbf{D}_{m+1}$ is diagonally dominant and well-conditioned, but its inverse is dense and ill-conditioned. Therefore, brute-force inversion is avoided. Instead, an LU-decomposition is used to solve for $\mathbf{d}_{m+1}$ in linear time [S1, unofficial].
5.  **Recursion:** Repeat steps 2-4 for the approximation coefficients $\tilde{\mathbf{c}}_{m+1}$ until the final level $M$ is reached. At level $M$, compute $\mathbf{c}_M$ similarly to the detail signal recovery [S1, unofficial].
6.  **Higher Dimensions:** For 3D data, apply the 1D transform separably. In higher dimensions, mixed bands of primal and dual coefficients occur. A single iteration involves operators $L^*, H^*, L^{-1}, H^{-1}$ to separate scaling and wavelet subbands, with the backprojection of the pure scaling function subband into primal space carried out separately [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bspline_wavelet_decompose(c_m, M, m, d, C_m, P_m, Q_m):
    """
    Decomposes a 1D vector c_m into wavelet coefficients using B-spline FWT.
    c_m: Input vector of length N = 2^(M-m) + d
    M: Max level
    m: Current level
    d: B-spline degree
    C_m: Band-diagonal inner-product matrix (primal to dual)
    P_m, Q_m: Projection matrices (precomputed or implicit)
    Returns: List of detail vectors [d_{m+1}, ..., d_M] and final scaling vector c_M
    """
    # Step 1: Expansion (if input is not already expanded)
    # Note: The source assumes input c_m is already expanded to length 2^(M-m) + d
    # If not, repeat last element until length matches.
    
    current_c = c_m
    details = []
    
    for level in range(m + 1, M + 1):
        # Step 2: Project to dual space
        # c_tilde = C_m * c
        c_tilde = mat_vec_mult(C_m, current_c)
        
        # Step 3: Decompose into dual approximation and detail
        # c_tilde_next = P^T * c_tilde
        # d_tilde = Q^T * c_tilde
        # Note: P and Q indices depend on level. 
        # Using P_{level} and Q_{level} from source notation.
        P_mat = get_projection_matrix(P_m, level)
        Q_mat = get_projection_matrix(Q_m, level)
        
        c_tilde_next = transpose_mat_vec_mult(P_mat, c_tilde)
        d_tilde = transpose_mat_vec_mult(Q_mat, c_tilde)
        
        # Step 4: Recover primal detail signal
        # D_{level} * d = d_tilde
        # D_{level} = Q^T * C_{prev} * Q
        # Solve using LU decomposition
        D_mat = compute_D_matrix(Q_mat, C_m, level) # C_m here refers to C_{level-1} in recursive context
        d_primal = solve_lu(D_mat, d_tilde)
        
        details.append(d_primal)
        
        # Prepare for next iteration
        current_c = c_tilde_next
        # Update C_m for next level if necessary, or use precomputed C_{level}
        C_m = get_C_matrix(level) # Assuming C_m is available for next level
        
    # Final scaling coefficients
    c_M = current_c
    return details, c_M

def solve_lu(D, rhs):
    """
    Solves D * x = rhs using LU decomposition.
    D is diagonally dominant and well-conditioned.
    """
    # Perform LU decomposition of D
    L, U = lu_decompose(D)
    # Solve L * y = rhs
    y = forward_substitution(L, rhs)
    # Solve U * x = y
    x = backward_substitution(U, y)
    return x
```

## Subtopics

### [B-spline Wavelet Lifting Scheme](7585-b-spline-wavelet-lifting-scheme.md)
A method to generate best approximations in a multiresolution hierarchy using B-spline wavelets, avoiding the over- and undershoots (and resulting topological changes in isosurfaces) caused by standard subdivision wavelets. It sets itself apart from standard subdivision wavelets by using B-spline lifting to avoid these artifacts, and it builds on [B-spline Wavelet Compression](5377-b-spline-wavelet-compression.md) for subsequent delta encoding, quantization, run-length, and Huffman coding.

### [Progressive Compression System](7668-progressive-compression-system.md)
The Progressive Compression System is a method for compressing volumetric subdivision meshes by constructing a progressive bit stream from a hierarchy of subdivision wavelets, optimized for coarse-to-fine reconstruction. It solves the problem of efficient storage and transmission of complex 3D meshes by exploiting the decay behavior of wavelet coefficients from coarse to fine spatial scales. Unlike standard zerotree algorithms that rely on a strict theoretical framework for parent-child relationships, this system applies intuitive spatial relationships analogous to image coding trees to define coefficient hierarchies for subdivision wavelets. It sets itself apart from its parent [B-spline Wavelet Compression](5377-b-spline-wavelet-compression.md) by specifically integrating the SPIHT coder to handle the hierarchical tree structure of subdivision wavelets, rather than relying solely on delta encoding and Huffman coding for general coefficient lists.

### [Nonstandard 2D Wavelet Decomposition](11778-nonstandard-2d-wavelet-decomposition.md)
Nonstandard 2D Wavelet Decomposition is a multi-dimensional extension strategy for wavelet transforms where 1D transforms are alternated between rows and columns at each resolution level. It solves the problem of mixed-resolution basis functions found in standard decomposition, providing direct control of the resolution level $m$ and ensuring all basis functions have square support. It is more computationally efficient than the standard row-then-column approach.

### Wavelet-GS
A compression method that compresses the SH coefficients of each Gaussian using a wavelet decomposition, discarding high-frequency components that are not visible at typical viewing distances [S3, unofficial].

### 2D Wavelet Decomposition
Adaptation of the 1D decomposition scheme to higher dimensions, handling mixed bands comprising both primal and dual coefficients [S1, unofficial]. It performs single iteration steps involving operators $L^*, H^*, L^{-1}, H^{-1}$ to separate scaling and wavelet subbands, and carries out the backprojection of the pure scaling function subband into primal space separately [S1, unofficial]. The overall complexity is $O(N)$ [S1, unofficial].

## Operating characteristics
**Time Complexity:** The decomposition and reconstruction complexity is $O(N)$, where $N$ is the length of the data vector [S1, unofficial]. This linear complexity is a key advantage over general bi- or semi-orthogonal wavelets where complexity might not be linear [S1, unofficial].
**Memory Complexity:** Space-efficient in-place implementation is possible because the total length of the decomposition vectors ($\tilde{\mathbf{c}}_{m+1}$ and $\tilde{\mathbf{d}}_{m+1}$) equals the length of the original vector [S1, unofficial].
**Performance:** Full decomposition and reconstruction of a 1D data vector requires approximately 30.25 multiplications per data value [S1, unofficial].
**Strengths:** Linear time complexity, smooth basis functions, avoidance of topological artifacts in isosurfaces (when using lifting variants), and efficient handling of semi-orthogonal bases [S1, unofficial].
**Limitations:** The dual functions do not have compact support, requiring the use of inner-product matrices ($\mathbf{C}_m, \mathbf{D}_m$) and LU-decomposition, which adds implementation complexity compared to orthogonal wavelets [S1, unofficial]. The inverse of $\mathbf{D}_{m+1}$ is dense and ill-conditioned, necessitating stable solvers [S1, unofficial].
**When to use:** Suitable for compressing smooth volumetric data where linear complexity and smooth reconstruction are prioritized over the simplicity of orthogonal transforms.
**When not to use:** When hardware acceleration for simple FIR filters is critical and the overhead of matrix operations is prohibitive, or when compact support of both primal and dual bases is strictly required.

## Background
Originating from the work of Quak and Weyrich, this method was detailed in the thesis "Multiresolution representation and compression of surfaces and volumes" [S1, unofficial]. It addresses the limitations of standard wavelet transforms when applied to B-spline bases, specifically the lack of compact support in dual functions. The method has been applied in volume rendering compression pipelines, such as those by Lippert et al., which use B-spline wavelets for preprocessing and compression [S2]. More recently, variants like Wavelet-GS have been applied to compress spherical harmonic coefficients in 3D Gaussian Splatting [S3, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
