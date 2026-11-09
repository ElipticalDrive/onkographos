# Interval Arithmetic Inclusion Strategies

Also called: InclusionFunction strategies

## Overview
Interval Arithmetic Inclusion Strategies are a family of methods used within level-synchronized bisection ray casting to compute valid bounds for an implicit function $f$ over an axis-aligned box. These bounds determine whether a box can contain the zero-level set (surface), enabling the pruning of empty space and the calculation of safe step sizes. They are distinguished from sibling methods by their specific approach to bounding the function's variation: some rely on black-box gradient sampling, while others exploit symbolic analytic formulas for tighter constraints.

## Shared principles
All inclusion strategies in this family share the goal of computing an interval $[f](\mathcal{B}) = [f_-, f_+]$ such that for all $\mathbf{x} \in \mathcal{B}$, $f(\mathbf{x}) \in [f_-, f_+]$ [S1, unofficial]. This interval is used by the parent [Bisection Trace with Gradient Inclusion](1320-bisection-trace-with-gradient-inclusion.md) to decide if a box $\mathcal{B}$ intersects the surface (i.e., if $0 \in [f_-, f_+]$) and to determine the maximum possible deviation of $f$ from a sample point.

The strategies are categorized by their access to the function $f$:
1.  **Black-box strategies**: Require only point evaluations $f(\mathbf{x})$. They must estimate bounds using auxiliary information like the Lipschitz constant [S1, unofficial].
2.  **Analytic strategies**: Require access to the symbolic formula of $f$. They apply interval arithmetic directly to the expression tree of $f$ [S1, unofficial].

The choice of strategy involves a trade-off between generality (black-box compatibility) and the tightness of the resulting bounds, which directly impacts rendering performance [S1, unofficial].

## Subtopics

### [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md)
See [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md). This method estimates the maximum rate of change $\lambda$ of $f$. It is a prerequisite for black-box inclusion strategies like `SamplingInclusion` and `GradientInclusion` to bound the function's variation over a domain.

### [Forward Linear Inclusion Function](1714-forward-linear-inclusion-function.md)
See [Forward Linear Inclusion Function](1714-forward-linear-inclusion-function.md). This method uses the Lipschitz constant $\lambda$ and the value $v_0 = f(t_0)$ at a query point to define linear bounds $[f](t) = v_0 \pm \lambda(t - t_0)$. It serves as the bounding mechanism for the `SamplingInclusion` and `GradientInclusion` strategies.

### SamplingInclusion
A black-box baseline strategy that works with any function $f: \mathbb{R}^d \rightarrow \mathbb{R}$ where only point evaluations are available [S1, unofficial]. It requires a Lipschitz constant $\lambda$ to bound the function's variation over the box $\mathcal{B}$ [S1, unofficial]. It is less tight than gradient-based or analytic methods but is the only option when the symbolic formula is unknown [S1, unofficial].

### GradientInclusion
The default strategy for `isoray` [S1, unofficial]. It uses the gradient $\nabla f$ of the implicit function to compute tighter bounds than `SamplingInclusion` [S1, unofficial]. It requires access to the gradient of $f$, typically via automatic differentiation or analytical derivation, allowing it to constrain the function's behavior more tightly than simple Lipschitz sampling [S1, unofficial].

### NaturalIntervalInclusion
An analytic strategy that uses Natural Interval Arithmetic (NIA) on the symbolic formula of $f$ [S1, unofficial]. It provides tighter bounds than `SamplingInclusion` and `GradientInclusion` when the analytic formula is known [S1, unofficial]. It cannot be used if only black-box point evaluations are available [S1, unofficial].

### AffineInclusion
An analytic strategy that uses Affine Arithmetic (AA) on the symbolic formula of $f$ [S1, unofficial]. It provides the tightest bounds among the four strategies, thereby reducing the number of bisection steps needed for convergence [S1, unofficial]. Like `NaturalIntervalInclusion`, it requires access to the symbolic formula [S1, unofficial].

### Parallel Tape Reduction
A method that simplifies the SDF expression in specific regions of space by coupling pruning and rendering, using interval arithmetic to evaluate nodes in a construction tree [S2]. It relies on well-defined interval arithmetic queries for each node to determine pruning eligibility [S2]. It differs from Lipschitz pruning by using interval arithmetic instead of 1-Lipschitz bounds, requiring explicit interval queries per node rather than treating primitives as black boxes [S2]. This approach allows for efficient pruning of hard and smooth boolean operators, scaling to scenes with thousands of nodes [S2].

## Background
The `isoray` library introduced four specific inclusion strategies (`SamplingInclusion`, `GradientInclusion`, `NaturalIntervalInclusion`, `AffineInclusion`) for robust iso-surface rendering via interval-arithmetic ray casting [S1, unofficial]. The library targets black-box functions but offers analytic optimizations when symbolic formulas are available [S1, unofficial]. Parallel Tape Reduction was introduced in the context of hierarchical simplification of primitive-based SDFs to handle large scenes with thousands of nodes by coupling pruning and rendering [S2].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs, https://doi.org/10.1111/cgf.70057, fetched 2026-10-26
