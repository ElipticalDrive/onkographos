# UltraGauss Rendering

Also called: 3D Gaussian Splatting for Ultrasound, Ultrasound Gaussian Splatting

## Overview
UltraGauss is a rendering method for 3D ultrasound volumes that treats the ultrasound probe plane as a 3D intersection target rather than a 2D projection target [S1, unofficial]. It evaluates 3D Gaussians by intersecting them with the probe plane in 3D space using a 3D Mahalanobis distance to evaluate opacity, thereby avoiding the occlusion modeling required in visual spectrum rendering [S1, unofficial]. Unlike [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md) which supervises on voxel grids, UltraGauss renders directly to the 2D pixel grid of the probe by lifting 2D points to the probe's 3D coordinate frame [S1, unofficial].

## How it works
**Inputs:**
- A set of $K$ 3D Gaussians, each defined by mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$, color $c_i \in \mathbb{R}^3$, and opacity $\alpha_i \in [0, 1]$.
- The probe's transformation matrix $W$ mapping world coordinates to probe coordinates.
- The probe image dimensions (width, height).
- Background color $c_{BG}$ and background opacity coefficient $\alpha_{BG}$.

**Outputs:**
- Rendered color $\hat{c}(x) \in \mathbb{R}^3$ and opacity $\hat{\alpha}(x) \in \mathbb{R}$ for each pixel $x$ in the probe image.

**Data Structures:**
- **Gaussian Attributes:** Stored in arrays of size $K$.
  - $\mu$: $K \times 3$ float32.
  - $L$: $K \times 6$ float32 (lower-triangular elements of the Cholesky-like factor, see below).
  - $c$: $K \times 3$ float32.
  - $\alpha$: $K$ float32 (sigmoid-activated).
- **Pixel Accumulators:** Two buffers of size $W \times H$ (probe width $\times$ height).
  - `color_accum`: $W \times H \times 3$ float32.
  - `opacity_accum`: $W \times H$ float32.

**Covariance Parameterization:**
To ensure positive-definiteness and efficient inversion, $\Sigma$ is parameterized using a lower-triangular matrix $L$ such that $\Sigma^{-1} = LL^T$ [S1, unofficial]. The matrix $L$ is constructed from 6 learnable parameters $L_{11}, L_{12}, L_{13}, L_{22}, L_{23}, L_{33}$ and a small regularization constant $\beta > 0$:
$$
L = \begin{bmatrix} L_{11}^2 + \beta & 0 & 0 \\ L_{12} & L_{22}^2 + \beta & 0 \\ L_{13} & L_{23} & L_{33}^2 + \beta \end{bmatrix}
$$
Note: The source text contains a typo in the matrix indices in Eq. 11 ($L_{12}$ repeated in row 2 col 3 position in the text description vs standard lower triangular). Based on the standard lower-triangular structure and the need for 6 parameters for a $3 \times 3$ lower triangular matrix, the correct layout is:
$$
L = \begin{bmatrix} L_{11}^2 + \beta & 0 & 0 \\ L_{12} & L_{22}^2 + \beta & 0 \\ L_{13} & L_{23} & L_{33}^2 + \beta \end{bmatrix}
$$
The source text explicitly writes:
$$
L = \begin{bmatrix} L_{11}^2 + \beta & 0 & 0 \\ L_{12} & L_{22}^2 + \beta & 0 \\ L_{13} & L_{12} & L_{33}^2 + \beta \end{bmatrix}
$$
However, the text also states "Lij in Eq. 11 are uniformly randomly initialized between 4, 5)". A $3 \times 3$ lower triangular matrix has 6 unique elements. The text's matrix shows $L_{12}$ in position (2,1) and (3,2). This is likely a typo in the source paper's equation rendering where $L_{23}$ was intended at (3,2). We follow the standard lower-triangular definition for implementation correctness, using 6 distinct parameters [S1, unofficial].

**Algorithm Steps:**

