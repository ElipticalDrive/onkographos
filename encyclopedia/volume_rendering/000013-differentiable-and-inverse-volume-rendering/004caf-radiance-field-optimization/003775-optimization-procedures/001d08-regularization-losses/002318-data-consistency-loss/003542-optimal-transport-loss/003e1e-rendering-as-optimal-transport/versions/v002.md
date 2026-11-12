# Rendering as Optimal Transport

Also called: OT-based Volume Rendering Formulation, Optimal Transport in Phase Space

## Overview
This method formulates volume rendering as an optimal transport problem in phase space, treating the renderer as a transport map that moves a source distribution of primitives to per-view pixel marginals [S1, unofficial]. It sets itself apart from standard rasterization by defining rendering as a conditional expectation of a joint coupling measure, allowing the entire representation formation process to be viewed as minimizing a transport cost subject to a complexity constraint [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Inputs:** A set of rendering primitives defined in phase space; a view $v$; parameters $\theta$; a target image resolution [S1, unofficial].
* **Outputs:** A rendered image $I_{v,\theta}$; a rendering coupling measure $\pi_{v,\theta}$ [S1, unofficial].

**Data Structures:**
* **Phase Space ($\mathcal{P}$):** Defined as the product of scene space $\mathcal{X} \subset \mathbb{R}^3$ (typically the unit ball) and the direction sphere $\Omega = S^2$ [S1, unofficial].
* **Primitives:** Defined as triples $(Q, w, \rho)$, where $Q$ is a cell in phase space, $w$ is an opacity weight, and $\rho$ is a value model (e.g., color/density) [S1, unofficial].
* **Coupling Measure ($\pi_{v,\theta}$):** A joint measure on $\mathcal{P} \times \text{Pixel}_v$ with source marginal $\mu_\theta$ (primitives) and pixel marginal $\nu_{v,\theta}$ [S1, unofficial].

**Steps:**
1. **Define Phase Space:** Construct $\mathcal{P} = \mathcal{X} \times \Omega$ [S1, unofficial].
2. **Construct Coupling:** For a given view $v$ and parameters $\theta$, define the joint measure $\pi_{v,\theta}$ on $\mathcal{P} \times \text{Pixel}_v$ [S1, unofficial]. For 3DGS-style compositing, this coupling follows the standard compositing rule: $\pi_{v,\theta}(p, \{i\}) \propto w_i(p) = \alpha_i(p) T_i(p)$, where $T_i = \prod_{j<i} (1-\alpha_j)$ [S1, unofficial].
3. **Render via Conditional Expectation:** Compute the pixel color $I_{v,\theta}(p)$ as the integral of the value model $\rho(x)$ over the phase space conditioned on pixel $p$:
   $$ I_{v,\theta}(p) = \int_{\mathcal{P}} \rho(x) \, d\pi_{v,\theta}(x \mid p) \quad (1) $$
4. **Optimize Representation:** Minimize the transport residual $\mathcal{R}_{\text{res}}(N)$ to find parameters $\theta$ that best match ground truth colors $c_\nu^{\text{gl}}(p)$ under a complexity budget $N$:
   $$ \mathcal{R}_{\text{res}}(N) = \min_{\theta} \sum_{\nu} \int_{\mathcal{P} \times \text{Pixel}_{\nu}} c \, d\mathfrak{p}_{\nu, \theta}, \quad c(x, p) := \left\| \rho(x) - c_{\nu}^{\text{gl}}(p) \right\|^2 \quad (2) $$
   Here, $\mathfrak{p}_{\nu, \theta}$ is the optimal transport plan between the rendered distribution and the target for view $\nu$ [S1, unofficial].

**Parameters:**
* **$N$:** The primitive count budget [S1, unofficial].
* **$Q$:** A quality target used to determine the minimal primitive count required [S1, unofficial].

**Edge Cases:**
* **Over-covering:** When Gaussians locally over-cover space, they may be merged into few representatives with bounded volumetric loss, certified by the Wang–Zahl union-volume estimate [S1, unofficial].

## Subtopics

### Capacity Floor
A measurable capacity floor derived from optimal transport in phase space that predicts the minimal achievable rendering error at a primitive count $N$ [S1, unofficial]. It answers how many primitives a scene actually needs based on Zador's law, stating that error falls as $N^{-2/d_{ps}}$, where $d_{ps}$ is the dimension of the scene's phase-space support [S1, unofficial]. It also provides an allocation law per tile: $N^*_j \propto K_j s_j^{d_j} / d_j$, where $N^*$ is the optimal primitive count, $K$ is a constant, $s$ is scale, and $d$ is dimension [S1, unofficial].

## Operating characteristics
* **Complexity:** The method provides a theoretical lower bound on error for a given primitive budget $N$, scaling as $N^{-2/d_{ps}}$ [S1, unofficial].
* **Strengths:** It offers a rigorous theoretical foundation for understanding redundancy and capacity in volume rendering, enabling "Budget-Centered Training" where models are trained at the optimal size rather than pruned later [S1, unofficial].
* **Limitations:** The formulation is theoretical; practical implementation requires mapping this transport cost to differentiable rasterization pipelines, which may introduce approximation errors [S1, unofficial].

## Background
This formulation was introduced in "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It builds on prior work using optimal transport in narrow roles, such as Wasserstein losses between Gaussian mixtures [S1, unofficial]. The theory relies on the resolution of the three-dimensional Kakeya set conjecture (Wang–Zahl union-volume estimate) to certify redundancy merging [S1, unofficial].

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
