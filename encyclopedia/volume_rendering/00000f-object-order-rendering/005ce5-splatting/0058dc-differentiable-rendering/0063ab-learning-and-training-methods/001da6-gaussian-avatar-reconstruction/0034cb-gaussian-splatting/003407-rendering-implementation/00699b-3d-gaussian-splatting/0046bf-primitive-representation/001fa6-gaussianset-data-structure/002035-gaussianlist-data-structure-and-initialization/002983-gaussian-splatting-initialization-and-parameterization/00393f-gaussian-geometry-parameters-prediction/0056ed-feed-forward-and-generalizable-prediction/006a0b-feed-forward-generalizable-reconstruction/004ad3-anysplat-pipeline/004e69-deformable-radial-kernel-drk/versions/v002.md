# Deformable Radial Kernel (DRK)
Also called: DRK Splatting
## Overview
Deformable Radial Kernel (DRK) Splatting is a 3D scene representation method that replaces the fixed radial symmetry of Gaussian primitives with a flexible planar kernel defined by multiple radial basis functions [S1]. It solves the overparameterization and geometric fidelity limitations of Gaussian Splatting by allowing adaptive stretching, rotation, and sharp boundary control, enabling high-fidelity rendering with significantly fewer primitives [S1]. Unlike standard 2D Gaussian Splatting, which relies on smooth, elliptical distributions, DRK supports straight edges and sharp corners through L1/L2 norm blending and piecewise edge sharpening [S1].
## How it works
### Inputs and Outputs
**Inputs:**
- Camera parameters: Ray origin $\mathbf{r}_o$, ray direction $\mathbf{r}_d$, view up vector $\mathbf{R}_z$ [S1].
- Scene representation: A set of $N$ DRK primitives [S1]. Each primitive $i$ is defined by parameters $\Theta_i = \{\boldsymbol{\mu}_i, \mathbf{q}_i, \{s_{k,i}, \theta_{k,i}\}_{k=1}^K, \eta_i, \tau_i, o_i, \mathbf{sh}_i\}$ [S1].
- Image resolution and tile configuration: $16 \times 16$ tiles [S1].

**Outputs:**
- Final color image $\mathbf{C}$ and depth buffer [S1].

### Data Structures
Each DRK primitive $i$ stores:
- $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Position in world space [S1].
- $\mathbf{q}_i \in SO(3)$: Orientation quaternion, converted to rotation matrix $\mathbf{R}_i = [\mathbf{R}_{x,i}, \mathbf{R}_{y,i}, \mathbf{R}_{z,i}]$ [S1].
- $\{s_{k,i}, \theta_{k,i}\}_{k=1}^K$: $K$ radial basis lengths and polar angles, ordered $0 < \theta_{1,i} < \dots < \theta_{K,i} \le 2\pi$ [S1].
- $\eta_i \in (0, 1)$: Blending weight between L1 and L2 norms [S1].
- $\tau_i \in (-1, 1)$: Edge sharpening coefficient [S1].
- $o_i \in (0, 1)$: Base opacity [S1].
- $\mathbf{sh}_i$: Spherical Harmonics coefficients for view-dependent appearance [S1].

### Step-by-Step Method

#### 1. Pre-computation: Calibrated Radial Lengths
For each radial basis $k$ of primitive $i$, compute the calibrated radial length $s_{ck,i}$ to determine the kernel boundary for culling [S1]:
$$ s_{ck,i} = s_{k,i} \sqrt{ -\log \left( \Psi^{-1}\left( \frac{e^{-9}}{o_i} \right) \right) } $$
Calculate the endpoint coordinates $\mathbf{v}_{k,i}$ in world space:
$$ \mathbf{v}_{k,i} = \boldsymbol{\mu}_i + s_{ck,i} (\cos(\theta_{k,i}) \mathbf{R}_{x,i} + \sin(\theta_{k,i}) \mathbf{R}_{y,i}) $$
These endpoints define the polygonal boundary of the kernel in the tangent plane [S1].

