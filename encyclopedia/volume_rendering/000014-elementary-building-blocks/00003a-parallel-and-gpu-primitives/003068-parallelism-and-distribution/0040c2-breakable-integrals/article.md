# Breakable Integrals

Also called: Distributable Integral Functions

## Overview
Breakable integrals are a family of integral functions defined over a domain $[t_1, t_{N+1}]$ that can be decomposed into a sum of products of integrals over individual segments $[t_k, t_{k+1}]$. This decomposition enables the parallel or distributed computation of the total integral by allowing partial results from independent segments to be combined, facilitating distribution across multiple GPUs. Unlike simple parallel image compositing which assumes independent rays or sort-last approaches that require global sorting, breakable integrals allow for the accumulation of per-ray data across GPUs after independent segment processing, provided the integral structure supports this factorization.

## Shared principles
The core mechanism of breakable integrals relies on the ability to rewrite a global integral $\mathcal{F}(t_1 \rightarrow t_{N+1})$ as a sum of products of functions evaluated on sub-segments. This property allows the computation to be partitioned: each segment $[t_k, t_{k+1}]$ can be processed independently (e.g., on different GPUs), and the final result is obtained by combining these partial results using specific algebraic operations.

### General Form
A function $\mathcal{F}$ is a breakable integral if it can be expressed as:

$$
\mathcal{F}(t_1 \rightarrow t_{N+1}) = \sum_{i=1}^N \left( \prod_{j=1}^N H_{ij}(t_j \rightarrow t_{j+1}) \right)
$$

where $H_{ij}$ are functions derived from the integrals over the segments $[t_j, t_{j+1}]$. The specific structure of $H_{ij}$ depends on the integral being computed.

### Properties
The family exhibits closure under certain operations, which allows complex integrals to be proven as breakable by decomposing them into simpler breakable components [S1, unofficial]:

1.  **Property 1 (Algebraic Closure):** If $\mathcal{F}$ and $\mathcal{G}$ are breakable integrals, then their sum $\mathcal{F} + \mathcal{G}$, difference $\mathcal{F} - \mathcal{G}$, and product $\mathcal{F} \cdot \mathcal{G}$ are also breakable integrals.
2.  **Property 2 (Integration Closure):** If $\mathcal{F}(t_1 \rightarrow t)$ is a breakable integral with respect to the upper limit $t$, then the integral $\mathcal{A}(t_1 \rightarrow t_{N+1}) = \int_{t_1}^{t_{N+1}} \mathcal{F}(t_1 \rightarrow t) f(t) dt$ is also a breakable integral.

### Verification Step
To determine if a given integral function is a breakable integral, one must verify if it can be decomposed into the sum-product form over segments. This typically involves:
1.  Identifying if the integrand consists of terms that are themselves breakable (e.g., transmittance).
2.  Applying Property 1 to handle products and sums within the integrand.
3.  Applying Property 2 to handle the outer integration.

### Examples of Breakable Integrals

#### Transmittance
Transmittance is a fundamental breakable integral. Defined as:
$$
T(t_1 \rightarrow t) = \exp \left( - \int_{t_1}^t \sigma(s) ds \right)
$$
It can be decomposed over segments $[t_i, t_{i+1}]$ as:
$$
T(t_1 \rightarrow t_{N+1}) = \prod_{i=1}^N T(t_i \rightarrow t_{i+1})
$$
This corresponds to the general form where the product structure naturally emerges from the exponential of a sum of integrals.

#### Volume Rendering
The volume rendering equation is a breakable integral:
$$
C(t_1 \rightarrow t_{N+1}) = \int_{t_1}^{t_{N+1}} T(t_1 \rightarrow t) \sigma(t) c(t) dt
$$
Since $T(t_1 \rightarrow t)$ is a breakable integral (Property 2 applies to the integral of a breakable function multiplied by a source term), the entire volume rendering integral is breakable.

#### Distortion Loss
The distortion loss used in NeRF training is also a breakable integral. It is defined as:
$$
\mathcal{L}_{dist}(t_1 \rightarrow t_{N+1}) = 2 \int_{t_1}^{t_{N+1}} T(t_1 \rightarrow u) \sigma(u) S(t_1 \rightarrow u) du
$$
where the inner component $S$ is:
$$
S(t_1 \rightarrow u) = \int_{t_1}^u T(t_1 \rightarrow v) \sigma(v) (u - v) dv
$$
The proof of breakability proceeds as follows [S1, unofficial]:
1.  $T(t_1 \rightarrow v)$ is breakable.
2.  By Property 2, $S(t_1 \rightarrow u)$ is a breakable integral.
3.  By Property 1, the product $T(t_1 \rightarrow u) S(t_1 \rightarrow u)$ is a breakable integral.
4.  By Property 2 again, the outer integral $\mathcal{L}_{dist}$ is a breakable integral.

#### Decomposition of Volume Rendering
The volume rendering integral can be explicitly decomposed into two parts, $\mathcal{A}_1$ and $\mathcal{A}_2$, to facilitate parallel accumulation [S1, unofficial]:

$$
\mathcal{A}_1(t_1 \rightarrow t_{N+1}) = \sum_{k=1}^N \sum_{i=1}^{k-1} \prod_{j=1}^{k-1} H_{ij}(t_j \rightarrow t_{j+1}) \cdot \hat{H}_{ik}(t_k \rightarrow t_{k+1})
$$

$$
\mathcal{A}_2(t_1 \rightarrow t_{N+1}) = \sum_{k=1}^N \prod_{j=1}^{k-1} H_{kj}(t_j \rightarrow t_{j+1}) \cdot \hat{H}_{kj}(t_k \rightarrow t_{k+1})
$$

where:
$$
\hat{H}_{ik}(t_k \rightarrow t_{k+1}) = \int_{t_k}^{t_{k+1}} H_{ik}(t_k \rightarrow t) f(t) dt
$$
and similarly for $\hat{H}_{kj}$. Here, $H_{ij}$ represents the transmittance-like terms $T(t_j \rightarrow t_{j+1})$ and $f(t)$ represents the source term $\sigma(t)c(t)$.

## Subtopics

### Partitioned Volume Rendering
This is the primary application of breakable integrals in the context of scaling NeRFs. It involves dividing the ray domain $[t_n, t_f]$ into $N$ segments and computing the volume rendering integral over each segment independently on different GPUs. The partial results are then accumulated. This method leverages the breakable nature of the volume rendering equation to distribute the computational load. See [Partitioned Volume Rendering](link_to_child_if_exists) for implementation details.

### Distortion Loss Distribution
The distortion loss, critical for anti-aliasing in NeRFs, is also distributed using breakable integrals. The nested integral structure of the distortion loss is decomposed into segments, allowing the inner and outer integrals to be computed in parallel across GPUs. This ensures that the regularization term can be scaled without prohibitive memory or communication costs. See [Distortion Loss Distribution](link_to_child_if_exists) for implementation details.

## Background
The concept of breakable integrals was introduced in the context of scaling Neural Radiance Fields (NeRFs) to large-scale scenes using multiple GPUs. The term "breakable integrals" and the associated mathematical framework were presented in the paper "NeRF-XL: Scaling NeRFs with Multiple GPUs" by Barron et al. [S1, unofficial]. The work demonstrated that not only the volume rendering equation but also the distortion loss, which is more complex due to its nested integral structure, could be distributed across GPUs by exploiting the mathematical properties of these integrals. This approach avoids the need for global sorting or complex synchronization, relying instead on the algebraic decomposability of the integrals.

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26 (unofficial)
