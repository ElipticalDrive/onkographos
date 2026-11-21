# Geometry and Surface Priors

Also called: Surface-guided 3DGS, SDF-constrained Gaussian Splatting

## Overview
Geometry and Surface Priors address the inherent ambiguity of 3D Gaussian Splatting (3DGS), where discrete Gaussian primitives often form "floating ellipsoids" rather than coherent, watertight surfaces [S1, S2]. This family of methods enforces geometric consistency by coupling the Gaussian representation with explicit surface constraints, such as neural Signed Distance Functions (SDFs), unsigned distance functions (UDFs), or metric monocular depth estimates [S1, S2]. Unlike standard 3DGS which relies solely on photometric rendering losses, these methods explicitly guide Gaussian centers and orientations toward true surface geometry, improving mesh extraction quality and structural fidelity [S1, S2].

## Shared principles
All methods in this family share the goal of aligning the discrete Gaussian representation with a continuous geometric prior. The core mechanism involves adding geometric loss terms to the standard photometric rendering loss $\mathcal{L}_{rgb}$. These losses typically enforce two properties:
1.  **Positional Consistency:** Gaussian centers $\mathbf{c}_i$ are encouraged to lie on the zero-level set of a distance function (SDF or UDF) or align with metric depth estimates.
2.  **Normal Consistency:** The orientation of the Gaussians (derived from their covariance matrix or rotation quaternion) is aligned with the surface normal estimated from the geometric prior.

The total loss function generally takes the form:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
where $\mathcal{L}_{geo}$ is the geometric consistency loss and $\mathcal{L}_{norm}$ is the normal consistency loss [S1]. The weights $\lambda_{geo}$ and $\lambda_{norm}$ are hyperparameters that balance photometric fidelity against geometric accuracy [S1].

## Subtopics

### NeuSG
NeuSG integrates the NeuS neural surface reconstruction method into 3DGS to transfer explicit surface properties to the discrete Gaussian primitives [S1]. It solves the problem of vanilla 3DGS producing floating ellipsoids by enforcing geometric consistency with a neural Signed Distance Function (SDF) [S2]. Unlike sibling methods that may use monocular depth or post-hoc mesh extraction, NeuSG directly couples a NeuS-style SDF network with the 3DGS rasterizer to align Gaussian centers and normals with the implicit surface [S1].

**Mechanism:**
*   **SDF Network:** A multi-layer perceptron (MLP) $f_\theta(\mathbf{x})$ outputs signed distance values for 3D coordinates [S1].
*   **SDF Loss:** Encourages Gaussian centers to lie on the zero-level set:
    $$ \mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)| $$
*   **Normal Loss:** Aligns the Gaussian geometric normal $\mathbf{n}_i$ (derived from covariance/rotation) with the SDF surface normal $\mathbf{m}_i = \frac{\nabla f_\theta(\mathbf{c}_i)}{\|\nabla f_\theta(\mathbf{c}_i)\|}$:
    $$ \mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i) $$
*   **Optimization:** Jointly optimizes Gaussian attributes and SDF parameters $\theta$ [S1].

**When to choose:** When a high-quality, watertight mesh is required directly from the Gaussian representation without post-processing, or when the scene has complex topology that benefits from implicit surface constraints [S1].

### [Monocular Depth Scale Alignment](9183-monocular-depth-scale-alignment.md)
Monocular Depth Scale Alignment is a post-optimization step that resolves the inherent up-to-scale ambiguity of scenes reconstructed from multi-view images by aligning the optimized Gaussian geometry with metric-scale monocular depth estimates. Unlike sibling methods that may prune initial points or use different depth priors, this method fixes the optimized Gaussian parameters $\Theta$ and optimizes a single global scene scale factor $a$ to minimize the difference between rendered inverse depth and metric inverse depth, ensuring fair comparison with other studies that do not prune initial COLMAP points.

**Mechanism:**
*   **Input:** Optimized Gaussians and metric monocular depth maps $D^{\text{mono}}$ (e.g., from DepthAnythingV2).
*   **Inverse Depth:** Converts metric depth to inverse depth for numerical stability: $D_{\text{inv}} = 1/D$.
*   **Loss:** Minimizes $L_1$ loss between scaled rendered inverse depth and metric inverse depth:
    $$ \arg \min_{a} \frac{1}{N_{\mathcal{I}}} \sum_{i=1}^{N_{\mathcal{I}}} \left\| D^{\text{mono}}_{\text{inv}, i} - a \cdot D^{\text{rendered}}_{\text{inv}, i} \right\|_1 $$
*   **Update:** Scales Gaussian means $\mathbf{c}_i$ and scales $\mathbf{s}_i$ by the optimal factor $a^*$.