#### 2. Rasterization: Tile Culling and Sorting
1. Divide the image into $16 \times 16$ tiles [S1].
2. For each tile, determine potentially intersecting primitives by projecting the kernel boundary polygon (defined by $\mathbf{v}_{k,i}$) to the image plane [S1].
3. For each candidate primitive, compute the distance $r_t$ from the camera to the intersection of the ray with the kernel's tangent plane:
   $$ r_t = \frac{(\boldsymbol{\mu}_i - \mathbf{r}_o)^T \mathbf{R}_{z,i}}{\mathbf{r}_d^T \mathbf{R}_{z,i}} $$
4. Maintain a sorted cache (array of length 8) of primitives for each tile, sorted by $r_t$ (ascending) [S1].
   - Insert new primitives into the cache using insertion sort [S1].
   - If the cache is full, pop the primitive with the smallest $r_t$ [S1].

#### 3. Pixel Rendering: Kernel Evaluation
For each pixel within a tile, and for each primitive in the sorted cache (until opacity contribution is negligible):

**A. Project to Local Tangent Coordinates**
Transform the pixel position $\mathbf{p}$ into the primitive's local tangent plane coordinates $(u, v)$:
1. Compute vector from center to pixel: $\mathbf{d} = \mathbf{p} - \boldsymbol{\mu}_i$ [S1].
2. Project onto tangent basis:
   $$ u = \mathbf{d}^T \mathbf{R}_{x,i} $$
   $$ v = \mathbf{d}^T \mathbf{R}_{y,i} $$

**B. Compute Polar Coordinates**
$$ r = \sqrt{u^2 + v^2} $$
$$ \theta = \arccos\left(\frac{u}{r}\right) \quad \text{Handle } r=0 \text{ as } \theta=0 \text{ [G]} $$
*Note: The source defines $\theta = \arccos(u/r)$ [S1]. Standard `arccos` returns values in $[0, \pi]$. To cover the full $2\pi$ range required by the ordered angles, use `atan2(-v, u)` or flip the angle when $v < 0$ [G].*

**C. Determine Radial Basis Interval**
Find index $k$ such that $\theta_{k,i} < \theta \le \theta_{k+1,i}$. Wrap around if $\theta > \theta_{K,i}$ [S1].

**D. Compute Relative Angular Distance**
$$ \Delta \theta_k = \frac{(\theta - \theta_{k,i})\pi}{\theta_{k+1,i} - \theta_{k,i}} $$
*Note: This maps the sector angle to $[0, \pi]$ for cosine weighting [S1].*

**E. Compute L2 Norm Component**
$$ \bar{s}^2 = \frac{1 + \cos(\Delta \theta_k)}{2 s_{k,i}^2} + \frac{1 - \cos(\Delta \theta_k)}{2 s_{k+1,i}^2} $$
$$ r_2^2 = r^2 $$

**F. Compute L1 Norm Component**
Calculate the L1 distance $r_1$ for point $(u,v)$ between endpoints $\mathbf{e}_k = (s_{k,i} \cos \theta_{k,i}, s_{k,i} \sin \theta_{k,i})$ and $\mathbf{e}_{k+1} = (s_{k+1,i} \cos \theta_{k+1,i}, s_{k+1,i} \sin \theta_{k+1,i})$. Correcting the garbled Eq. 7 to match the L1 diamond-mapping description:
$$ r_1 = \left( \left| \frac{u}{s_{k,i} \cos \theta_{k,i}} \right| + \left| \frac{v}{s_{k,i} \sin \theta_{k,i}} \right| \right)^{-1} $$
*Note: $r_1^2$ requires no explicit scaling term as it is inherently scaled through the inverse transformation [S1].*

**G. Blended Kernel Value**
$$ g = \exp \left( - \frac{\eta_i r_1^2 + (1 - \eta_i) \frac{r_2^2}{2}}{\bar{s}^2} \right) $$

