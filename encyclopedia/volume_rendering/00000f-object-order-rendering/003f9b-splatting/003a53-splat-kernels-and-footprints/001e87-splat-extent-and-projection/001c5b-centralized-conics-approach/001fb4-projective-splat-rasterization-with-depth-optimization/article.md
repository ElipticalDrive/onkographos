# Projective Splat Rasterization with Depth Optimization

Also called: Adjoint Matrix Splat Rasterization, Inverse Projective Mapping for Splatting

## Overview
A rasterization technique for point-based rendering that uses the inverse of a projective mapping matrix to transform screen coordinates back to object space for kernel evaluation, while simultaneously computing window-space depth values via linear interpolation of homogeneous coordinates [S1]. It sets itself apart from standard perspective splatting by avoiding explicit matrix inversion through the use of adjoint matrices and optimizing depth computation by embedding the window-space depth formula directly into the homogeneous interpolation weights [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Splat center $\mathbf{p}_k$, tangent vectors $\mathbf{s}_k, \mathbf{t}_k$, view frustum near/far constants $a, b$, and screen coordinates $(x, y)$ of the fragment [S1].
*   **Output:** Object-space coordinates $(u, v)$ for kernel evaluation and window-space depth $d(x)$ for z-buffering [S1].

**Data Structures:**
*   $\mathbf{M}_k$: A $3 \times 3$ matrix representing the forward projective mapping from parameter space $(u, v)$ to screen space [S1].
*   $\mathbf{N}_k$: A $4 \times 3$ matrix representing the scaled inverse mapping with the depth optimization row [S1].

**Steps:**

1.  **Compute Forward Mapping Matrix $\mathbf{M}_k$:**
    Construct the matrix from the splat's screen-space center and tangent vectors:
    $$ \mathbf{M}_k = \begin{bmatrix} \mathbf{s}_k & \mathbf{t}_k & \mathbf{p}_k \end{bmatrix} $$
    where $\mathbf{s}_k, \mathbf{t}_k, \mathbf{p}_k$ are column vectors forming the $3 \times 3$ matrix acting on $\begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$ [S1]. The forward mapping is defined as:
    $$ \begin{bmatrix} x_z \\ y_z \\ z \end{bmatrix} = \mathbf{M}_k \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} $$ [S1].

2.  **Compute Adjoint Matrix:**
    Calculate the adjoint of $\mathbf{M}_k$ using three cross products to avoid explicit matrix inversion [S1]:
    $$ \text{adj}(\mathbf{M}_k) = \begin{bmatrix} \mathbf{t}_k \times \mathbf{p}_k \\ \mathbf{p}_k \times \mathbf{s}_k \\ \mathbf{s}_k \times \mathbf{t}_k \end{bmatrix} $$ [S1].
    Let $\text{adj}(\mathbf{M}_k)_3$ denote the third row of this matrix [S1].

3.  **Compute Scaling Factor:**
    Calculate the scalar $\alpha$ to normalize the homogeneous coefficient $w$ such that it relates to the view-space depth $p_z$ (the $z$-component of $\mathbf{p}_k$) and the window-space depth constant $b$:
    $$ \alpha = \frac{b}{\text{adj}(\mathbf{M}_k)_3 \cdot \mathbf{p}_k} $$ [S1].
    Note that $\text{adj}(\mathbf{M}_k)_3 \cdot \mathbf{p}_k$ is the dot product of the third row of the adjoint matrix and the center vector $\mathbf{p}_k$ [S1].

4.  **Construct Inverse Mapping Matrix $\mathbf{N}_k$:**
    Scale the adjoint matrix by $\alpha$ to form the first three rows of $\mathbf{N}_k$ [S1]. To optimize depth computation, extend $\mathbf{N}_k$ to a $4 \times 3$ matrix by adding a fourth row $(\mathbf{N}_k)_4$:
    $$ (\mathbf{N}_k)_4 = \alpha \cdot \text{adj}(\mathbf{M}_k)_3 + \begin{bmatrix} 0 \\ 0 \\ a \end{bmatrix} $$ [S1].
    This construction ensures that the homogeneous weight $w$ in the inverse mapping equals $b/p_z$, allowing the fourth component to directly yield the window-space depth $d(x) = a + b/z$ via linear interpolation [S1].

