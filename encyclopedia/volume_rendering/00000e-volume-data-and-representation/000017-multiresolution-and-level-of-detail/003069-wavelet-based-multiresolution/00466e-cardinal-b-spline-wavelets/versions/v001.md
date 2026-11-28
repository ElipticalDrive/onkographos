# Cardinal B-spline wavelets

Also called: B-spline Wavelet Construction, B-spline wavelets, B-wavelet, Chui wavelets, Unser wavelets

## Overview
Cardinal B-spline wavelets are a class of semi-orthogonal bases used in computer graphics and volume rendering for their strict local support, smooth shape, symmetry, and fast decay in the frequency domain [S1, unofficial]. They solve the problem of constructing hierarchical multiresolution representations that allow for efficient compression and detail culling while maintaining high visual fidelity [S1, unofficial]. What sets them apart from orthogonal wavelets is their semi-orthogonality, which allows for compactly supported dual basis functions, enabling linear time complexity transform algorithms, unlike general bi-orthogonal wavelets where duals may not be compactly supported [S1, unofficial].

## Shared principles
This family builds on [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md). The core mechanism relies on constructing a hierarchy of approximation spaces $V_m$ and detail spaces $W_m$ using scaling functions $\phi$ and wavelets $\psi$. The construction is defined by an order parameter $j$, which determines the polynomial degree and smoothness.

### Cardinal B-spline Scaling Functions
The foundation is the cardinal B-spline of order $j$, denoted $N^j(x)$, which is a piecewise-polynomial spline function with equally spaced simple knots [S1, unofficial].

**Base Case (Haar Scaling Function):**
For $j=1$, the scaling function is the box function:
$$
N^1(x) := \begin{cases} 
1 & \text{if } 0 \leq x \leq 1 \\
0 & \text{otherwise}
\end{cases}
$$
[S1, unofficial]

**Recurrence Relation:**
For $j \geq 2$, higher-order B-splines are constructed via self-convolution of the previous order:
$$
N^j(x) = (N^{j-1} * N^j)(x) = \int_0^1 N^{j-1}(x-t) dt = \frac{x}{j-1} N^{j-1}(x) + \frac{j-x}{j-1} N^{j-1}(x-1)
$$
[S1, unofficial]

**Properties:**
*   **Support:** The support of $N^j(x)$ is bounded by $[0, j]$.
*   **Symmetry:** The scaling functions are symmetric with respect to the center of their support.
*   **Vanishing Moments:** The number of vanishing moments is equal to $j-1$, which makes them attractive for compression applications as they efficiently represent smooth data with few coefficients [S1, unofficial].

### B-spline Wavelet Construction
The wavelet $\psi^{[j]}(x)$ of order $j$ spans the orthogonal complement between two approximation spaces of scaling functions of different resolution, specifically $W_m = V_{m-1} \ominus V_m$ [S1, unofficial].

**Wavelet Formula:**
The wavelet is defined as a linear combination of scaled scaling functions:
$$
\psi^{[j]}(x) = \sum_{k=0}^{3j-2} q_{j,k} \phi^{[j]}(2x-k)
$$
[S1, unofficial]

**Coefficients:**
The coefficients $q_{j,k}$ are calculated using the scaling function of order $2j$:
$$
q_{j,k} = \frac{(-1)^k}{2^{j-1}} \sum_{l=0}^j \binom{j}{l} \phi^{[2j]}(k+1-l)
$$
[S1, unofficial]

**Properties:**
*   **Support:** The support of $\psi^{[j]}(x)$ is bounded by $[0, 2j-1]$.
*   **Symmetry:** Unlike the scaling functions, the wavelets are only symmetric for even $j$, but anti-symmetric for odd $j$ [S1, unofficial].

### Semi-Orthogonality and Duality
Cardinal B-spline wavelets are semi-orthogonal, a special case of bi-orthogonal wavelets [S1, unofficial].
*   **Dual Functions:** The dual functions $\tilde{\phi}$ and $\tilde{\psi}$ do not have compact support but exhibit fast exponential decay [S1, unofficial].
*   **Usage:** Due to the lack of compact support in the duals, the dual functions are usually used for decomposition (analysis), while the primal functions are used for reconstruction (synthesis) [S1, unofficial].
*   **Efficiency:** This semi-orthogonality enables the design of fast wavelet transform algorithms with linear time complexity, which is not generally possible with full bi-orthogonal wavelets where duals may not be compactly supported [S1, unofficial].

## Subtopics

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)
Focuses on the projection of detail coefficients for culling.

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)
Visualizes the tree structure for navigation.

### [Wavelet Tree](5390-wavelet-tree.md)
The hierarchical data structure storing coefficients.

### Endpoint-Interpolating B-spline MRA
A multi-resolution analysis on the bounded interval $L^2[0,1]$ using B-splines with multiple knots at the endpoints to handle boundaries without artificial discontinuities [S1, unofficial]. Standard zero-padding introduces discontinuities, and periodization defines the function on a torus; endpoint-interpolating bases provide a solution for bounded domains [S1, unofficial].

**Definition:**
The knot sequence $t_m^j$ on $[0,1]$ is defined with multiple knots at the boundaries:
$$
t_m^{-j+1} = t_m^{-j+2} = \dots = t_m^0 = 0
$$
$$
t_m^k = k 2^{-m}, \quad k = 1, \dots, 2^m - 1
$$
$$
t_m^{2^m} = t_m^{2^m+1} = \dots = t_m^{2^m+j-1} = 1
$$
[S1, unofficial]

**Scaling Functions:**
The scaling functions are defined as:
$$
B_{mp}^j(x) = N^j(2^m x - p), \quad p = 0, \dots, 2^m - j
$$
where $N^j(x)$ is the cardinal B-spline of order $j$. For indices $p = -j+1, \dots, -1$, $B_{mp}^j$ contains a multiple knot at 0, and for $p = 2^m-j+1, \dots, 2^m-1$, it contains a multiple knot at 1 [S1, unofficial]. These functions span the approximation space $V_m$ [S1, unofficial].

**Parameter:**
*   $j$: The order of the B-spline (e.g., cubic implies $j=4$) [S1, unofficial].

## Background
Cardinal B-spline wavelets were developed independently by Chui and Unser [S1, unofficial]. They are widely used in computer graphics and modelling due to their desirable properties for surface and volume representation [S1, unofficial]. The construction of the dual functions and the fast wavelet transform algorithm are discussed in detail in the foundational literature [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
