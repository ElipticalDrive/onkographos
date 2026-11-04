# Interval Arithmetic Inclusion Strategies

Also called: InclusionFunction strategies

## Overview
Interval Arithmetic Inclusion Strategies compute conservative bounds on an implicit function $f$ over a spatial domain, enabling the bisection trace to prune intervals that cannot contain the zero-level set [S1, unofficial]. They distinguish themselves from sibling methods by providing a configurable trade-off between generality (black-box compatibility) and bound tightness (performance/accuracy), spanning sampling-based estimates to analytic arithmetic [S1, unofficial].

## Shared principles
Every strategy constructs a conservative interval $[f](\mathcal{B}) = [f_-, f_+]$ guaranteeing that $f(\mathbf{x}) \in [f_-, f_+]$ for all $\mathbf{x} \in \mathcal{B}$ [S1, unofficial]. This bound is used to determine if the zero-level set (isosurface) can exist within the box $\mathcal{B}$ [G]. If $0 \notin [f_-, f_+]$, the box is pruned [G]. The strategies generally rely on a Lipschitz constant $\lambda$ or symbolic derivatives to bound the function's rate of change [G]. SamplingInclusion requires only point evaluations and relies on $\lambda$ to bound variation over the box diameter [S1, unofficial]. GradientInclusion improves tightness by incorporating $\nabla f$ directly [S1, unofficial]. NaturalIntervalInclusion and AffineInclusion operate on the symbolic formula of $f$ to produce progressively tighter bounds without sampling [S1, unofficial].

## Subtopics

### [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md)
Computes a conservative upper bound $\lambda$ on the rate of change of $f$ over a 3D bounding box. Choose this when you need a scalar bound for gradient-based or sampling-based inclusion, particularly for black-box functions where analytical gradients are unavailable.

### [Forward Linear Inclusion Function](1714-forward-linear-inclusion-function.md)
Computes valid, conservative bounds for a scalar field along a ray using first-order linear approximations. It sets itself apart by using linear bounds rather than quadratic ones, prioritizing computational simplicity. Choose this for basic sphere tracing or when gradient access is limited to local estimates.

## Background
The approach formalizes robust iso-surface rendering via interval-arithmetic bisection for arbitrary implicit geometries, primarily targeting black-box functions where only point evaluations are available [S1, unofficial]. The `isoray` library implements four distinct strategies: `SamplingInclusion`, `GradientInclusion`, `NaturalIntervalInclusion`, and `AffineInclusion` [S1, unofficial]. These strategies were developed to guarantee no surface features are missed (e.g., thin shells, cusps, high-frequency surfaces) while optimizing performance through tighter bounds when analytic formulas are known [S1, unofficial]. When only point evaluations are available, `SamplingInclusion` or `GradientInclusion` must be used, as the analytic strategies require symbolic formula access [S1, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