**When to choose:** When metric scale is required for downstream tasks (e.g., robotics, measurement) but the initial 3DGS reconstruction lacks metric scale due to the scale ambiguity of multi-view stereo.

### [Sparse-view 3D Gaussian Splatting with Depth Priors](9185-sparse-view-3d-gaussian-splatting-with-depth-priors.md)
This method addresses the challenge of 3D reconstruction from sparse-view inputs by integrating depth priors estimated by pre-trained 2D networks into the 3D Gaussian Splatting framework. It distinguishes itself from sibling methods like [NeuSG](9172-neusg.md) and [GSDF](9205-gsdf.md) by explicitly leveraging monocular depth estimation to supervise rendered depth, removing Gaussians with incorrect depth values, and utilizing Score Distillation Sampling (SDS) to ensure fidelity in novel viewpoints.

**Mechanism:**
*   **Depth Estimation:** Uses a pre-trained 2D network to estimate monocular depth $D_i$ for each input image.
*   **Depth Consistency:** Enforces consistency between rendered depth and estimated depth priors.
*   **Pruning:** Removes Gaussians with incorrect depth values relative to the estimated prior.
*   **SDS Loss:** Applies Score Distillation Sampling to encourage rendered results from novel viewpoints to be faithful to the underlying scene distribution.
*   **Loss:** $\mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{depth} \mathcal{L}_{depth} + \lambda_{sds} \mathcal{L}_{sds}$.

**When to choose:** When reconstructing scenes from very few input views (sparse-view) where standard multi-view consistency fails, and additional semantic or geometric priors are needed to constrain the solution space.

### GSDF
GSDF is a training optimization technique that transfers explicit surface properties to the discrete 3D Gaussian representation by enforcing consistency with a neural Signed Distance Function (SDF) [S1]. It enhances geometry reconstruction quality and recovers detailed mesh structures, distinguishing itself from standard 3D Gaussian Splatting by explicitly modeling the zero-level set of the surface [S2].

**Mechanism:**
*   Similar to NeuSG, GSDF couples 3DGS with an SDF network [S1].
*   **SDF Loss:** $\mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)|$ [S1].
*   **Normal Loss:** $\mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i)$ [S1].
*   **Optimization:** Jointly optimizes Gaussian attributes and SDF weights $\theta$ [S1].

**When to choose:** When high-fidelity surface geometry is the primary goal, and the computational overhead of maintaining an SDF network during training is acceptable [S1].

### [Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md)
[Not given in the sources.]

#### GaussianPro
**Definition:** A method that considers neighboring views' normal direction consistency to produce better reconstruction results [S1].
**Steps:** Introduces a progressive propagation strategy for updating Gaussians, considering normal consistency between neighboring views [S1]. Adds plane constraints to guide the growth of 3D Gaussian Splatting [S1].

#### 3DGSR
**Definition:** A method that encourages consistency between Signed Distance Function (SDF) and 3DGS to enhance geometry reconstruction quality [S1].

#### Gaussian Splatting Surface Refinement
**Definition:** Using pretrained volume rendering priors (UDF) to push Gaussians toward true surfaces by optimizing along the direction of increasing rendering weights, avoiding off-surface drift and discreteness [S2]. A post-training refinement strategy that uses a pretrained volume rendering prior to optimize Gaussian distributions toward true surfaces, particularly effective for open and thin structures [S2].
**Steps:** Integrate the prior into existing Gaussian Splatting variants (e.g., GaussianUDF, QGS) without modifying inference-time resource costs, as the prior network is bypassed during inference [S2].
**Comparison:** Outperforms handcrafted renderers and closed-surface-tailored 3DGS methods on open and thin structures, where standard Gaussian distributions degrade severely [S2].

## Background
The standard 3D Gaussian Splatting representation, introduced by Kerbl et al. [2023], excels at real-time novel view synthesis but lacks explicit surface constraints, often resulting in "floating" geometry that is difficult to extract as a clean mesh [S1]. To address this, subsequent works have integrated geometric priors. NeuSG [2023] was among the first to combine 3DGS with the NeuS implicit surface representation [S1]. GSDF [2023] and 3DGSR [2023] further explored SDF consistency for geometry enhancement [S1]. More recently, methods like Gaussian Splatting Surface Refinement [2026] have utilized pretrained volume rendering priors (UDF) to refine geometry, particularly for thin and open structures [S2]. Monocular depth alignment techniques have also been developed to resolve scale ambiguity in 3DGS reconstructions [S1]. These methods are widely used in applications requiring high-fidelity 3D assets, such as virtual reality, autonomous navigation, and digital twin creation [S1, S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
