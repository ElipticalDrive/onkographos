# Completion Framework

Also called: GPN Completion Framework

## Overview
The Completion Framework repairs and complements incomplete point clouds derived from RGB or RGB-D frames that only partially capture an object [S1]. It distinguishes itself from standard completion methods by using two separate encoders to produce distinct latent representations for the existing and missing parts of the point cloud, which are then concatenated and decoded to output the weights of a Neural Radiance Field (NeRF) [S1]. This approach generates a complete, colored point cloud by enforcing a Gaussian prior on the missing part's latent space while maintaining multi-view consistency with scanned images [S1].

## How it works
The framework maps an incomplete point cloud to a continuous parameterization of a 3D object space via NeRF weights [S1].

**Inputs and Outputs**
*   **Input:** An incomplete point cloud $P$ obtained from RGB/RGB-D scanning, split into two disjoint subsets: $P_e$ (existing parts) and $P_m$ (missing parts) [S1].
*   **Output:** The weights $\phi$ of a NeRF model $\Phi_\phi$, which render the complete point cloud and match the input cloud and existing scanning images [S1].

**Data Structures**
*   **Point Cloud Subsets:** $P_e$ and $P_m$ are disjoint subsets of the input point cloud $P$ [S1].
*   **Latent Representations:**
    *   $z_e = \varepsilon_e(P_e)$: Latent code for existing parts [S1].
    *   $z_m = \varepsilon_m(P_m)$: Latent code for missing parts [S1].
    *   Combined Latent: $z = z_e \oplus z_m$, where $\oplus$ denotes concatenation [S1].
*   **NeRF Weights:** $\phi = \Psi_\psi(z)$, where $\Psi_\psi$ is the decoder network [S1].

**Numbered Steps**
1.  **Split Input:** Divide the input point cloud $P$ into disjoint subsets $P_e$ (existing) and $P_m$ (missing) [S1].
2.  **Encode Existing Parts:** Process $P_e$ with encoder $\varepsilon_e$ to produce latent representation $z_e$ [S1].
3.  **Encode Missing Parts:** Process $P_m$ with encoder $\varepsilon_m$ to produce latent representation $z_m$ [S1].
4.  **Concatenate Latents:** Combine $z_e$ and $z_m$ into a single latent vector $z = z_e \oplus z_m$ [S1].
5.  **Decode NeRF Weights:** Pass $z$ through decoder $\Psi_\psi$ to output NeRF weights $\phi$ [S1].
6.  **Inference Optimization:** During inference, the encoder predicts an initial latent code and generates a unique NeRF model [S1]. Perform latent-based conditional optimization to refine the predicted NeRF model to match the reconstructed cloud and existing scanning images [S1].

**Formulas**
The training loss function combines a color rendering loss and a KL divergence term [S1]:

$$
L_{color} = \sum_{r \in R} ||\Phi_{\phi=\Psi_\psi(\varepsilon_e(P_e) \oplus \varepsilon_m(P_m))}(r) - C(r)||^2
$$

$$
L = L_{color} + D_{KL}(\varepsilon_e(P_e) \oplus \varepsilon_m(P_m), \mathcal{N}(0, I))
$$

Where:
*   $R$ is the set of rays [S1].
*   $\Phi_\phi$ is the NeRF model parameterized by weights $\phi$ [S1].
*   $C(r)$ is the color of ray $r$ from the input data [S1].
*   $D_{KL}$ is the Kullback-Leibler divergence [S1].
*   $\mathcal{N}(0, I)$ is the standard normal distribution, serving as the prior for the latent space [S1].

**Parameters**
*   **Latent Combination:** The operation $\varepsilon_e(P_e) \oplus \varepsilon_m(P_m)$ represents the concatenation of the two latent representations [S1].
*   **Prior:** The latent code for the missing part $z_m$ is forced to follow a Gaussian prior $\mathcal{N}(0, I)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def split_point_cloud(P, mask):
    return P[mask], P[~mask]

def encoder_e(P_e):
    # Architecture not specified in sources [G]
    return np.random.randn(latent_dim)

