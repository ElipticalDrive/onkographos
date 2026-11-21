# Unidirectional Differentiable Path Tracing

Also called: Estimating the interior integral

## Overview
Unidirectional Differentiable Path Tracing is an algorithm to estimate the interior integral of the differential path integral using unidirectional path tracing, computing both the radiometric measurement $L$ and its scene derivative $\dot{L}$ [S1, unofficial]. It sets itself apart from standard unidirectional path tracing by jointly estimating the measurement and its gradient with respect to scene parameters $\pi$ using automatic differentiation, while carefully separating PDF terms from the differentiation graph to ensure correctness [S1, unofficial]. It builds on [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md).

## How it works
**Inputs:** Two spatial points $\mathbf{x}_1, \mathbf{x}_2$ representing the endpoints of the boundary segment being evaluated.
**Outputs:** The radiometric measurement $L(\mathbf{x}_1 \to \mathbf{x}_2)$ and its scene derivative $\dot{L}$.

**Data Structures:**
*   Throughput scalars $T, \dot{T}$: Initialize to $(1, 0)$.
*   Accumulators $L, \dot{L}$: Initialize to $(L_e(\mathbf{x}_1 \to \mathbf{x}_2), \frac{\partial L_e(\mathbf{x}_1 \to \mathbf{x}_2)}{\partial \pi})$. Note: The derivative of the direct emission term is computed via autodiff on the full emission function [S1, unofficial].

**Steps:**
1.  Initialize $L \leftarrow L_e(\mathbf{x}_1 \to \mathbf{x}_2)$ and $\dot{L} \leftarrow \frac{\partial L_e(\mathbf{x}_1 \to \mathbf{x}_2)}{\partial \pi}$ [S1, unofficial].
2.  Initialize path throughput $T \leftarrow 1$ and its derivative $\dot{T} \leftarrow 0$ [S1, unofficial].
3.  Loop indefinitely:
    a.  **Direct Illumination (Light Sampling):**
        i.  Draw a light sample $\mathbf{p}_0 \sim P_{light}(\mathbf{p}_0)$ in area measure [S1, unofficial].
        ii. Compute the spatial point $\mathbf{x}_0 \leftarrow X(\mathbf{p}_0, \pi)$, where $X$ maps material points to spatial points given parameters $\pi$ [S1, unofficial].
        iii. Compute the value $\alpha_{direct}$ and its scene derivative $\dot{\alpha}_{direct}$ of the direct contribution term:
        $$ \alpha_{direct} = L_e(\mathbf{x}_0 \to \mathbf{x}_1) f_s(\mathbf{x}_0 \to \mathbf{x}_1 \to \mathbf{x}_2) G(\mathbf{x}_0 \leftrightarrow \mathbf{x}_1) J(\mathbf{p}_0) $$
        where $f_s$ is the BSDF, $G$ is the geometric term, and $J$ is the Jacobian of the transformation [S1, unofficial].
        iv. Update accumulators:
        $$ L \leftarrow L + \frac{T \alpha_{direct}}{P_{light}(\mathbf{p}_0)} $$
        $$ \dot{L} \leftarrow \dot{L} + \frac{T \dot{\alpha}_{direct} + \dot{T} \alpha_{direct}}{P_{light}(\mathbf{p}_0)} $$
        v. [S1, unofficial] Detach $P_{light}(\mathbf{p}_0)$ from the autodiff graph if using automatic differentiation to prevent incorrect gradients from omitted cosine factors in traditional implementations [S1, unofficial].
    b.  **Indirect Illumination (BSDF Sampling):**
        i.  Draw a direction $\boldsymbol{\omega}_i \sim P_{bsdf}(\boldsymbol{\omega}_i)$ in solid-angle measure [S1, unofficial].
        ii. Trace the ray: $\mathbf{x}_0 \leftarrow \text{rayTrace}(\mathbf{x}_1, \boldsymbol{\omega}_i)$ [S1, unofficial].
        iii. If $\mathbf{x}_0$ is valid (hit):
            1.  Compute the material point $\mathbf{p}_0 \leftarrow P(\mathbf{x}_0, \pi)$ [S1, unofficial].
            2.  Compute the value $\alpha_{indirect}$ and its scene derivative $\dot{\alpha}_{indirect}$ of the BSDF term:
            $$ \alpha_{indirect} = f_s(\mathbf{x}_0 \to \mathbf{x}_1 \to \mathbf{x}_2) G(\mathbf{x}_0 \leftrightarrow \mathbf{x}_1) J(\mathbf{p}_0) $$
            3.  Convert the BSDF probability to area measure:
            $$ q \leftarrow \frac{P_{bsdf}(\boldsymbol{\omega}_i) |\mathbf{n}(\mathbf{x}_0, \pi) \cdot -\boldsymbol{\omega}_i|}{\|\mathbf{x}_0 - \mathbf{x}_1\|^2} $$
            4.  Update throughputs:
            $$ T \leftarrow \frac{T \alpha_{indirect}}{q} $$
            $$ \dot{T} \leftarrow \frac{T \dot{\alpha}_{indirect} + \dot{T} \alpha_{indirect}}{q} $$
            5.  [S1, unofficial] Detach $q$ from the autodiff graph [S1, unofficial].
            6.  Shift points: $\mathbf{x}_2 \leftarrow \mathbf{x}_1, \mathbf{x}_1 \leftarrow \mathbf{x}_0$ [S1, unofficial].
        iv. Else (miss): Break loop [S1, unofficial].
4.  Return $(L, \dot{L})$ [S1, unofficial].

**Pitfalls:**
*   **Autodiff with Simplified Terms:** Traditional path tracers often omit cosine factors from $G$ because they cancel with PDFs. Applying automatic differentiation directly to such simplified implementations yields incorrect derivatives [S1, unofficial].
*   **Solution:** Ensure the full representation of $L_e$, $f_s$, and $G$ is differentiated. Design the renderer to completely separate measurement contribution and PDF terms, detaching the PDFs from the autodiff process [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is described in "Path-Space Differentiable Rendering" [S1, unofficial]. It is used to estimate the interior integral of the differential path integral, enabling the computation of scene derivatives for light paths starting with a boundary segment. It distinguishes itself from prior works by completely decoupling the boundary integral from its interior counterpart [S1, unofficial].

## Sources
[S1] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
