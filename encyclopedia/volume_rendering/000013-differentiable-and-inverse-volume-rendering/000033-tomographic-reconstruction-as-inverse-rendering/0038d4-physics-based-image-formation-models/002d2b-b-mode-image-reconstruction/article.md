# B-mode Image Reconstruction

Also called: Image Formation Model

## Overview
B-mode Image Reconstruction is a differentiable forward model for synthesizing ultrasound B-mode images from volumetric data, primarily used in differentiable ultrasound rendering [S1]. It distinguishes itself from standard ray-marching volume rendering by modeling coupled forward and backward wave amplitudes across voxel interfaces using a sparse linear system, rather than simple line integrals of density [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric representation of tissue acoustic properties, specifically reflection coefficients $R$ and transmission coefficients $T$ [S1].
*   **Output:** A 2D B-mode image where pixel intensity corresponds to echo amplitude first differences, mapped to beam angle and depth [S1].

**Data Structures:**
The core data structure is a sparse matrix $A$ representing the coupled recursion relations for wave amplitudes [S1]. For a depth of $N$ voxels, the matrix has dimensions $(2N+2) \times (2N+2)$ [S1]. The matrix is strictly banded with at most three non-zero entries per row, resulting in $O(N)$ non-zero elements [S1]. The indices $r$ (row) and $c$ (column) are defined for $i = 0, \dots, N-1$ and $r, c = 0, \dots, 2N+1$ [S1].

**Formulas:**
The matrix elements $A_{r,c}$ are defined as [S1]:

$$
A_{r,c} = \begin{cases} 
-T_{i+1 \rightarrow i} & r = 2i + 1, c = 2i + 3 \\
-T_{i \rightarrow i+1} & r = 2i + 2, c = 2i \\
1 & r = 2i + 2, c = 2i + 2 \\
-R_{i+1 \rightarrow i} & r = 2i + 2, c = 2i + 3 \\
0 & \text{otherwise}
\end{cases}
$$

Where:
*   $T_{i \rightarrow i+1}$ is the transmission coefficient from voxel $i$ to $i+1$ [S1].
*   $R_{i+1 \rightarrow i}$ is the reflection coefficient from voxel $i+1$ back to $i$ [S1].

**Steps:**
1.  **Initialize Wave System:** For a given beam direction, set up the linear system $A \mathbf{x} = \mathbf{b}$, where $\mathbf{x}$ contains the forward ($g_i$) and backward ($d_i$) wave amplitudes for each voxel interface [S1].
2.  **Solve for Amplitudes:** Solve the sparse linear system to determine the wave amplitudes at each depth step $k = 1, \dots, N$ [S1]. The implementation may use a dense solver for simplicity, though banded solvers are suitable due to sparsity [S1].
3.  **Measure Echoes:** At each depth step $k$, measure the returning echo amplitude $d_0^{(k)}$ at the transducer location [S1].
4.  **Generate Depth Profile:** Compile the sequence $\{d_0^{(1)}, \dots, d_0^{(N)}\}$ to form the time-sequence of echoes for the current beam direction [S1].
5.  **Compute B-mode Intensity:** Calculate pixel intensities as the first differences of the echo amplitudes [S1].
6.  **Fan-Shaped Acquisition:** Repeat steps 1–5 for multiple ray directions originating from the virtual transducer position to create a fan-shaped acquisition pattern [S1].
7.  **Map Coordinates:** Assign spatial coordinates to pixels based on the beam angle and depth [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_bmode(coefficients_per_beam, beam_angles):
    # coefficients_per_beam: list of lists, each inner list contains (R_i, T_i) tuples for i=0..N-1
    image = []
    for angle_idx, angle in enumerate(beam_angles):
        coeffs = coefficients_per_beam[angle_idx]
        N = len(coeffs)
        size = 2 * N + 2
        A = [[0.0] * size for _ in range(size)]
        for i in range(N):
            R_i1, T_i1 = coeffs[i + 1] if i + 1 < N else (0.0, 0.0)
            T_i = coeffs[i][1]
            if 2 * i + 3 < size:
                A[2 * i + 1][2 * i + 3] = -T_i1
            if 2 * i >= 0:
                A[2 * i + 2][2 * i] = -T_i
            A[2 * i + 2][2 * i + 2] = 1.0
            if 2 * i + 3 < size:
                A[2 * i + 2][2 * i + 3] = -R_i1
        b = [0.0] * size
        b[0] = 1.0
        x = solve_linear_system(A, b)
        echoes = [x[2 * k + 1] for k in range(1, N + 1)]
        intensities = [abs(echoes[k] - echoes[k - 1]) for k in range(1, len(echoes))]
        image.append(intensities)
    return image

def solve_linear_system(A, b):
    # Solve Ax = b using Gaussian elimination with partial pivoting [G]
    n = len(b)
    M = [A[i][:] + [b[i]] for i in range(n)]
    for i in range(n):
        max_row = max(range(i, n), key=lambda k: abs(M[k][i]))
        M[i], M[max_row] = M[max_row], M[i]
        for k in range(i + 1, n):
            factor = M[k][i] / M[i][i]
            for j in range(i, n + 1):
                M[k][j] -= factor * M[i][j]
    x = [0.0] * n
    for i in range(n - 1, -1, -1):
        x[i] = (M[i][n] - sum(M[i][j] * x[j] for j in range(i + 1, n))) / M[i][i]
    return x
```

**Parameters:**
*   **Voxel Depth ($N$):** Number of voxels along the ray path [S1].
*   **Reflection/Transmission Coefficients:** Material properties defining wave interaction at interfaces [S1].

**Edge Cases and Pitfalls:**
*   **Solver Choice:** While the matrix $A$ is sparse with a fixed bandwidth, the source notes that a dense solver is used for simplicity [S1]. Using a dense solver increases computational cost to $O(N^3)$ for solving, whereas a banded solver would be $O(N)$ [S1].

## Subtopics

### Speckle Noise Injection
An optional post-processing stage that injects speckle as multiplicative noise with depth-dependent amplitude [S1]. The noise consists of a radially oriented term and a granular term [S1]. The strength and granularity of the noise intensify as the signal-to-noise ratio declines in deeper regions [S1].

### Depth-Dependent Blurring
An optional post-processing stage that imposes depth-related blurring by convolving the image with a Gaussian kernel [S1]. The width of the Gaussian kernel grows linearly with depth [S1].

## Operating characteristics
*   **Time Complexity:** The construction of the sparse matrix $A$ is $O(N)$ due to its fixed bandwidth and local interactions [S1]. Solving the system depends on the solver: $O(N^3)$ for dense solvers, $O(N)$ for banded solvers [S1].
*   **Memory Complexity:** The matrix $A$ requires $O(N)$ storage for non-zero elements [S1].
*   **Strengths:** Physically accurate modeling of wave propagation and interference, enabling realistic ultrasound simulation for differentiable rendering [S1].
*   **Limitations:** Computational cost can be high if dense solvers are used for large $N$ [S1].

## Background
The method is described in the DiffUS paper, which introduces a differentiable ultrasound rendering framework [S1]. It models wave propagation using coupled recursion relations for forward and backward wave amplitudes across voxel interfaces [S1].

## Sources
[S1] DiffUS: Differentiable Ultrasound Rendering from Volumetric Imaging., https://doi.org/10.1007/978-3-032-06329-8_3, fetched 2026-10-26
