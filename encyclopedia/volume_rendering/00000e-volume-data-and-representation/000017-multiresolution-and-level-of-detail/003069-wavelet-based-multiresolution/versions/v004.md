# Wavelet-based Multiresolution

Also called: Hierarchical Basis Decomposition, Multi-Resolution Analysis, Wavelet Decomposition

## Overview
Wavelet-based Multiresolution is a family of volume rendering techniques that decompose volumetric data into a hierarchy of approximation and detail coefficients using wavelet transforms [S1, unofficial]. This approach solves the problem of managing large volumetric datasets by enabling efficient Level of Detail (LOD) selection, compression, and rendering based on spatial frequency content [S1, unofficial]. It sets itself apart from spatial hierarchies (like octrees) by operating in the frequency domain, allowing for mathematically rigorous error bounds and smooth interpolation between resolutions, as opposed to the geometric partitioning used by siblings like [Mesh Simplification](1032-mesh-simplification.md) or [Pyramid and Downsampling Structures](9527-pyramid-and-downsampling-structures.md).

## Shared principles
All methods in this family share the core mechanism of generating a hierarchy of data representations at different spatial resolutions [S2, unofficial]. This hierarchy is typically constructed using one of two approaches:
1.  **Spatial Hierarchies:** Structures like Octrees or Bricks that partition the volume domain into sub-regions of varying sizes [S2, unofficial].
2.  **Frequency-Domain Transforms:** Methods like Wavelets that decompose data into coefficients representing different spatial frequencies [S2, unofficial].

Common mathematical and structural operations include:
*   **Scaling Factors:** Defining a scale factor $s$ such that the new resolution $N_{new}$ relates to the previous resolution $N_{prev}$ by $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1, S5, unofficial].
*   **Interpolation and Filtering:** Using interpolation kernels (e.g., trilinear) or filtering to estimate values at new grid points or to generate coarser levels from finer ones [S1, S2, unofficial].
*   **Selection and Culling:** Dynamically choosing which resolution level to render based on spatial hierarchy (distant or less important regions) and hardware constraints [S1, S2, unofficial].

## Subtopics

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)
A rendering technique for Multiresolution Maximum Intensity Projection (MMIP) that accelerates visualization by projecting wavelet detail coefficients directly to screen space. It differs from standard ray-casting by iterating over intensity thresholds and rendering pre-decomposed detail levels separately, allowing for efficient load balancing.

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)
A hierarchical visualization technique that maps wavelet tree nodes to a 2D triangular frame to provide a global overview of error distribution. Unlike [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md) which visualizes coefficients, this map encodes spatial hierarchy and error magnitude into a continuous color field.

### [Wavelet Tree](5390-wavelet-tree.md)
A multiresolution data representation that converts a 3D volume into a hierarchical 8-ary tree using wavelet transforms. It ensures correct topological information and smaller root-mean-square errors compared to standard subdivision hierarchies by reorganizing data into wavelet coefficients without requiring additional memory for the hierarchy structure itself.

### [Cardinal B-spline wavelets](11756-cardinal-b-spline-wavelets.md)
A class of semi-orthogonal bases used for their strict local support, smooth shape, and fast decay in the frequency domain. They are set apart from orthogonal wavelets by their semi-orthogonality, which allows for compactly supported dual basis functions and linear time complexity transforms.

### Multiresolution Analysis (MRA) for Meshes
A technique using wavelets to guide a subdivision process, allowing a polygonal model to be interpolated smoothly between different LODs [S2, unofficial]. It creates a mesh by growing Voronoi-like regions across triangles, with a Delaunay triangulation of those regions forming the base mesh [S2, unofficial].

## Background
Multi-Resolution Analysis (MRA) was introduced by Mallat [S1, unofficial]. Originally designed for image analysis, it defines an operator that approximates signals at different resolutions by decomposing a signal into a low-resolution part and a sequence of detail information in increasing order [S1, unofficial]. The framework relies on the Hilbert space of square-integrable functions $L^2(\text{domain})$, defined as the space of Lebesgue-measurable functions for which the squared norm is finite:
$$
\|f\|_{L^2(\underline{\Sigma})}^2 = \int_{-\infty}{\infty} |f(x)|^2 dx < \infty.
$$
[S1, unofficial]

