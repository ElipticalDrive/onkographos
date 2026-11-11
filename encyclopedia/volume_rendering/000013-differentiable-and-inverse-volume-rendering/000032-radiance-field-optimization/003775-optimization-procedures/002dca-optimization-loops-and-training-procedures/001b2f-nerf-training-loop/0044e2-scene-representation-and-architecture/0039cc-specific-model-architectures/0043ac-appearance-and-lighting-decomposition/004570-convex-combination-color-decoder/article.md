# Convex Combination Color Decoder

Also called: Color Decoder

## Overview
The Convex Combination Color Decoder is a method for predicting view-dependent appearance in neural radiance fields by blending a direct view-dependent color component with a reflection-based color component [S1]. It addresses the challenge of modeling glossy or reflective surfaces where standard view-dependent MLPs fail to capture consistent reflections across varying viewpoints [S1]. Unlike the standard NeRF color decoder which maps position and direction directly to color, this method explicitly separates the appearance into a base view-dependent term and a reflection term, weighted by a geometry-derived coefficient [S1].

## How it works
The decoder predicts the color $\mathbf{c}$ for a sample point $\mathbf{x}^{(i)}$ along a camera ray with viewing direction $\mathbf{d}$ [S1]. It relies on intermediate features from a geometry encoder, specifically a bottleneck vector $\mathbf{b}^{(i)}$ and a surface normal $\mathbf{n}^{(i)}$, as well as a reflection feature $\bar{\mathbf{f}}$ [S1].

**Inputs:**
*   $\mathbf{x}^{(i)}$: 3D position of the sample point.
*   $\mathbf{d}$: Viewing direction of the camera ray.
*   $\mathbf{b}^{(i)}$: Bottleneck feature vector, output by a small MLP applied to geometry features [S1].
*   $\mathbf{n}^{(i)}$: Surface normal at $\mathbf{x}^{(i)}$, output by the geometry encoder [S1].
*   $\bar{\mathbf{f}}$: Reflection feature, computed in 3D space (not image space) [S1].

**Outputs:**
*   $\mathbf{c}(\mathbf{x}^{(i)}, \mathbf{d})$: Predicted RGB color.

**Data Structures:**
*   $\beta^{(i)}$: A scalar weighting coefficient in $[0, 1]$, output by the geometry encoder followed by a sigmoid nonlinearity [S1].
*   $g(\cdot)$: A small Multi-Layer Perceptron (MLP) for the view-dependent color component [S1].
*   $h(\cdot)$: A small Multi-Layer Perceptron (MLP) for the reflection color component [S1].

**Steps:**
1.  Compute the weighting coefficient $\beta^{(i)}$ from the geometry encoder features using a sigmoid activation [S1].
2.  Compute the view-dependent color component $\mathbf{c}_v$:
    $$
    \mathbf{c}_v(\mathbf{x}^{(i)}, \mathbf{d}) = g(\mathbf{x}^{(i)}, \mathbf{b}^{(i)}, \mathbf{n}^{(i)}, \mathbf{d})
    $$
    where $g$ takes the position, bottleneck, normal, and viewing direction as inputs [S1].
3.  Compute the reflection direction $\mathbf{d}'$ based on $\mathbf{d}$ and $\mathbf{n}^{(i)}$ (standard specular reflection: $\mathbf{d}' = 2(\mathbf{d} \cdot \mathbf{n}^{(i)})\mathbf{n}^{(i)} - \mathbf{d}$) [S1].
4.  Compute the reflection color component $\mathbf{c}_r$:
    $$
    \mathbf{c}_r(\mathbf{x}^{(i)}, \mathbf{d}') = h(\mathbf{x}^{(i)}, \mathbf{b}^{(i)}, \mathbf{n}^{(i)}, \mathbf{d} \cdot \mathbf{n}^{(i)}, \mathbf{d}', \bar{\mathbf{f}})
    $$
    where $h$ takes the position, bottleneck, normal, dot product $\mathbf{d} \cdot \mathbf{n}^{(i)}$, reflection direction $\mathbf{d}'$, and reflection feature $\bar{\mathbf{f}}$ as inputs [S1].