5.  **Rasterize and Interpolate:**
    Rasterize the triangle covering the splat's bounding box [S1]. For each pixel $(x, y)$, compute the homogeneous coordinates by multiplying the inverse matrix $\mathbf{N}_k$ with the screen coordinate vector $\begin{bmatrix} x \\ y \\ 1 \end{bmatrix}$:
    $$ \begin{bmatrix} uw \\ vw \\ w \\ d \end{bmatrix} = \mathbf{N}_k \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} $$ [S1].
    The values $uw, vw, w$ (and $d$ if using the $4 \times 3$ matrix) are linearly interpolated across the primitive [S1]. The object-space coordinates are recovered by perspective division:
    $$ u = \frac{uw}{w}, \quad v = \frac{vw}{w} $$ [S1].
    The window-space depth is directly available as the fourth component $d$ when the $4 \times 3$ matrix is used, or computed as $a + b/z$ if only the $3 \times 3$ matrix is used [S1].

6.  **Kernel Evaluation:**
    Use the recovered $(u, v)$ to evaluate the splat's reconstruction kernel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projective_splat_rasterization(p_k, s_k, t_k, a, b, x, y):
    # Forward mapping matrix M_k = [s_k, t_k, p_k]
    M_k = column_stack([s_k, t_k, p_k])
    # Adjoint matrix rows are cross products
    adj_M = row_stack([cross(t_k, p_k), cross(p_k, s_k), cross(s_k, t_k)])
    # Scaling factor alpha ensures w = b/p_z at the center
    denom = dot(adj_M[2], p_k)
    alpha = b / denom
    # Construct 4x3 inverse mapping matrix N_k
    N_k = zeros((4, 3))
    N_k[:3, :] = alpha * adj_M
    N_k[3, :] = alpha * adj_M[2] + array([0.0, 0.0, a])
    # Compute homogeneous coordinates for screen point (x, y)
    coords = N_k @ array([x, y, 1.0])
    uw, vw, w, d = coords
    # Recover object-space coordinates
    u = uw / w
    v = vw / w
    return u, v, d
```

**Parameters:**
*   $a, b$: Constants derived from the near and far planes of the view frustum, mapping $1/z$ to the window space depth range $[0, 1]$ [S1].

**Edge Cases and Pitfalls:**
*   **Matrix Inversion Cost:** Explicitly inverting $\mathbf{M}_k$ is computationally expensive, which the method avoids by using the adjoint matrix [S1].
*   **Degenerate Splats:** If $\mathbf{p}_k, \mathbf{s}_k, \mathbf{t}_k$ are collinear or $\mathbf{p}_k$ is at the origin, the adjoint matrix may be singular or the scaling factor undefined [G]. In practice, check for near-zero denominators and skip rasterization or fall back to a safe default [G].
*   **Precision:** Linear interpolation of homogeneous coordinates is valid for projective mappings [S1]. However, ensure that $w$ does not become zero or negative during interpolation, which would cause division by zero or incorrect depth ordering [G]. Clip fragments where $w \le 0$ [G].

## Operating characteristics
Time and memory complexity are Not given in the sources [S1]. Measured performance with specific hardware, resolution, and scene data is Not given in the sources [S1]. Scaling behavior is Not given in the sources [S1]. Strengths include perspective accuracy, avoidance of expensive matrix inversion, and seamless integration with standard z-buffering via window-space depth interpolation [S1]. Limitations include the requirement for careful handling of homogeneous weights to avoid division by zero [G]. Performance may degrade if splats are very large or numerous, requiring efficient culling [G].

## Background
Originated in the context of point-based rendering, specifically for perspective-accurate splatting [S1]. The method is described in "Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering" [S1]. It addresses the need for efficient, perspective-correct rasterization of splats without the overhead of full 3D-to-2D projection inversion per pixel [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