1.  **Transform Gaussians to Probe Frame:**
    For each Gaussian $i$, transform its mean and covariance to the probe's coordinate frame using the inverse of the probe's world-to-probe transform matrix $W$ [S1, unofficial].
    $$ \mu_i^{3D} = W^{-1} \mu_i $$
    $$ \Sigma_i^{3D} = W \Sigma_i W^T $$
    Since $\Sigma_i = (LL^T)^{-1}$, we compute $\Sigma_i^{3D}$ and its inverse factor. Note that $\Sigma_i^{3D}$ is not necessarily lower-triangular in the probe frame, but its inverse can be computed. However, the source states $\Sigma_i^{3D} = W \Sigma_i W^T$. To maintain the efficient inversion property, the source implies we work with the transformed covariance. The opacity formula uses $(\Sigma_i^{3D})^{-1}$.
    *Correction:* The source says $\Sigma_i^{3D} = W \Sigma_i W^T$. It does not re-factorize $L$ in the probe frame. It uses the inverse of the transformed covariance. For efficiency, one might precompute $(\Sigma_i^{3D})^{-1}$ or keep $L$ in world space and transform the query point. The source formula (9) says $\mu_i^{3D} = W^{-1} \mu_i$ and $\Sigma_i^{3D} = W \Sigma_i W^T$. Then Eq. 8 uses $(\Sigma_i^{3D})^{-1}$.
    Implementation detail: Compute $\Sigma_i^{3D}$ explicitly or compute $(\Sigma_i^{3D})^{-1} = (W \Sigma_i W^T)^{-1} = (W^T)^{-1} \Sigma_i^{-1} W^{-1} = (W^T)^{-1} L L^T W^{-1}$. This is expensive.
    Re-reading Eq. 9: $\mu_i^{3D} = W^{-1} (W \mu_i)$? No, it says $\mu_i^{3D} = W^{-1} (W \mu_i)$ which is identity. The text says: "gaussian's parameters are moved to the probe's coordinate-frame using its inverse transform matrix W". And $\mu_i^{3D} = W^{-1} (W \mu_i)$ is likely a typo for $\mu_i^{3D} = W^{-1} \mu_i^{world}$? Or is $W$ the world-to-probe transform?
    Standard convention: If $W$ maps World $\to$ Probe, then $x_{probe} = W x_{world}$.
    The source says: "using its inverse transform matrix W". This is ambiguous. Let's look at Eq. 9: $\mu_i^{3D} = W^{-1} (W \mu_i)$. This simplifies to $\mu_i$. This is clearly a typo in the source text.
    Context: "lifting the 2D image point to 3D in the coordinate-frame of the probe".
    Let $x_{2D} = [x_1, x_2, 0]^T$ in probe coordinates.
    The Gaussian is in world coordinates.
    To evaluate distance, we need both in the same frame.
    Option A: Transform Gaussian to Probe Frame. $\mu^{probe} = W \mu^{world}$. $\Sigma^{probe} = W \Sigma^{world} W^T$.
    Option B: Transform Query Point to World Frame. $x^{world} = W^{-1} [x_1, x_2, 0]^T$.
    The source Eq. 8 uses $x_{|0} = [x_1, x_2, 0]^T$ and $\mu_i^{3D}$.
    Eq. 9 says $\mu_i^{3D} = W^{-1} (W \mu_i)$. This is nonsensical.
    However, Eq. 9 also says $\Sigma_i^{3D} = W \Sigma_i W^T$.
    If $\mu_i^{3D}$ is in probe frame, it should be $W \mu_i$.
    If $\Sigma_i^{3D}$ is in probe frame, it should be $W \Sigma_i W^T$.
    The text says "using its inverse transform matrix W". This suggests $W$ might be the *probe-to-world* matrix? If $W$ is probe-to-world, then $x_{world} = W x_{probe}$.
    Let's assume $W$ is the matrix that transforms Probe coordinates to World coordinates (common in some libraries).
    Then $\mu^{world} = W \mu^{probe}$. So $\mu^{probe} = W^{-1} \mu^{world}$.
    And $\Sigma^{world} = W \Sigma^{probe} W^T$. So $\Sigma^{probe} = W^{-1} \Sigma^{world} (W^{-1})^T$.
    The source Eq. 9 has $\Sigma_i^{3D} = W \Sigma_i W^T$. This matches $\Sigma^{probe} = W \Sigma^{world} W^T$ if $W$ is World-to-Probe.
    But then $\mu^{probe} = W \mu^{world}$.
    The source text for $\mu$ has $W^{-1}$.
    There is a contradiction in the source text's Eq. 9 regarding $\mu$.
    Given "inverse transform matrix W" in the text description, it is highly likely $W$ in the text refers to the World-to-Probe transform, but the formula for $\mu$ has a typo ($W^{-1}$ instead of $W$). Or $W$ is Probe-to-World, and the formula for $\Sigma$ has a typo.
    Standard practice in these papers: Transform Gaussians to Camera/Probe space.
    We will assume $W$ is the **World-to-Probe** transformation matrix.
    $\mu_i^{3D} = W \mu_i$
    $\Sigma_i^{3D} = W \Sigma_i W^T$
    The source text says "using its inverse transform matrix W". This is confusing. Let's look at the variable name. $W$. Usually $W$ is World.
    Let's stick to the geometric intent: Evaluate Mahalanobis distance in the probe frame.
    $x_{|0} = [x_1, x_2, 0]^T$ is in Probe Frame.
    $\mu_i^{3D}$ must be in Probe Frame.
    $\Sigma_i^{3D}$ must be in Probe Frame.
    If the input $\mu_i, \Sigma_i$ are in World Frame, and $T$ is World-to-Probe:
    $\mu^{probe} = T \mu^{world}$
    $\Sigma^{probe} = T \Sigma^{world} T^T$
    The source Eq. 9 writes $\Sigma_i^{3D} = W \Sigma_i W^T$. This implies $W=T$.
    The source Eq. 9 writes $\mu_i^{3D} = W^{-1} (W \mu_i)$. This is likely a typo for $W \mu_i$.
    We will implement $\mu_i^{3D} = W \mu_i$ and $\Sigma_i^{3D} = W \Sigma_i W^T$.

