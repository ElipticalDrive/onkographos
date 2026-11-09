# Sparse Linear System Solver for Wave Propagation


## Overview
This method solves for forward and backward acoustic wave amplitudes in a layered medium by formulating the coupled wave equations as a sparse linear system $A\mathbf{x} = \mathbf{b}$ [S1]. It captures multiple internal reflections and transmissions at voxel boundaries defined by acoustic impedance contrasts [S1]. Unlike integral equation solvers that approximate scattering over volumes, this approach models wave propagation along straight-line ray paths through discrete interfaces [S1].

## How it works
**Inputs:**
*   $N$: Number of voxels along the ray path [S1].
*   $Z$: Array of acoustic impedances $Z_0, \dots, Z_N$ for each voxel [S1].
*   $c$: Speed of sound, constant at $1540$ m/s for soft tissue [S1].

**Outputs:**
*   Vector $\mathbf{x}$: Wave amplitudes at each interface [S1].

**Data Structures:**
The unknown vector $\mathbf{x}$ has size $2N+2$ and layout:
$$ \mathbf{x} = [g_0, d_0, g_1, d_1, \dots, g_N, d_N]^T $$
where $g_i$ is the forward-traveling amplitude and $d_i$ is the backward-traveling amplitude at interface $i$ [S1]. The system matrix $A$ is a sparse matrix of size $(2N+2) \times (2N+2)$ [S1]. The right-hand side vector $\mathbf{b}$ has size $2N+2$ [S1].

**Step 1: Compute Interface Coefficients**
For each interface $i$ between voxel $i$ and voxel $i+1$ (where $i = 0, \dots, N-1$), calculate the reflection and transmission coefficients [S1]:
$$ R_{i \rightarrow i+1} = \left| \frac{Z_{i+1} - Z_i}{Z_{i+1} + Z_i} \right| $$
$$ T_{i \rightarrow i+1} = \frac{2Z_{i+1}}{Z_{i+1} + Z_i} $$
The backward transmission coefficient is calculated by swapping indices:
$$ T_{i+1 \rightarrow i} = \frac{2Z_i}{Z_i + Z_{i+1}} $$

**Step 2: Construct Matrix A and Vector b**
Initialize $A$ as a zero matrix of size $(2N+2) \times (2N+2)$ and $\mathbf{b}$ as a zero vector of size $2N+2$ [S1].

1.  **Boundary Conditions:**
    *   Set $A_{0,0} = 1$ and $b_0 = 1$. This enforces $g_0 = 1$ (unit pulse injection at the transducer) [S1].
    *   Set $A_{2N+1, 2N+1} = 1$ and $b_{2N+1} = 0$. This enforces $d_N = 0$ (far-field termination, no incoming waves from beyond the last voxel) [S1].

2.  **Coupled Recurrence Relations:**
    For each voxel index $i$ from $0$ to $N-1$, the physical coupling at the interface is governed by [S1]:
    $$ g_{i+1} = T_{i \rightarrow i+1} \cdot g_i + R_{i+1 \rightarrow i} \cdot d_{i+1} $$
    $$ d_i = R_{i \rightarrow i+1} \cdot g_i + T_{i+1 \rightarrow i} \cdot d_{i+1} $$
    These equations map to rows in the linear system using the variable-to-index mapping $g_k \to 2k$ and $d_k \to 2k+1$ [S1].
    *   **Row for $g_{i+1}$ (index $2i+2$):** Rearranging to $g_{i+1} - T_{i \rightarrow i+1} g_i - R_{i+1 \rightarrow i} d_{i+1} = 0$ yields $A_{2i+2, 2i} = -T_{i \rightarrow i+1}$, $A_{2i+2, 2i+2} = 1$, and $A_{2i+2, 2i+3} = -R_{i+1 \rightarrow i}$ [S1].
    *   **Row for $d_i$ (index $2i+1$):** Rearranging to $d_i - R_{i \rightarrow i+1} g_i - T_{i+1 \rightarrow i} d_{i+1} = 0$ yields $A_{2i+1, 2i} = -R_{i \rightarrow i+1}$, $A_{2i+1, 2i+1} = 1$, and $A_{2i+1, 2i+3} = -T_{i+1 \rightarrow i}$ [S1].

**Parameters:**
*   Speed of sound: $c = 1540$ m/s [S1].
*   Frequency range: $2$–$15$ MHz [S1].

