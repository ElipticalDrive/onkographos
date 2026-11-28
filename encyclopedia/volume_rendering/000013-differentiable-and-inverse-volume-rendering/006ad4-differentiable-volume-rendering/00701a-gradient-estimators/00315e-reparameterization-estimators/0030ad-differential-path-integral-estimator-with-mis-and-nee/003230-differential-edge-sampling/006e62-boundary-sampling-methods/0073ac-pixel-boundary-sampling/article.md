# Pixel Boundary Sampling

Also called: Boundary Integral for Pixel Filters, Silhouette Integral Method

## Overview
Pixel Boundary Sampling is a technique in differentiable rendering that handles discontinuities at pixel filter boundaries by explicitly integrating over the boundary, allowing the use of box filters instead of Gaussian filters to reduce variance. It sets itself apart from alternatives like the unbounded Gaussian filter approach [Bangaru et al. 2020] by avoiding the extra variance introduced by the derivative of Gaussian weights in the divergence term.

## How it works
**Inputs and Outputs:**
*   **Input:** The pixel domain $\mathcal{U}$, the silhouette region $\mathcal{U}_{\text{sil}}$, the warp field $\mathcal{V}$, and the radiance $L$.
*   **Output:** The silhouette integral $I_{\text{sil}}$.

**Data Structures:**
*   $\mathcal{U}_b$: The pixel filter boundary, defined as the set of axis-aligned line segments bounding the pixel domain $\mathcal{U}$.
*   $\mathbf{n}_b$: The outward-pointing normal vector of the pixel filter boundary $\mathcal{U}_b$.

**Steps:**
1.  Define the unbounded space $\mathcal{U}_\infty$ containing the pixel domain $\mathcal{U}$.
2.  Split the integral domain into two parts: the region inside the pixel filter but outside the silhouette ($\mathcal{U} \setminus \mathcal{U}_{\text{sil}}$) and the region outside the pixel filter but excluding the boundary ($(\mathcal{U}_\infty \setminus \mathcal{U}) \setminus \mathcal{U}_b$).
3.  Apply the divergence theorem to the second area integral (the region outside the pixel filter) to convert it into a boundary integral over $\mathcal{U}_b$.
4.  Compute the final integral using the derived formula.

**Formulas:**
The silhouette integral $I_{\text{sil}}$ is initially split as:
$$I_{\text{sil}} = \int_{\mathcal{U} \setminus \mathcal{U}_{\text{sil}}} \nabla \cdot (L\mathcal{V}) + \int_{(\mathcal{U}_\infty \setminus \mathcal{U}) \setminus \mathcal{U}_b} \nabla \cdot (L\mathcal{V}) \quad (15)$$

Applying the divergence theorem to the second term yields:
$$I_{\text{sil}} = \int_{\mathcal{U} \setminus \mathcal{U}_{\text{sil}}} \nabla \cdot (L\mathcal{V}) - \oint_{\mathcal{U}_b} L(\mathcal{V} \cdot \mathbf{n}_b) \quad (16)$$

Where:
*   $I_{\text{sil}}$ is the silhouette integral.
*   $\mathcal{U}$ is the pixel domain.
*   $\mathcal{U}_{\text{sil}}$ is the silhouette region.
*   $\mathcal{U}_\infty$ is the unbounded space.
*   $\mathcal{U}_b$ is the pixel filter boundary.
*   $L$ is radiance.
*   $\mathcal{V}$ is the warp field.
*   $\mathbf{n}_b$ is the outward pointing normal of the pixel filter boundary.
*   The negative sign accounts for considering regions outside the pixel filter.

**Parameters:**
*   **Filter type:** Box filter. This is chosen to avoid the extra variance introduced by the derivative of Gaussian weights in the divergence [S1].

**Edge Cases and Pitfalls:**
*   **Gaussian Filter Variance:** Using an unbounded Gaussian filter to avoid boundary complexity introduces extra variance due to the variation in pixel filters in the divergence [S1].
*   **Boundary Discontinuity:** The product $L\mathcal{V}$ does not vanish smoothly at the pixel filter boundary $\mathcal{U}_b$, necessitating the explicit boundary integral [S1].
*   **Sampling $\mathcal{U}_b$:** Unlike silhouette boundaries in $\mathcal{U}_{\text{sil}}$, $\mathcal{U}_b$ is easy to sample since it only contains axis-aligned line segments of equal length [S1].

## Operating characteristics
Not given in the sources.

## Background
The method was introduced by Loubet et al. [2019] and Bangaru et al. [2020] in the context of differentiable rendering of neural SDFs through reparameterization. It addresses the complexity of handling pixel filter boundaries that arises when using box filters, which are preferred for their lower variance compared to Gaussian filters.

## Sources
[S1] Differentiable Rendering of Neural SDFs through Reparameterization, https://doi.org/10.1145/3550469.3555397, fetched 2026-10-26
