# Phong BRDF with Precomputed Integrals

Also called: Phong BRDF Shading with Precomputed Integrals, Precomputed Phong Shading

## Overview
This method accelerates the evaluation of the Phong BRDF model by precomputing the integrals of incoming radiance over the hemisphere, replacing slow per-pixel numerical integration with table lookups and interpolation [S1, book]. It is distinguished from standard Phong shading by its reliance on precomputed hemispherical integrals for both diffuse and specular components, and from [Phong BRDF Model](21654-phong-brdf-model.md) by its focus on runtime speed via tabulation rather than differentiable optimization of material parameters.

## How it works
**Inputs and Outputs:**
- **Inputs:** Incoming radiance field $I(\omega, k)$, surface normal $N$, view direction $V$, wavelength $k$, and Phong parameters $d(k)$, $s$, $c$.
- **Outputs:** Surface intensity $S(V, k)$.

**Data Structures:**
1.  **Diffuse Table:** A 3D texture or grid indexed by sampled surface normals $N$. Stores the precomputed integral $\int_{\Omega^+} I(\omega, k) \cos \theta \, d\omega$.
2.  **Specular Table:** A 3D texture or grid indexed by the vector $Q$, where $Q$ is the 180° rotation of $V$ about $N$. Stores the precomputed integral $\int_{S^2} I(\omega, k) \{-\omega \cdot Q\}^c \, d\omega$.

**Formulas:**
The Phong BRDF is defined as [S1, book]:
$$ f(\omega, V, N, k) \cos \theta = d(k) \cos \theta + s \{R \cdot V\}^c $$
Where:
- $\omega$ is the incoming light direction (opposite to incident ray).
- $V$ is the outgoing view direction.
- $N$ is the surface normal.
- $k$ is the wavelength.
- $\theta$ is the angle between $\omega$ and $N$.
- $d(k)$ is the wavelength-dependent diffuse reflection coefficient.
- $s$ is the wavelength-independent specular reflection coefficient.
- $R$ is the mirror reflection of $\omega$ about $N$ (180° rotation of $\omega$ about $N$).
- $c$ is the specular exponent.
- The term $\{R \cdot V\}$ is defined as $R \cdot V$ if $R \cdot \omega > 0$, and $0$ otherwise [S1, book].

The surface intensity $S(V, k)$ is the integral of incoming radiance weighted by the BRDF over the upper hemisphere $\Omega^+$:
$$ S(V, k) = \int_{\Omega^+} I(\omega, k) f(-\omega, V, N, k) \cos \theta \, d\omega $$
Splitting into diffuse and specular parts:
$$ S(V, k) = d(k) \int_{\Omega^+} I(\omega, k) \cos \theta \, d\omega + s \int_{\Omega^+} I(\omega, k) \{R \cdot V\}^c \, d\omega $$

**Precomputation Steps:**
1.  **Diffuse Precomputation:** Compute the integral $D(N, k) = \int_{\Omega^+} I(\omega, k) \cos \theta \, d\omega$ for a set of sampled normals $N$. Store in the Diffuse Table. Note that the same table can be used for diffuse transmission by using the opposite normal $-N$ [S1, book].
2.  **Specular Precomputation:** Define $Q$ as the 180° rotation of $V$ about $N$. Note that $\{R \cdot V\} = \{\omega \cdot Q\} = \{-\omega \cdot Q\}$. Replace the integral over the hemisphere with an integral over the full unit sphere $S^2$, assuming specular reflection from the back of the surface:
    $$ Specular(Q, k) = \int_{S^2} I(\omega, k) \{-\omega \cdot Q\}^c \, d\omega $$
    Store $Specular(Q, k)$ in the Specular Table for sampled $Q$ vectors [S1, book].

**Shading Step:**
1.  Retrieve $D(N, k)$ from the Diffuse Table via interpolation using the current surface normal $N$.
2.  Compute $Q$ by rotating $V$ 180° about $N$.
3.  Retrieve $Specular(Q, k)$ from the Specular Table via interpolation using $Q$.
4.  Compute final intensity:
    $$ S(V, k) = d(k) D(N, k) + s \cdot Specular(Q, k) $$
5.  Add direct sunlight contribution using the standard Phong BRDF if the point is not in shadow [S1, book].

**Parameters:**
-   $s$: Specular coefficient. Kept independent of wavelength $k$ to produce uncolored highlights, appropriate for waxy leaf coatings [S1, book].
-   $c$: Specular exponent. Large values are appropriate for glossy surfaces [S1, book].

**Edge Cases and Pitfalls:**
-   Extending the specular integral to the full sphere introduces error for glancing viewing rays where $Q$ is near the hemisphere boundary. This error is acceptable because Fresnel reflection law predicts brighter highlights in these cases anyway [S1, book].
-   The special dot product $\{R \cdot V\}$ must be zero if $R \cdot \omega \le 0$ [S1, book].

## Operating characteristics
**Time and Memory Complexity:**
-   **Time:** $O(1)$ per pixel for shading, replacing $O(N)$ integration.
-   **Memory:** $O(N_{samples})$ for the precomputed tables, where $N_{samples}$ is the number of sampled normals and $Q$ vectors.

**Measured Performance:**
-   Not given in the sources.

**Scaling:**
-   Not given in the sources.

**Strengths and Limitations:**
-   **Strengths:** Significant speedup for global illumination in vegetation by avoiding per-pixel integration [S1, book].
-   **Limitations:** Approximation error for glancing viewing rays in specular component; requires precomputation storage [S1, book].

**When to Use:**
-   When rendering scenes with many surfaces (e.g., vegetation) where Phong BRDF is appropriate and runtime speed is critical [S1, book].

**When Not to Use:**
-   When high accuracy for glancing angles is required without Fresnel correction [S1, book].

## Background
This method was developed for global illumination in vegetation, specifically for modeling the waxy outer coating of leaves [S1, book]. It builds on the Phong shading model [Bui] and uses precomputation techniques proposed by Miller and Hoffman [Mill] for diffuse reflection and by Cabral, Max, and Springmeyer [Cabr] for specular reflection [S1, book].

## Sources
[S1] Plane parallel radiance transport for global illumination in vegetation, https://www.osti.gov/biblio/649229, fetched 2026-10-26 (book)
