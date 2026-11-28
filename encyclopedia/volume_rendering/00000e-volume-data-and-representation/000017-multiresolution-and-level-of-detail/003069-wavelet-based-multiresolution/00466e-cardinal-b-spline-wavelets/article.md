# Cardinal B-spline wavelets

Also called: B-spline Wavelet Construction, B-spline wavelets, B-wavelet, Chui wavelets, Unser wavelets

## Overview
Cardinal B-spline wavelets are a class of semi-orthogonal wavelet bases used in computer graphics for volume data compression and rendering [S1, unofficial]. They are chosen when analytic expressions in both spatial and frequency domains are required for efficient splatting or filtering operations, offering strict local support, smooth shapes, symmetry, and fast decay in the frequency domain [S1, unofficial]. Unlike orthogonal wavelets, they employ semi-orthogonal duals to ensure perfect reconstruction while maintaining the smoothness and vanishing moments of B-splines [S1, unofficial].

## Shared principles
The family constructs a hierarchy of approximation spaces $V_m$ and detail spaces $W_m$ using cardinal B-splines of order $j$ [S1, unofficial]. The scaling functions $\phi^{[j]}$ are generated recursively from the Haar basis ($N^1(x) = 1$ for $0 \leq x \leq 1$, else $0$) via self-convolution:
$$N^j(x) = \frac{x}{j-1} N^{j-1}(x) + \frac{j-x}{j-1} N^{j-1}(x-1), \quad j \geq 2$$
[S1, unofficial]. The wavelet function $\psi^{[j]}$ spans the orthogonal complement between $V_m$ and $V_{m-1}$ and is defined as a linear combination of scaled and shifted scaling functions:
$$\psi^{[j]}(x) = \sum_{k=0}^{3j-2} q_{j,k} \phi^{[j]}(2x-k)$$
[S1, unofficial]. The coefficients $q_{j,k}$ are computed using:
$$q_{j,k} = \frac{(-1)^k}{2^{j-1}} \sum_{l=0}^j \binom{j}{l} \phi^{[2j]}(k+1-l)$$
[S1, unofficial]. The support of $\psi^{[j]}$ is bounded by $[0, 2j-1]$ [S1, unofficial]. Wavelets are symmetric for even $j$ and anti-symmetric for odd $j$ [S1, unofficial]. The number of vanishing moments equals $j-1$ [S1, unofficial]. Decomposition uses dual functions with fast exponential decay, while reconstruction uses the primary scaling and wavelet functions [S1, unofficial]. Three-dimensional extensions use non-standard tensor-product constructions [S1, unofficial].

## Subtopics

### [B-spline Wavelet Basis Construction](16466-b-spline-wavelet-basis-construction.md)
This method constructs the spatial and frequency-domain basis functions for Chui and Wang B-spline wavelets of order $j$, which are used for volume data compression and rendering. It differs from orthogonal wavelet bases by employing semi-orthogonal duals to ensure perfect reconstruction while maintaining the smoothness and vanishing moments of B-splines. It is chosen when analytic expressions in both spatial and frequency domains are required for efficient splatting or filtering operations.

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)

### [Wavelet Tree](5390-wavelet-tree.md)

### [Multi-Resolution Analysis (MRA) conditions](23140-multi-resolution-analysis-mra-conditions.md)

### Endpoint-Interpolating B-spline MRA
A multi-resolution analysis on the bounded interval $L^2[0,1]$ using B-splines with multiple knots at the endpoints to handle boundaries without artificial discontinuities [S1, unofficial]. It defines a knot sequence $t_m^j$ on $[0,1]$ with multiple knots at $0$ and $1$: $t_m^{-j+1} = \dots = t_m^0 = 0$ and $t_m^{2^m} = \dots = t_m^{2^m+j-1} = 1$ [S1, unofficial]. The scaling functions are defined as $B_{mp}^j(x) = N^j(2^m x - p)$ for $p=0,\dots,2^m-j$, where $N^j$ is the cardinal B-spline of order $j$ [S1, unofficial]. The parameter $j$ represents the order of the B-spline (e.g., cubic implies $j=4$) [S1, unofficial].

## Background
Cardinal B-spline wavelets were developed independently by Chui and Unser [S1, unofficial]. They are widely used in computer graphics and modelling [S1, unofficial]. The cardinal B-splines of order $j$ are piecewise-polynomial spline functions with equally spaced simple knots [S1, unofficial]. They are derived from each other by self-convolution of an initial basis of order 1 [S1, unofficial]. The construction of a corresponding B-wavelet of order $j$ spans the orthogonal complement between two approximation spaces of scaling functions of different resolution [S1, unofficial]. The number of vanishing moments of B-wavelets is equal to $j-1$, which makes them an attractive choice for compression applications [S1, unofficial]. Unlike orthogonal wavelets, B-spline wavelets are semi-orthogonal, allowing for compactly supported dual basis functions which enables linear time complexity transform algorithms, unlike general bi-orthogonal wavelets where duals may not be compactly supported [S1, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
