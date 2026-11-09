# Directed Distance Fields (DDF)


## Overview
Directed Distance Fields (DDF) are 5D neural field representations that map an oriented point (position $p$ and view direction $v$) to a binary visibility indicator $\xi(p, v)$ and a non-negative depth value $d(p, v)$ [S1]. This representation enables efficient differentiable geometric rendering with a single forward pass per directed geometric query, bypassing the iterative root-finding or multiple sampling steps required by standard implicit shapes like SDFs or NeRFs [S1]. Unlike its siblings, DDF explicitly decouples visibility and distance into a coupled tuple $(\xi, d)$ constrained by view-consistency conditions to ensure the 5D field corresponds to a valid 3D shape [S1].

## Shared principles
A Directed Distance Field (DDF) is a tuple of fields $F = (\xi, d)$ defined on the product space $\Gamma = B \times S^2$, where $B \subset \mathbb{R}^3$ is a convex compact bounding domain and $S^2$ is the unit sphere of directions [S1]. The field maps an oriented point $\tau = (p, v) \in \Gamma$ to:
1.  **Visibility** $\xi(p, v) \in \{0, 1\}$: A binary indicator where $\xi(p, v) = 1$ if the ray $\ell_{p,v}(t) = p + tv$ intersects the shape $S$ for some $t \ge 0$, and $0$ otherwise [S1].
2.  **Distance** $d(p, v) \in \mathbb{R}_{\ge 0}$: For visible rays ($\xi(p, v)=1$), this returns the minimum distance from $p$ to the surface $S$ along $v$ (i.e., the first intersection point) [S1].

### View Consistency and Constraints
The 5D nature of DDFs permits view-dependent geometric inconsistencies; therefore, specific constraints are enforced to ensure the field represents a valid 3D shape $S \subset B_\varepsilon$ [S1]. The $\varepsilon$-domain $B_\varepsilon$ is the set of points in $B$ at least distance $\varepsilon$ from the boundary $\partial B$ [S1].

**Visibility Field (VF) Constraints:**
The visibility field $\xi$ must be a Binary Open Zero (BOZ) field, meaning its zero set is open: if $\xi(\tau)=0$, there exists $\epsilon > 0$ such that $\xi(\tilde{\tau})=0$ for all $\tilde{\tau}$ in the open ball $B_\epsilon(\tau) \cap \Gamma$ [S1]. A VF satisfies three constraints:
1.  **Non-Visible Boundary Condition (BC$_\xi$):** $\xi(\tau) = 0$ for all $\tau \in O[B \setminus B_\varepsilon, B_\varepsilon]$, where $O[V, U]$ is the set of rays starting in $V$ that miss $U$. This ensures rays starting near the boundary and looking outward are non-visible [S1].
2.  **Directed Eikonal Constraint (DE$_\xi$):** $\xi$ is non-increasing along any ray: $f_\xi(s_1 | \tau) \le f_\xi(s_2 | \tau)$ for all $s_1 > s_2$. This ensures visibility transitions only from 'seeing' to 'not seeing' [S1].
3.  **Isotropic Opaqueness Constraint (IO$_\xi$):** If there exists a direction $v$ such that a "one-to-zero flip" occurs at point $q$ (defined as $\mathcal{F}_{\rightarrow, \xi, v}(q) = 1 \land \mathcal{F}_{\leftarrow, \xi, v}(q) = 0$), then $\xi(\tau) = 1$ for all $\tau \in I_q$ (all rays intersecting $q$). This makes flip positions 'isotropically opaque' [S1].

**Distance Field Constraints:**
The distance field $d$ is a Non-Negative Bounded (NNBC) field. A "Simple DDF" satisfies:
1.  **Positive Boundary Condition (BC$_d$):** $\inf_{v \in S^2} d(p, v) > 0$ for all $p \in B \setminus B_\varepsilon$ [S1].
2.  **Directed Eikonal Constraint (DE$_d$):** $\partial_s f_d(s | \tau) = -1$ along rays, except at along-ray zeroes where $\mathcal{A}_d(q, v)=0$. This enforces a distance-like gradient [S1].
3.  **Isotropic Opaqueness Constraint (IO$_d$):** If $\mathcal{A}_d(q) = 0$, then $\mathcal{A}_d(q, v) = 0$ for all $v$. If distance approaches zero from any direction, it must approach zero from all directions [S1].