2.  **Compute Bounding Boxes:**
    For each Gaussian, compute the 3D bounding box of the ellipsoid encompassing $p\%$ of probability density [S1, unofficial].
    Calculate $\lambda = \frac{\chi_{3,1-p}^2}{\det(\Sigma_i^{3D})}$.
    For $p=95\%$, $\chi_{3,0.05}^2 = 7.815$.
    The bounding box limits are:
    $$ b^{\min}_j = \mu^{3D}_j - \sqrt{\lambda \cdot (\Sigma_i^{3D})_{jj}} $$
    $$ b^{\max}_j = \mu^{3D}_j + \sqrt{\lambda \cdot (\Sigma_i^{3D})_{jj}} $$
    *Correction:* Eq. 13 says $b^{\min/\max} = \mu^{3D} \pm \sqrt{\lambda v}$. The text says "using $\Sigma^{3D}_{jk}$ to index...". Usually, the axis-aligned bounding box of an ellipsoid $(x-\mu)^T \Sigma^{-1} (x-\mu) \le C$ is given by $\mu \pm \sqrt{C \cdot \text{diag}(\Sigma)}$.
    The source Eq. 13 is ambiguous about $v$. Standard derivation: The extrema of $x_j$ on the ellipsoid are $\mu_j \pm \sqrt{C \Sigma_{jj}}$.
    So $v$ is the diagonal element $\Sigma^{3D}_{jj}$.
    $b^{\min}_j = \mu^{3D}_j - \sqrt{\lambda \Sigma^{3D}_{jj}}$
    $b^{\max}_j = \mu^{3D}_j + \sqrt{\lambda \Sigma^{3D}_{jj}}$

3.  **Phase 1: Load Balancing / Rejection:**
    Iterate through all Gaussians.
    Check if the Gaussian's 3D bounding box intersects the probe plane (which is at $z=0$ in the probe frame, assuming the probe plane is the $xy$-plane).
    Condition: Reject if $b^{\min}_3 > 0$ or $b^{\max}_3 < 0$.
    Compact the list of accepted Gaussians into a contiguous buffer for efficient processing [S1, unofficial].

4.  **Phase 2: Rasterization:**
    For each accepted Gaussian $i$:
    a. Determine the 2D bounding box on the probe plane ($z=0$):
       $b^{\min}_{1,2} = \max(0, \lfloor b^{\min}_{1,2} \rfloor)$
       $b^{\max}_{1,2} = \min(W, \lceil b^{\max}_{1,2} \rceil)$
    b. Iterate over pixels $x = (x_1, x_2)$ within this 2D bounding box.
    c. Lift pixel to 3D probe frame: $x_{|0} = [x_1, x_2, 0]^T$.
    d. Compute opacity $\hat{\alpha}_i(x)$:
       $$ \hat{\alpha}_i(x) = \alpha_i \exp \left( -\frac{1}{2} (x_{|0} - \mu_i^{3D})^T (\Sigma_i^{3D})^{-1} (x_{|0} - \mu_i^{3D}) \right) $$
       To compute $(\Sigma_i^{3D})^{-1}$ efficiently, note that $\Sigma_i^{3D} = W \Sigma_i W^T$.
       $(\Sigma_i^{3D})^{-1} = (W^T)^{-1} \Sigma_i^{-1} W^{-1} = (W^T)^{-1} L L^T W^{-1}$.
       Let $v = W^{-1} (x_{|0} - \mu_i^{3D})$.
       Then $(x_{|0} - \mu_i^{3D})^T (\Sigma_i^{3D})^{-1} (x_{|0} - \mu_i^{3D}) = v^T L L^T v = \| L^T v \|^2$.
       This avoids explicit matrix inversion.
    e. If $\hat{\alpha}_i(x) > \epsilon$ (threshold):
       Atomically add to pixel accumulators:
       `opacity_accum[x] += \hat{\alpha}_i(x)`
       `color_accum[x] += \hat{\alpha}_i(x) * c_i`