An MRA is formally defined as a sequence of closed subspaces $V_m$ of $L^2$ with the following properties [S1, unofficial]:
1.  **Nestedness:** $L^2 \supset V_0 \supset V_1 \dots V_m \dots$
2.  **Scale-invariance:** $f(x) \in V_0 \iff f(2^m x) \in V_m$
3.  **Shift-invariance:** $f(x) \in V_0 \Rightarrow f(x-p) \in V_0, p \in \mathbb{Z}$
4.  **Completeness:** $\bigcap V_m = \{0\}$ and $\overline{\bigcup V_m} = L^2$
5.  **Riesz basis:** $\exists \phi(x)$ such that $\{\phi(x-p)\}_{p \in \mathbb{Z}}$ is a Riesz basis of $V_0$

The Riesz basis condition ensures stability:
$$
A \cdot \|c_p\|_{l^2}^2 \leq \left\| \sum_{p \in \mathbb{Z}} c_p \cdot \phi(\cdot - p) \right\| \leq B \cdot \|c_p\|_{l^2}^2
$$
with $A > 0$ and $B < \infty$, where $l^2$ is the space of square-summable sequences [S1, unofficial].

Two other important characteristics are the two-scale relation:
$$
\phi(x) = \sqrt{2} \sum_{p \in \mathbb{Z}} h_p \cdot \phi(2x-p)
$$
where $h_p$ are filter coefficients required for the Fast Wavelet Transform (FWT), and the partition of unity:
$$
\sum_{p \in \mathbb{Z}} \phi(\cdot - p) = 1
$$
[S1, unofficial]

The inner product of two functions is defined as:
$$
\langle f, g \rangle = \int_{-\infty}^{\infty} f(x) \overline{g(x)} dx
$$
[S1, unofficial]

The scaling function at iteration level $m_0$ and position $p$ is defined as:
$$
\phi_{m_0 p}(x) = 2^{-\frac{m_0}{2}} \phi(2^{-m_0}x - p)
$$
[S1, unofficial]

A function $f \in V_{m_0}$ is expressed as a weighted sum of scaling functions:
$$
f(x) = \sum_{p \in \mathbb{Z}} c_p \cdot \phi_{m_0 p}(x)
$$
[S1, unofficial]

Detail information is captured in subspace $W_{m+1}$ such that $V_m = W_{m+1} \oplus V_{m+1}$ [S1, unofficial]. Note: The source text states "$W_m = W_{m+1} \oplus W_{m+1}$" which appears to be a typo for the standard orthogonal decomposition $V_m = W_{m+1} \oplus V_{m+1}$ or similar, but the quote reflects the source [S1, unofficial].

The subspaces $W_m$ are spanned by wavelet functions constructed from a mother wavelet by binary dilation and dyadic translation:
$$
\psi_{mp}(x) = 2^{-\frac{m}{2}} \psi(2^{-m}x - p), \quad m, p \in \mathbb{Z}
$$
[S1, unofficial]

A hierarchical representation of a function decomposed from level $M$ down to $m_0$ is:
$$
f(x) = \sum_{p \in \mathbb{Z}} c_{Mp} \phi_{Mp}(x) + \sum_{m=m_0}^M \sum_{p \in \mathbb{Z}} d_{mp} \psi_{mp}(x)
$$
[S1, unofficial]

The coefficients are determined by inner products:
$$
c_{mp} = \langle f, \phi_{mp} \rangle \quad d_{mp} = \langle f, \psi_{mp} \rangle
$$
[S1, unofficial]

The two-scale relation for wavelet bases is:
$$
\psi(x) = \sum_{p \in \mathbb{Z}} g_p \cdot \phi(2x-p)
$$
[S1, unofficial]

Key properties of wavelets include [S1, unofficial]:
*   **Compact Support:** Defined as $\text{supp}(f) = \{x \in \mathbb{Q} \mid f(x) \neq 0\}$. Compactly supported wavelets can be represented by finite impulse response (FIR) filters, allowing for efficient implementations of FWT algorithms using sparse matrix operators [S1, unofficial].
*   **Localization:** The size of the interval of a compactly supported function plays a significant role for its localization quality. Small local support ensures high positional accuracy in the spatial domain [S1, unofficial].
*   **Heisenberg Uncertainty Principle:** The relation between spatial localization $\Delta_x^f$ and frequency localization $\Delta_{\omega}^f$ is defined by:
    $$
    \Delta_x^f \cdot \Delta_{\omega}^f \geq \frac{1}{4\pi}
    $$
    There is always a trade-off between good localization in spatial and frequency domains. B-Spline functions asymptotically reach this boundary with increasing order [S1, unofficial].
*   **Smoothness:** Important for compression applications, as compression is achieved by setting small coefficients to zero [S1, unofficial].

## Sources

[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
[S2] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