**Compatibility:**
$\xi$ and $d$ are compatible if: (a) positional zeroes $Q_{d,\xi}$ are visible, and (b) every visible ray intersects a positional zero [S1]. A DDF is View Consistent (VC) if it is equivalent to a DDF induced by a shape $S$ [S1].

### Geometric Properties
*   **Directed Eikonal Equation:** For visible $(p, v)$, $\nabla_p d(p, v)v = -1$, implying $\|\nabla_p d(p, v)\|_2 \ge 1$ [S1].
*   **Surface Normals:** For visible $(p, v)$ where $n(p, v) \not\perp v$, the normal is $n(p, v) = -\nabla_p d(p, v)^T / (\nabla_p d(p, v)^T v)$ [S1]. Alternatively, $n(p, v) = \varsigma \nabla_p d(p, v)^T / \|\nabla_p d(p, v)\|_2$ where $\varsigma$ ensures $n^T v < 0$ [S1].
*   **Gradient Consistency:** $\nabla_v d(p, v) = d(p, v) \nabla_p d(p, v) P_v$, where $P_v = I - vv^T$ [S1].
*   **UDF Extraction:** $UDF(p) = \min_{v \in S^2} d(p, v)$ subject to $\xi(p, v)=1$ [S1].

### Rendering
Rendering is a single forward pass. Given camera position $p_0$ and pixel ray direction $v_\rho$:
1.  If $p_0 \in B$, query $d(p_0, v_\rho)$.
2.  If $p_0 \notin B$, compute intersection $p_r$ of ray with $\partial B$. Output $d(p_r, v) + \|p - p_r\|_2$ or set $\xi(p_r, v)=0$ if no intersection [S1].

## Subtopics

### [Intrinsic Appearance DDF (IADDF)](9285-intrinsic-appearance-ddf-iaddf.md)
Combines DDF geometry with INRs for appearance/lighting.

### DDF Composition
The ability to combine multiple DDFs into a single combined DDF for internal structure representation and compositionality, useful for manipulation without retraining [S1]. Given $N$ DDFs $\zeta = \{T^{(i)}, \xi^{(i)}, d^{(i)}, B^{(i)}\}_{i=1}^N$, where $T^{(i)}$ transforms world to object coordinates:
- **Visibility:** $\xi_\zeta(p, v) = 1 - \prod_k (1 - \xi^{(k)}(T^{(k)}(p, v)))$ [S1].
- **Depth:** $d_\zeta(p, v) = \sum_k a_\zeta^{(k)}(p, v) d^{(k)}(T^{(k)}(p, v))$ [S1].
- **Weights:** $a_\zeta^{(k)}(p, v) = \text{Softmax} \left( \left\{ \frac{\eta_T^{-1} \xi^{(k)}(T^{(k)}(p, v))}{\varepsilon_s + d^{(k)}(T^{(k)}(p, v))} \right\}_k \right)$, where $\eta_T$ is temperature and $\varepsilon_s$ is max inverse depth scale [S1].

### UDF Extraction via Minimal Direction Field
Extracting an Unoriented Distance Field (UDF) from a DDF by optimizing a field $v^*: B \to S^2$ such that $UDF(p) = d(p, v^*(p))$ [S1]. The Minimal Direction Field (MDF), $v^*$, points to the closest point on the surface $S$ [S1]. When normals exist, $v^*(p) = -n(p, v^*(p))$ [S1]. This requires optimizing an objective preferring high visibility and low depth, handling local minima and non-visible directions [S1]. Discontinuities in $v^*$ occur at surfaces and on the medial surface of $S$ [S1].

## Background
Directed Distance Fields were introduced in "Probabilistic Directed Distance Fields for Ray-Based Shape Representations" (CVPR 2022) by Aumentado-Armstrong et al. [S1]. The method was developed to address the inefficiency of standard implicit shapes (SDFs/occupancy) which require multiple network forward passes and complex calculations per pixel, and the topological constraints of meshes [S1]. DDFs are applied to 3D data extraction, fitting shapes, UDF extraction, composition, generative modelling, and single-image 3D reconstruction [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
