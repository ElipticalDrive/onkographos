# Affine-Transformations Method
Also called: Hybrid method, Image warping method

## Overview
The Affine-Transformations Method is a hybrid volume rendering technique that uses affine transformations to warp volume slices or the entire volume into the image plane, ensuring data samples become pixel-aligned before compositing [S1, unofficial]. It solves the problem of efficient visibility calculation by combining feed-forward image warping with feed-backward compositing, distinct from pure ray-casting and pure element-tossing methods [S1, unofficial]. It sets itself apart from its siblings by explicitly constructing affine parameters (rotation, scaling, translation) to align the sampling grid with the image pixels, rather than relying solely on shear-warp factorization or neural warp fields [S1, unofficial][S11626].

## Shared principles
The Affine-Transformations Method family shares the fundamental mechanism of applying an affine transformation to the volume data or its sampling grid to align the rendered slices with the image plane. This alignment allows for standard 2D compositing operations.

1.  **Affine Transformation Construction:** An affine transformation matrix $M$ is constructed to map coordinates from the volume space to the image-aligned space. For a rotation by angle $\theta$, the core rotation matrix $R$ is defined as:
    $$ R = \begin{bmatrix} \cos(\theta) & -\sin(\theta) & 0 \\ \sin(\theta) & \cos(\theta) & 0 \end{bmatrix} $$
    for 2D slices, or extended to 3D homogeneous coordinates for volumetric data [S11626, unofficial].

2.  **Centering and Offset:** To ensure rotation occurs around the volume's center rather than the origin, an offset vector $o$ is calculated:
    $$ o = c - R \cdot c $$
    where $c$ is the geometric center of the volume (e.g., `volume.shape / 2`) [S11626, unofficial]. The transformation applied to a point $x$ is $y = R \cdot x + o$ [S11626, unofficial].

3.  **Resampling and Interpolation:** The volume data is resampled using the transformed grid. Linear interpolation (order 1) is commonly used to sample the original volume at the transformed coordinates [S11626, unofficial]. Boundary conditions are handled by clamping or returning a default value (e.g., 0) for out-of-bounds samples [S11626, unofficial].

4.  **Compositing:** Once the data is pixel-aligned, the transformed slices are composited together to form the final image. This often involves converting the warped data into multiple datasets (substance, opacity, gradient) to generate shaded slices before compositing [S1, unofficial].

## Subtopics

### [Warp-Space Affine Mixture](4115-warp-space-affine-mixture.md)
A warping technique for neural volume rendering that computes affine mixture weights in the warped coordinate space. It sets itself apart by evaluating spatial weighting volumes after the point has been transformed, preventing warps from getting "stuck" and improving generalization on dynamic scenes.

### [Image Transformation Methods](10425-image-transformation-methods.md)
A forward projection method where volume data is transformed so rays can be projected into the volume and onto the image plane using a scan line method at any angle via Euler angle transformations.

### [Volume Rotation via Affine Transform](11626-volume-rotation-via-affine-transform.md)
This method rotates a 3D volume or 2D slice by applying an affine transformation to the sampling grid. It differs from sibling methods by explicitly constructing a rotation matrix and offset to warp the volume coordinates, rather than mixing warp-space representations.

### Drebin-Carpenter-Hanrahan Warping
This variant uses conventional two-pass image warping to resample volume elements in each slice so the resulting slices are pixel-aligned [S1, unofficial]. The warped data set is converted into multiple data sets: substance, opacity, gradient, and possibly mask [S1, unofficial]. These multiple data sets are used to generate shaded slices, which are then composited together to form the final image [S1, unofficial].

### World-Space Affine Mixture
A baseline warping variant where the affine mixture is performed in world space rather than warped space, meaning mixture weights are not warped before evaluation [S2]. The affine transform is $A_i(\mathbf{x}) = \mathbf{R}_i(\mathbf{s}_i \circ (\mathbf{x} - \mathbf{t}_i))$ and the normalized mixture weight is $a_i(\mathbf{x}) = \frac{w_i(\mathbf{x})}{\sum_j w_j(\mathbf{x})}$ [S2]. Since mixture weights are not warped, the decoder must match any motion of the template by moving the mixture weights, which is less efficient [S2].

## Background
The method originated in 1988 with Drebin, Carpenter, and Hanrahan at Pixar, who presented a feed-forward algorithm using image warping [S1, unofficial]. Hanrahan extended this in 1990 to a three-pass algorithm, developing affine-mapping equations and discussing the ordering of magnification and minification passes to avoid resampling artifacts [S1, unofficial]. It is used in direct volume rendering as a hybrid approach, distinct from pure feed-backward (ray-casting) and pure feed-forward (element-tossing) methods [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
