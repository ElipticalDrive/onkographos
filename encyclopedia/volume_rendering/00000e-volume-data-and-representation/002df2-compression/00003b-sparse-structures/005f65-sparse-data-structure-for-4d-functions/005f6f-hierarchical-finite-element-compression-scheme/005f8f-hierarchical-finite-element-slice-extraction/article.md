# Hierarchical Finite Element Slice Extraction

Also called: 2D slice extraction from 4D function, Slice Extraction Algorithm

## Overview
Hierarchical Finite Element Slice Extraction is an algorithm for reconstructing a 2D slice $S_2$ from a compressed 4D function $f_c$ given a focus point $p$ [S1, unofficial]. It leverages the finite element property to compute function values at the finest resolution level using Lagrange polynomials, enabling efficient random access for hyperslice reconstruction [S1, unofficial]. It builds on [Hierarchical Finite Element Compression Scheme](17906-hierarchical-finite-element-compression-scheme.md) by providing the specific traversal and interpolation logic required to map the sparse hierarchical structure to a dense 2D output.

## How it works
**Inputs:**
- $p$: The focus point (coordinates in the 4D domain).
- $f_c$: The compressed 4D function data structure.
- $n$: The maximum resolution level (domain size is $2^n$).
- $h$: The base grid spacing.

**Outputs:**
- $S_2$: A 2D slice (array of values) initialized to undefined.

**Data Structures:**
- $f_c$: A sparse structure supporting random access. The existence of a value at a coordinate is checked via `fc(c)`. The structure implicitly defines elements; an element $I_{ij}$ exists if its center $C(I_{ij})$ is present in $f_c$ [S1, unofficial].

**Parameters:**
- $h$: Base grid spacing.
- $n$: Number of resolution levels.
- $M$: Number of points in the output slice.

**Algorithm Steps:**
1. Initialize output slice $S_2$ to undefined.
2. Iterate through resolution levels $j$ from $n-1$ down to $1$ (step -1).
3. Calculate the grid spacing $h_j$ for the current level: $h_j = 2^{n-j} h$.
4. Calculate the center coordinate $c$ for the current dimension relative to focus point $p$: $c = \lfloor p/h_j \rfloor h_j + h_j/2$.
5. Iterate through possible element centers $c_2$ starting from $h_j/2$ up to $2^n$ (step $h_j$).
6. Check if the function value exists at center $c$ in the compressed function $f_c$. If it exists, proceed to compute the polynomial.
7. Define the interval bounds $m$ and $M$ for the element intersection: $m = c_2 - h_j/2$ and $M = c_2 + h_j/2$.
8. Build the Lagrange polynomial $L$ using the compressed function $f_c$, level $j$, center $c$, and focus point $p$.
9. Iterate through slice coordinates $x_2$ from $m$ to $M$ (step $h$).
10. If $S_2(x_2)$ is not yet defined, evaluate the polynomial $L$ at $x_2$ and store the result in $S_2(x_2)$. The evaluation uses the normalized coordinate $(x_2 - m)/(M - m)$.

**Polynomial Construction:**
The approximation of function $f$ on the interval $[c_2 - h_j/2, c_2 + h_j/2]$ is a 1D Lagrange polynomial. By considering the reference element $[0, 2]$, the reference polynomial $L(x)$ is defined as [S1, unofficial]:

$$
L(x) = f_0 + (f_1 - f_0)x + \frac{1}{2}(f_2 - 2f_1 + f_0)x(x - 1)
$$

where:
- $f_0 = f_c(p_1, c_2 - h_j/2)$
- $f_1 = f_c(p_1, c_2)$
- $f_2 = f_c(p_1, c_2 + h_j/2)$

The center of element $I_{ij}$ is defined as [S1, unofficial]:

$$
C(I_{ij}) = \frac{x_i + x_{i+1}}{2} = x_i + \frac{h_j}{2}
$$