5.  Compute the final color as a convex combination:
    $$
    \mathbf{c}(\mathbf{x}^{(i)}, \mathbf{d}) = \beta^{(i)} \mathbf{c}_v(\mathbf{x}^{(i)}, \mathbf{d}) + (1 - \beta^{(i)}) \mathbf{c}_r(\mathbf{x}^{(i)}, \mathbf{d}')
    $$
    [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_combination_color_decoder(x, d, b, n, f_bar, beta, mlp_g, mlp_h):
    """
    Predicts color using convex combination of view-dependent and reflection components.
    
    Args:
        x: Position vector (3,)
        d: Viewing direction vector (3,)
        b: Bottleneck feature vector
        n: Normal vector (3,)
        f_bar: Reflection feature vector
        beta: Weighting coefficient scalar in [0, 1]
        mlp_g: MLP for view-dependent color
        mlp_h: MLP for reflection color
        
    Returns:
        c: Predicted color vector (3,)
    """
    # Compute view-dependent color component
    # Inputs: position, bottleneck, normal, viewing direction
    c_v = mlp_g(x, b, n, d)
    
    # Compute reflection direction d' = 2(d . n)n - d
    dot_prod = np.dot(d, n)
    d_prime = 2 * dot_prod * n - d
    
    # Compute reflection color component
    # Inputs: position, bottleneck, normal, dot product, reflection direction, reflection feature
    c_r = mlp_h(x, b, n, dot_prod, d_prime, f_bar)
    
    # Convex combination
    c = beta * c_v + (1 - beta) * c_r
    
    return c
```

**Parameters:**
*   $\beta^{(i)}$: Output by geometry encoder + sigmoid. Typical value range $[0, 1]$ [S1].
*   MLPs $g$ and $h$: Described as "small MLPs" in the source [S1]. Specific layer counts and widths are not given in the main text, referring to a supplement [S1].

**Edge Cases and Pitfalls:**
*   **Fuzzy Geometry:** The method feeds reflected features $\bar{\mathbf{f}}$ into the color decoder at every sample along the ray, not just in image space. This is crucial for convergence when geometry is fuzzy during early optimization, as every point along the ray can utilize the reflected feature [S1].
*   **Distant Reflections:** Proper downweighting of distant content is crucial to prevent aliasing in reflections of distant illumination [S1].

## Subtopics

### Disentangled Appearance Modeling
A technique to handle inconsistent lighting, transient objects, or weather variations by disentangling scene content from appearance factors, enabling robust reconstruction from unstructured image collections [S2, unofficial]. It introduces learnable per-image latent appearance codes as input to the network alongside samples to explain per-view variability such as lighting, tone, or transient content [S2, unofficial]. This allows the core geometry and consistent aspects of the scene to be reconstructed while residual factors are absorbed by the appearance codes [S2, unofficial]. Extensions such as GRAF, HaNeRF, and RawNeRF further refine this idea [S2, unofficial]. Introduced in NeRF-in-the-wild [S2, unofficial].

## Operating characteristics
The method adds computational overhead due to the additional MLPs ($g$ and $h$) and the computation of reflection features and directions [S1]. By computing reflection features in 3D space and feeding them to the decoder at every sample, it ensures consistency but requires careful management of memory and computation along the ray [S1]. The use of a convex combination allows for smooth transitions between diffuse and glossy appearances based on the learned coefficient $\beta$ [S1].

## Background
The Convex Combination Color Decoder is inspired by UniSDF [S1]. It was introduced in NeRF-Casting to improve view-dependent appearance with consistent reflections [S1]. The geometry representation is based on Zip-NeRF, and it applies orientation loss similar to Ref-NeRF [S1].

## Sources
[S1] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
