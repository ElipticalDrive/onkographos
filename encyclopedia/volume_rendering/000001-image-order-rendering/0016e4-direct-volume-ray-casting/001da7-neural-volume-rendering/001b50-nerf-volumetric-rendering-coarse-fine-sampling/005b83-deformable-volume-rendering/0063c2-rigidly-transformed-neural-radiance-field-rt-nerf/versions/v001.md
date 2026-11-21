# Rigidly Transformed Neural Radiance Field (RT-NeRF)
Also called: RT-NeRF

## Overview
RT-NeRF estimates a radiance field in an object's local coordinate system where density is constant with respect to local 3D location, conditioned on bone parameters [S1]. It differs from standard NeRF by explicitly incorporating rigid transformations and shape parameters as network inputs, allowing a single unified model to encode multiple parts or shape variations without part dependency [S1].

## How it works
**Inputs:**
*   $\mathbf{x}$: 3D location in world coordinates.
*   $\mathbf{d}$: 2D view direction in world coordinates.
*   $l = \{R, t\}$: Rigid transformation (rotation $R$, translation $t$).
*   $\xi$: 6D vector representing the se(3) transformation $l$, used as input for color prediction [S1].
*   $\zeta$: Bone parameter vector, used to condition the model on shape variations (e.g., limb length, thickness) [S1].

**Outputs:**
*   $\sigma$: Volume density.
*   $c$: View-dependent color.

**Data Structures:**
The method uses two neural networks (or one combined network with conditional inputs):
1.  $F_{\Theta_\sigma^l}^{l, \zeta}$: Maps local position and bone parameters to density and hidden features.
2.  $F_{\Theta_c^L}^{1, \zeta}$: Maps hidden features, local view direction, and transformation vector to color.

**Steps:**
1.  **Local Coordinate Transformation:** Transform the world 3D location $\mathbf{x}$ to the local object coordinate system $\mathbf{x}^l$ using the inverse rigid transformation:
    $$ \mathbf{x}^l = \mathbf{R}^{-1} (\mathbf{x} - \mathbf{t}) $$
    [S1]

2.  **Local View Direction Transformation:** Transform the world view direction $\mathbf{d}$ to the local object coordinate system $\mathbf{d}^l$:
    $$ \mathbf{d}^l = \mathbf{R}^{-1}\mathbf{d} $$
    [S1]

3.  **Density and Feature Prediction:** Encode the local position $\mathbf{x}^l$ and bone parameter $\zeta$ using positional encoding $\gamma(\cdot)$ and pass them through the density network to obtain volume density $\sigma$ and hidden features $\mathbf{h}$:
    $$ (\sigma, \mathbf{h}) = F_{\Theta_\sigma^l}^{l, \zeta} (\gamma(\mathbf{x}^l), \gamma(\boldsymbol{\zeta})) $$
    [S1]

4.  **Color Prediction:** Encode the hidden features $\mathbf{h}$, local view direction $\mathbf{d}^l$, and se(3) transformation vector $\xi$ using positional encoding $\gamma(\cdot)$ and pass them through the color network to obtain color $c$:
    $$ c = F_{\Theta_c^L}^{1, \zeta} (\mathbf{h}, \boldsymbol{\gamma}(\mathbf{d}^l), \boldsymbol{\gamma}(\boldsymbol{\xi})) $$
    [S1]

5.  **Combined Output:** The complete RT-NeRF function is expressed as:
    $$ F_\Theta^{l,\xi} : (\gamma(\mathbf{x}^l), \gamma(\mathbf{d}^l), \gamma(\boldsymbol{\xi}), \gamma(\boldsymbol{\zeta})) \rightarrow (c, \sigma) $$
    [S1]

**Parameters:**
*   $\xi$: A 6D vector representing the se(3) transformation $l = \{R, t\}$ [S1].
*   $\zeta$: Bone parameter vector for shape variation [S1].
*   $\gamma(\cdot)$: Positional encoding function (standard NeRF encoding implied) [S1].

**Edge Cases and Pitfalls:**
*   The method assumes the object's density is constant in its local coordinate system. If the object undergoes non-rigid deformation beyond the scope of $\zeta$, the model may fail [S1].
*   The se(3) representation $\xi$ is used directly as input; care must be taken to ensure the network can interpret this vector correctly [S1].

## Operating characteristics
Not given in the sources.

## Background
RT-NeRF was introduced in the "Neural Articulated Radiance Field" paper [S1]. It serves as the basic building block in neural articulated radiance fields, designed to overcome "Implicit Transformations" and "Part Dependency" issues by estimating the radiance field in an object's local coordinate system [S1].

## Sources
[S1] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
