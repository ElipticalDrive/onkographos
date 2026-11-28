# Boundary Conditions for Wavelet Transform on Intervals

Also called: Handling Finite Datasets in Wavelet Transforms, Wavelet Boundary Conditions

## Overview
This method defines how to apply wavelet transforms, originally formulated for the infinite real line $\mathbb{R}$, to finite datasets confined to the interval $[0, 1]$, which is a prerequisite for volume rendering applications using bounded data. It solves the problem of artificial discontinuities and basis function count inflation that arise when naively truncating the signal. It distinguishes itself from sibling methods by offering specific extension strategies—Spatial Windowing, Reflection, Repetition, and Multiple Knots—each imposing different continuity and symmetry constraints on the extended function $\tilde{f}(x)$.

## Shared principles
All methods address the mismatch between the unbounded support of standard wavelet bases $\phi_{mp}, \psi_{mp}$ and the bounded domain of the data $f(x)$. The core mechanism involves extending the original function $f(x)$, defined on $[0, 1]$, to an extended function $\tilde{f}(x)$ defined on $\mathbb{R}$ (or a sufficiently large subset thereof). The wavelet transform is then computed on $\tilde{f}(x)$ using the standard real-line basis functions.

Alternatively, this extension can be viewed as constructing a new set of basis functions on $[0, 1]$ by folding, periodizing, or modifying the original basis functions. The choice of extension determines the continuity class ($C^k$) of $\tilde{f}(x)$ at the boundaries $x=0$ and $x=1$, which directly impacts the frequency content of the transform coefficients.

## Subtopics

### Spatial Windowing
**What it is:** The simplest approach where the function is zero-padded outside the interval.
**Definition:**
$$ \tilde{f}(x) := \begin{cases} f(x) & \text{if } x \in [0, 1] \\ 0 & \text{else} \end{cases} $$
**Distinction:** Introduces discontinuities at $f(0)$ and $f(1)$ unless the data naturally vanishes at the boundaries. This causes high-frequency bursts in the signal spectrum.
**Consequence:** The number of relevant basis functions increases by $2^{-m}$ due to the unbounded decomposition structure on the real line interacting with the sharp edges [S1, unofficial].
**When to choose:** When computational simplicity is prioritized over spectral accuracy, or when the data is known to be zero at boundaries.

### Reflection
**What it is:** Extends the function by mirroring it across the boundaries.
**Definition:**
$$ \tilde{f}(x) := \begin{cases} \tilde{f}(-x) & \text{if } x < 0 \\ f(x) & \text{if } x \in [0, 1] \\ \tilde{f}(2-x) & \text{if } x > 1 \end{cases} $$
**Distinction:** Ensures that if $f(x)$ is continuous, $\tilde{f}(x)$ is at least $C^0$-continuous. This is equivalent to using folded wavelets:
$$ \phi_{mp}^{fold}(x) := \sum_i \phi_{mp}(x - 2i) + \sum_i \phi_{mp}(2i - x) $$
**Symmetry Cases:** The resulting biorthogonal system on $[0, 1]$ depends on the symmetry of the wavelets $\psi$ and $\tilde{\psi}$ with respect to $1/2$ [S1, unofficial]:
1.  **Symmetric:** If $\psi, \tilde{\psi}$ are symmetric, there are $2^{-m} + 1$ scaling functions and $2^{-m}$ wavelets.
2.  **Antisymmetric:** If $\psi, \tilde{\psi}$ are antisymmetric, there are $2^{-m}$ scaling functions and $2^{-m}$ wavelets.
**Constraint:** To ensure the support of the wavelet and scaling functions remains smaller than 1, the parameter $m$ must be chosen as $-2$ [S1, unofficial].
**Specific Counts (Cardinal Spline Basis, $m=-2$):**
*   Symmetric case ($j=4$): 5 scaling functions, 4 wavelets [S1, unofficial].
*   Antisymmetric case ($j=3$): 4 scaling functions, 4 wavelets [S1, unofficial].
**When to choose:** When preserving continuity at boundaries is critical and the data does not exhibit periodic behavior.

### Repetition
**What it is:** Extends the function periodically.
**Definition:**
$$ \tilde{f}(x) := \begin{cases} \tilde{f}(x+1) & \text{if } x < 0 \\ f(x) & \text{if } x \in [0, 1] \\ \tilde{f}(x-1) & \text{if } x > 1 \end{cases} $$
**Distinction:** Considered the most natural approach from a sampling theory perspective. It is equivalent to using periodized basis functions:
$$ \phi_{mp}^{period}(x) := 2^{-m/2} \sum_i \phi_{mp}(2^{-m}x + 2^{-m}i - p) $$
$$ \psi_{mp}^{period}(x) := 2^{-m/2} \sum_i \psi_{mp}(2^{-m}x + 2^{-m}i - p) $$
**Consequence:** The space $V_m^{period}$ and its complement $U_m^{period}$ are spanned by $2^{-m}$ functions each [S1, unofficial].
**When to choose:** When the data represents a periodic signal or when the boundaries are expected to match ($f(0) \approx f(1)$).

### Multiple Knots
**What it is:** A construction of B-spline based multiresolution analysis on a bounded interval by introducing multiple knots at the boundaries.
**Distinction:** Unlike the extension methods above which modify the data or basis via folding/periodization, this method modifies the basis definition itself to satisfy boundary conditions intrinsically.
**When to choose:** When using B-spline bases and requiring specific boundary interpolation properties without artificial extension artifacts.
**Link:** See [Endpoint-Interpolating B-spline MRA](11775-endpoint-interpolating-b-spline-mra.md) for the specific implementation using multiple knots.

## Background
The need for boundary conditions in wavelet transforms for volume rendering arises because standard wavelet bases are defined on $\mathbb{R}$. Volume data is inherently bounded. The methods described in [S1, unofficial] were developed to handle finite datasets efficiently while maintaining the multiresolution properties required for ray-tracing and compression. The distinction between symmetric and antisymmetric reflection cases highlights the trade-off between basis count and symmetry preservation in biorthogonal systems.

## Sources
[S1] Ray–tracing of multiresolution B–Spline volumes, https://doi.org/10.3929/ethz-a-006651431, fetched 2026-10-26 (unofficial)