5.  **Final Compositing:**
    For each pixel $x$:
    $$ \hat{\alpha}(x) = \text{opacity\_accum}[x] + \alpha_{BG} $$
    $$ \hat{c}(x) = \frac{1}{\hat{\alpha}(x)} \left( \text{color\_accum}[x] + \alpha_{BG} c_{BG} \right) $$
    [S1, unofficial]

**Parameters:**
- $\alpha_{BG}$: Small positive value to avoid division by zero. Value not specified, but "small" [S1, unofficial].
- $p$: Probability density percentage for bounding box. Typically 95% [S1, unofficial].
- $\beta$: Regularization constant in $L$ parameterization. Value not specified, but "small" [S1, unofficial].
- Initialization:
  - $\mu_i$: Uniformly at random within imaged volume boundaries [S1, unofficial].
  - $L_{ij}$: Uniformly randomly initialized between 4 and 5. This corresponds to diagonal elements $L_{jj}^2 + \beta \approx 16-25$. The source says "variance between 0.0017 and 0.0043". This implies the *resulting* covariance variance is small. If $L_{jj} \approx 4$, $L_{jj}^2 \approx 16$. $\Sigma^{-1}_{jj} \approx 16$. $\Sigma_{jj} \approx 1/16 = 0.0625$. This contradicts "variance 0.0017".
  Let's re-read: "Lij ... initialized between 4, 5), corresponding to a variance between 0.0017 and 0.0043".
  If variance $\sigma^2 \in [0.0017, 0.0043]$, then $\Sigma_{jj} \approx 0.003$.
  Then $\Sigma^{-1}_{jj} \approx 333$.
  Then $L_{jj}^2 \approx 333 \implies L_{jj} \approx 18$.
  The source text says "between 4, 5". This is a significant discrepancy.
  However, the source explicitly states "between 4, 5". We must follow the source text for the *initialization value*, even if the derived variance comment seems inconsistent with standard scaling. It is possible the units are different or the text has a typo. We cite the text's explicit range [S1, unofficial].
- Learning Rate: 0.05 for Gaussian parameters ($c, \alpha, L$). 0.00016 for means $\mu$ with exponential decay [S1, unofficial].
- Optimizer: Adam [S1, unofficial].
- Hardware: NVIDIA RTX A4000 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Numerical Instability:** Quaternion normalization for covariance parameterization causes instability in this setting. The lower-triangular parameterization with $\beta$ regularization is used instead [S1, unofficial].
- **Ill-conditioned Inversion:** Explicitly inverting $\Sigma^{-1}$ for resampling is numerically unstable. The lower-triangular factorization $L$ allows efficient inversion via forward substitution [S1, unofficial].
- **Occlusion:** Ultrasound rendering does not model occlusions in the same way as RGB. The probe plane intersection is the primary mechanism. Accumulated transmittance $T$ from standard splatting is incorrect for ultrasound [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(K \cdot A)$ where $K$ is the number of Gaussians and $A$ is the average number of pixels covered by a Gaussian's bounding box. The rejection phase reduces $K$ to active Gaussians.
- **Memory Complexity:** $O(K)$ for Gaussian attributes and $O(W \cdot H)$ for pixel accumulators.
- **Performance:** Supports up to 2 million Gaussians with respectable reconstruction times on a single NVIDIA RTX A4000 GPU [S1, unofficial].
- **Strengths:** Efficient rasterization via bounding box rejection and lower-triangular covariance factorization. Suitable for volumetric ultrasound data where occlusion is less critical than surface intersection.
- **Limitations:** Requires custom CUDA kernels for efficiency. Initialization is random, not from SfM.

## Background
UltraGauss was introduced in the paper "UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes" [S1, unofficial]. It adapts 3D Gaussian Splatting to the specific domain of 3D ultrasound, addressing the limitations of standard splatting which projects to 2D image space and models occlusion. The method was developed to handle volumetric ultrasound scans and freehand ultrasound video sequences [S1, unofficial].

## Sources
[S1] UltraGauss: Ultrafast Gaussian Reconstruction of 3D Ultrasound Volumes, https://doi.org/10.48550/arXiv.2505.05643, fetched 2026-10-27 (unofficial)