**H. Edge Sharpening**
Apply the piecewise sharpening function $\Psi(g)$ with coefficient $\tau_i$:
$$ \Psi(g) = \begin{cases} 
\frac{1-\tau_i}{1+\tau_i} g & \text{if } 0 \le g < \frac{1+\tau_i}{4} \\
g & \text{if } \frac{1+\tau_i}{4} \le g < \frac{3-\tau_i}{4} \\
\frac{1+\tau_i}{1-\tau_i} \left(g - \frac{3-\tau_i}{4}\right) + \frac{1+\tau_i}{4} & \text{if } \frac{3-\tau_i}{4} \le g \le 1
\end{cases} $$
Final opacity contribution:
$$ \alpha = o_i \cdot \Psi(g) $$

**I. Low-Pass Filtering**
To prevent aliasing, apply a low-pass filter approximation:
$$ \tilde{\alpha} = \max \left( \alpha, o_i \cdot \exp \left( - \frac{1}{2} \left( \frac{\Delta p_w^2}{s_l^2} + \frac{\Delta p_h^2}{s_l^2 |\mathbf{r}_d \cdot \mathbf{R}_{z,i}|^2} \right) \right) \right) $$
Where $\Delta p_w, \Delta p_h$ are image-space distances from the pixel to the projected kernel center, and $s_l$ is the filter radius [S1].

**J. Compositing**
Composite the color $\mathbf{C}_i$ (computed from SH and view direction) and opacity $\tilde{\alpha}$ using standard alpha blending:
$$ \mathbf{C}_{out} = \mathbf{C}_{in} + \tilde{\alpha} \mathbf{C}_i (1 - \mathbf{C}_{in}) $$
$$ \alpha_{out} = \alpha_{in} + \tilde{\alpha} (1 - \alpha_{in}) $$

### Parameters
- $K$: Number of radial bases. Default/Typical: Not specified, but 2D-GS is a special case with $K=4$ [S1].
- $\eta \in (0, 1)$: L1/L2 blend weight [S1].
- $\tau \in (-1, 1)$: Edge sharpness coefficient [S1].
- $s_l$: Filter radius. Not given in the sources.

### Edge Cases and Pitfalls
- **Division by Zero**: When $r=0$ (kernel center), $\theta$ is undefined. Set $\theta=0$ [G].
- **View Orthogonality**: If $\mathbf{r}_d \cdot \mathbf{R}_{z,i} \approx 0$, the kernel is viewed edge-on. The low-pass filter denominator becomes small, increasing the filter size to prevent aliasing [S1].
- **Degenerate Input**: If $\theta_k = \theta_{k+1}$, the sector is empty. Ensure $\theta_k$ are strictly increasing [S1].

## Operating characteristics
- **Time Complexity**: Rasterization is $O(N \cdot T)$ where $N$ is primitives and $T$ is tiles, but optimized by cache sorting (length 8) and culling [S1]. Per-pixel cost is proportional to the number of overlapping primitives in the cache (max 8) [S1].
- **Memory Complexity**: $O(N \cdot K)$ to store radial basis parameters [S1].
- **Performance**: DRK achieves superior rendering quality with significantly fewer primitives (e.g., 30 vs thousands for Gaussians) on detailed textures and sharp geometry [S1].
- **Strengths**: Better representation of sharp edges and straight lines; reduced overparameterization [S1].
- **Limitations**: More complex kernel evaluation than Gaussians; requires careful tuning of $\eta$ and $\tau$ [S1].

## Background
DRK Splatting was proposed by Yi-Hua Huang et al. in 2025 (arXiv:2412.11752v2) [S1]. It extends 2D Gaussian Splatting by introducing deformable radial bases to overcome the smoothness and symmetry constraints of Gaussians [S1]. It is backwards-compatible, reducing to a Gaussian kernel when using two orthogonal bases with specific decay rates [S1].

## Sources
[S1] Deformable Radial Kernel Splatting, https://doi.org/10.1109/CVPR52734.2025.02004, fetched 2026-10-30