def encoder_m(P_m):
    # Architecture not specified in sources [G]
    return np.random.randn(latent_dim)

def concatenate(z_e, z_m):
    return np.concatenate([z_e, z_m], axis=-1)

def decoder(z):
    # Architecture not specified in sources [G]
    return np.random.randn(nerf_weight_dim)

def nerf_render(phi, r):
    # Standard volume rendering along ray r with weights phi [G]
    return np.random.rand(3)

def sum_squared_error(rendered, ground_truth):
    return np.sum((np.array(rendered) - np.array(ground_truth))**2)

def kl_divergence(z_m):
    # D_KL(z_m || N(0, I)) for VAE latent [G]
    return 0.5 * np.sum(z_m.var + z_m.mean**2 - 1 - np.log(z_m.var))

def optimize_neural_radiance_field(phi, rays, colors, point_cloud, lr=0.01, steps=100):
    for _ in range(steps):
        rendered = [nerf_render(phi, r) for r in rays]
        loss = sum_squared_error(rendered, colors)
        # Gradient update step [G]
        phi -= lr * np.gradient(loss, phi)
    return phi

def completion_framework_training(P_e, P_m, rays, colors):
    z_e = encoder_e(P_e)
    z_m = encoder_m(P_m)
    z_combined = concatenate(z_e, z_m)
    phi = decoder(z_combined)
    L_color = sum_squared_error([nerf_render(phi, r) for r in rays], colors)
    L_kl = kl_divergence(z_m)
    return L_color + L_kl

def completion_framework_inference(P_incomplete, mask):
    P_e, P_m = split_point_cloud(P_incomplete, mask)
    z_e = encoder_e(P_e)
    z_m = encoder_m(P_m)
    z_combined = concatenate(z_e, z_m)
    phi = decoder(z_combined)
    phi_refined = optimize_neural_radiance_field(phi, rays, colors, P_incomplete)
    return phi_refined
```

**Edge Cases and Pitfalls**
*   **Missing Part Definition:** The method assumes a predefined split of the point cloud into $P_e$ and $P_m$ [S1].
*   **Gaussian Prior Enforcement:** The requirement for $z_m$ to follow $\mathcal{N}(0, I)$ is critical for the generative aspect [S1].

## Subtopics

### [Partial Point Cloud Completion Optimization](19835-partial-point-cloud-completion-optimization.md)
This method optimizes an object-level latent code to reconstruct a complete 3D shape from a partial point cloud, using a pre-trained generative prior (ObjectNet) to infer missing geometry, followed by a patch-level refinement stage to recover high-frequency details. It distinguishes itself from global implicit field methods by decoupling the coarse shape completion (via the object prior) from local detail recovery (via patch latent codes), allowing for generalizable completion without per-scene network training.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on ShapeNet categories (cars and chairs) [S1]. The framework uniformly samples 16,384 colored points on each 3D model and uses 100 views for rendering [S1].
*   **Strengths:** Lightweight, generalizable, and capable of completing unseen portions of real-world objects with colors, unlike methods that only reconstruct synthetic objects [S1].
*   **Limitations:** Relies on the quality of the initial split between existing and missing parts [S1]. Performance depends on the ability of the encoders to disentangle and complement the latent representations [S1].
*   **When to Use:** Suitable for RGB/RGB-D scanning reconstruction applications where objects are partially captured and color completion is required [S1].
*   **When Not to Use:** Not suitable for complete point clouds where no completion is needed, or for applications requiring explicit mesh output rather than implicit NeRF representations [S1].

## Background
The Completion Framework is part of the GPN (Generative Point-Based NeRF) framework, proposed by Haipeng Wang et al. [S1]. It builds upon the "Generation Framework" and is inspired by Hyperpocket [S1]. The method addresses the limitation of existing point cloud completion techniques that rely on learning to reconstruct synthetic objects rather than completing the unseen portions of real-world objects [S1]. It was evaluated on standard shape generation benchmarks, specifically cars and chairs, using the ShapeNet dataset [S1].

## Sources
[S1] GPN: Generative Point-Based NeRF, https://doi.org/10.1007/978-3-031-82021-2_20, fetched 2026-10-30