**Edge Cases and Pitfalls:**
*   **Dense Solver:** The source implementation uses a dense solver, which is less efficient than exploiting the matrix sparsity [S1].
*   **Impedance Contrast:** If $Z_i = Z_{i+1}$, then $R=0$ and $T=1$, resulting in no reflection [S1].
*   **Precision:** Standard floating-point arithmetic is assumed; no specific precision requirements are given in the sources [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_wave_propagation(N, Z):
    """
    Solves for wave amplitudes in a layered medium.
    
    Args:
        N (int): Number of voxels.
        Z (list of float): Acoustic impedances Z[0]...Z[N].
        
    Returns:
        x (list of float): Wave amplitudes [g0, d0, ..., gN, dN].
    """
    size = 2 * N + 2
    A = [[0.0] * size for _ in range(size)]
    b = [0.0] * size

    # Boundary conditions: g0 = 1, dN = 0 [S1]
    A[0][0] = 1.0
    b[0] = 1.0
    A[2 * N + 1][2 * N + 1] = 1.0
    b[2 * N + 1] = 0.0

    for i in range(N):
        Z1, Z2 = Z[i], Z[i + 1]
        R_i_to_i1 = abs((Z2 - Z1) / (Z2 + Z1))
        T_i_to_i1 = (2 * Z2) / (Z2 + Z1)
        T_i1_to_i = (2 * Z1) / (Z1 + Z2)

        # Row for g_{i+1} (index 2i+2) [S1]
        r_g = 2 * i + 2
        A[r_g][2 * i] = -T_i_to_i1
        A[r_g][2 * i + 2] = 1.0
        A[r_g][2 * i + 3] = -abs((Z1 - Z2) / (Z1 + Z2))

        # Row for d_i (index 2i+1) [S1]
        r_d = 2 * i + 1
        A[r_d][2 * i] = -R_i_to_i1
        A[r_d][2 * i + 1] = 1.0
        A[r_d][2 * i + 3] = -T_i1_to_i

    # Solve Ax=b using a dense solver [S1]
    x = solve_linear_system(A, b)
    return x

def solve_linear_system(A, b):
    """Standard dense Gaussian elimination solver [G]."""
    n = len(b)
    A = [row[:] for row in A]
    b = b[:]
    for i in range(n):
        pivot = A[i][i]
        if abs(pivot) < 1e-12:
            for j in range(i + 1, n):
                if abs(A[j][i]) > abs(pivot):
                    pivot, A[i], A[j] = A[j][i], A[j], A[i]
                    b[i], b[j] = b[j], b[i]
        if abs(A[i][i]) < 1e-12:
            raise ValueError("Singular matrix")
        for j in range(i + 1, n):
            factor = A[j][i] / A[i][i]
            b[j] -= factor * b[i]
            for k in range(i, n):
                A[j][k] -= factor * A[i][k]
    x = [0.0] * n
    for i in range(n - 1, -1, -1):
        x[i] = b[i]
        for j in range(i + 1, n):
            x[i] -= A[i][j] * x[j]
        x[i] /= A[i][i]
    return x
```

## Subtopics

### Coupled Reflection-Transmission Ray Tracing
This is the underlying ray tracing approach used by the solver [S1]. It models acoustic beams traversing a linear sequence of voxels, with wave amplitudes undergoing reflection and transmission at each voxel boundary according to local impedance contrasts [S1]. It is chosen when straight-line propagation and layered impedance models are sufficient, such as in soft tissue ultrasound simulation [S1]. It differs from full wave solvers by ignoring diffraction and side-lobes [S1].

## Operating characteristics
*   **Complexity:** The matrix $A$ has $O(N)$ non-zero elements due to a fixed-bandwidth structure with at most three non-zero entries per row [S1].
*   **Performance:** The source notes that a dense solver is currently used, which is $O(N^3)$ time complexity, negating the sparsity benefits [S1]. An optimized sparse solver would be $O(N)$ [S1].
*   **Strengths:** Captures multiple internal reflections accurately within the 1D ray approximation [S1].
*   **Limitations:** Assumes straight-line paths, constant speed of sound, and negligible side-lobes [S1]. Not suitable for complex scattering or diffraction effects [S1].

## Background
This method is derived from the physical principles of ultrasound imaging, where acoustic waves propagate through biological tissues [S1]. It was formalized in the DiffUS framework for differentiable ultrasound rendering from volumetric imaging [S1].

## Sources
[S1] DiffUS: Differentiable Ultrasound Rendering from Volumetric Imaging., https://doi.org/10.1007/978-3-032-06329-8_3, fetched 2026-10-26