**Missing Values Handling:**
If the required function values ($f_0, f_1, f_2$) do not exist in $f_c$, they must be computed using a similar 1D Lagrange polynomial in the other dimension [S1, unofficial]. For example, if $f_0$ is missing, it is built using $f_c(c_1 - h_j/2, c_2 - h_j/2)$, $f_c(c_1, c_2 - h_j/2)$, and $f_c(c_1 + h_j/2, c_2 - h_j/2)$ evaluated at $p_1$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_slice(f_c, p, n, h):
    """
    Extracts a 2D slice S2 from a compressed 4D function f_c.
    
    Args:
        f_c: Compressed function data structure with random access.
        p: Focus point (tuple/list of coordinates).
        n: Maximum resolution level.
        h: Base grid spacing.
        
    Returns:
        S2: 2D slice array.
    """
    # Initialize output slice to undefined
    # Assuming S2 is a 2D array covering the domain [0, 2^n] x [0, 2^n]
    # with resolution h.
    domain_size = 2**n
    num_points = int(domain_size / h)
    S2 = [[None for _ in range(num_points)] for _ in range(num_points)]
    
    # Iterate through resolution levels from n-1 down to 1
    for j in range(n - 1, 0, -1):
        # Calculate grid spacing for current level
        h_j = (2**(n - j)) * h
        
        # Calculate center coordinate c for the current dimension relative to p
        # Note: The source describes a 1D slice extraction from a 2D function
        # as the primary example. For a 4D function, this logic extends to
        # 2D traversal for the slice coordinates.
        # Here we follow the source's 1D slice example logic for S2(x2).
        
        # Calculate center c (assuming p is 1D for the slice dimension or
        # projecting p onto the slice dimension)
        # The source uses c = floor(p/h_j)*h_j + h_j/2
        c = (int(p / h_j) * h_j) + (h_j / 2)
        
        # Iterate through possible element centers c2
        # Starting from h_j/2 up to 2^n with step h_j
        c2 = h_j / 2
        while c2 <= domain_size:
            # Check if function value exists at center c in f_c
            # Note: The source checks fc(c). In a multi-dimensional context,
            # this implies checking the existence of the element center.
            if f_c.exists(c):
                # Define interval bounds m and M
                m = c2 - (h_j / 2)
                M = c2 + (h_j / 2)
                
                # Build Lagrange polynomial L
                # This requires f0, f1, f2.
                # f0 = f_c(p1, c2 - h_j/2)
                # f1 = f_c(p1, c2)
                # f2 = f_c(p1, c2 + h_j/2)
                # Where p1 is the coordinate of p in the other dimension.
                
                # Attempt to get values
                try:
                    f0 = f_c.get(p[0], c2 - h_j/2)
                    f1 = f_c.get(p[0], c2)
                    f2 = f_c.get(p[0], c2 + h_j/2)
                except KeyError:
                    # If values are missing, compute them using 1D polynomial
                    # in the other dimension.
                    # Example for f0:
                    # f0 = build_1d_poly(f_c, c1 - h_j/2, c1, c1 + h_j/2, c2 - h_j/2, p[0])
                    # This requires a helper function or recursive call.
                    # For the sake of complete pseudocode, we assume a helper exists.
                    f0 = compute_missing_value(f_c, p, c, c2, h_j, 'f0')
                    f1 = compute_missing_value(f_c, p, c, c2, h_j, 'f1')
                    f2 = compute_missing_value(f_c, p, c, c2, h_j, 'f2')
                
                # Build polynomial coefficients
                # L(x) = f0 + (f1 - f0)x + 0.5*(f2 - 2*f1 + f0)*x*(x - 1)
                # where x is normalized coordinate in [0, 2]
                
                # Iterate through slice coordinates x2 from m to M with step h
                x2 = m
                while x2 <= M:
                    if S2[x2_index(x2)][y2_index(x2)] is None:
                        # Evaluate polynomial at x2
                        # Normalized coordinate
                        x_norm = (x2 - m) / (M - m)
                        # Map to reference element [0, 2]
                        # The source uses reference element [0, 2].
                        # If x2 is in [m, M], then (x2-m)/(M-m) is in [0, 1].
                        # The source formula uses x in [0, 2].
                        # Let's assume the source's x corresponds to the normalized
                        # coordinate scaled to [0, 2].
                        x_ref = 2 * x_norm
                        
                        val = f0 + (f1 - f0) * x_ref + 0.5 * (f2 - 2*f1 + f0) * x_ref * (x_ref - 1)
                        S2[x2_index(x2)][y2_index(x2)] = val
                    x2 += h
            c2 += h_j

def compute_missing_value(f_c, p, c, c2, h_j, target):
    """
    Computes a missing function value using a 1D Lagrange polynomial
    in the other dimension.
    """
    # This is a placeholder for the specific logic described in the source.
    # For f0, use fc(c1 - h_j/2, c2 - h_j/2), fc(c1, c2 - h_j/2), fc(c1 + h_j/2, c2 - h_j/2)
    # evaluated at p1.
    # The exact implementation depends on the dimensionality and indexing.
    pass

def x2_index(x2):
    return int(x2 / h)

def y2_index(x2):
    # Assuming 2D slice, y2 might be fixed or derived from p.
    # The source focuses on S2(x2).
    return 0
```

## Operating characteristics
- **Time Complexity:** $O(M)$, where $M$ is the number of points in the slice [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Execution speed depends on the efficiency of random accesses to $f_c$ values [S1, unofficial]. The amount of computation is not very important [S1, unofficial].
- **Strengths:** Enables interactive performance by keeping efficient random accesses and loading the whole compressed function into core memory [S1, unofficial].
- **Limitations:** Performance is heavily dependent on the underlying data structure's random access efficiency [S1, unofficial].

## Background
The method is part of a dedicated compression scheme for large multidimensional functions visualization [S1, unofficial]. It was developed to allow interactive exploration of large datasets by replacing quantization and encoding steps with a dedicated sparse data structure [S1, unofficial]. The technique applies to the reconstruction of slices from 4D compressed functions, extending the principle from 2D to higher dimensions via tensor product construction [S1, unofficial].

## Sources
[S1] A dedicated Compression Scheme for Large Multidimensional Functions Visualization, https://inria.hal.science/inria-00591076v1, fetched 2026-10-26 (unofficial)
