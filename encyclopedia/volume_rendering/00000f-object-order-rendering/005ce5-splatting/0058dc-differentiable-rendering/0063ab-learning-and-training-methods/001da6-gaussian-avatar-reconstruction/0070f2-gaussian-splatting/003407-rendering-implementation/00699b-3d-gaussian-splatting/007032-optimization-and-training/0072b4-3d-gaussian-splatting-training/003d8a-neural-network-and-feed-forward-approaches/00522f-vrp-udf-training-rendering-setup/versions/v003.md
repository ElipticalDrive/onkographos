# VRP-UDF Training & Rendering Setup

Also called: UDF-Gaussian Joint Training, Volume Rendering Prior Network

## Overview
VRP-UDF is a neural network-based volume rendering prior that learns unsigned distance functions (UDFs) to guide the optimization of 3D Gaussian Splatting. It solves the problem of inaccurate surface reconstruction and depth errors common in standard Gaussian splatting by providing dense, geometrically consistent queries around the UDF zero-level set. Unlike sibling methods like MVSplat which primarily regress Gaussian parameters directly from images, VRP-UDF introduces an intermediate implicit surface representation (the UDF) that regularizes the Gaussian distribution, particularly improving thin structure recovery.

## How it works
The method integrates a UDF network and a color network into the Gaussian Splatting training pipeline. The UDF network predicts the distance to the nearest surface, while the color network predicts appearance. These networks guide the placement and refinement of Gaussians during the later stages of training.

### Inputs and Outputs
*   **Inputs:** Multi-view images $\{I_1, \dots, I_N\}$ with known camera parameters; ground truth UDF values and depth maps for the training scenes.
*   **Outputs:** Refined 3D Gaussian parameters $\{\mathbf{\mu}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i\}$ optimized via the UDF prior; rendered images and depth maps.

### Data Structures
The core data structures are four Multi-Layer Perceptrons (MLPs) with 256 hidden units per layer. The specific architectures are:
1.  **$f_{w1}$**: 3-layer MLP. Used for initial feature processing.
2.  **$f_{w2}$**: 6-layer MLP with skip connections.
3.  **$f_u$ (UDF Network)**: 8-layer MLP with skip connections. Outputs the unsigned distance value.
4.  **$f_c$ (Color Network)**: 2-layer MLP. Outputs color.

### Numbered Steps
1.  **Initialization:** Initialize the 3D Gaussian Splatting scene with initial Gaussians derived from sparse structure-from-motion (SfM) points or similar initialization methods (standard for Gaussian Splatting). Initialize the MLPs $f_{w1}, f_{w2}, f_u, f_c$ with random weights.
2.  **Early Training (Iterations 1–9,000):** Train the Gaussian Splatting model using standard differentiable rasterization and photometric loss. The UDF networks are not active or updated during this phase.
3.  **UDF Network Activation (Iteration 9,000):** Begin training the UDF networks ($f_u$ and $f_c$). The learning rate and decay scheduler for $f_u$ and $f_c$ follow the schedules established in prior works [1, 24] (exact values not specified in the primary source).
4.  **UDF Constraint Application:**
    *   Query the UDF network $f_u$ at Gaussian centers or ray-sampling points.
    *   Apply **Softplus** activation to the output of $f_u$ to enforce non-negative outputs, ensuring the function remains a valid UDF.
    *   Compute loss between predicted UDF values and ground truth UDFs.
5.  **Color Prediction:**
    *   Query the color network $f_c$.
    *   **Crucial Step:** Do **not** include normal vectors in the input to $f_c$. Normals are ambiguous at the zero-level set due to the non-differentiability of UDFs. Inputs to $f_c$ should typically be positional encoding of coordinates and view direction.
    *   Compute photometric loss between predicted color and ground truth image color.
6.  **Gaussian Refinement (Final 10,000 Iterations):**
    *   Cast rays for rendering.
    *   Use a batch size of **2,048** rays.
    *   Sample **64 points** along each ray.
    *   Use the UDF values to guide the sampling distribution or weighting, ensuring dense queries near the zero-level set.
    *   Update Gaussian parameters ($\mathbf{\mu}, \mathbf{R}, \mathbf{s}, \alpha, \mathbf{c}$) and network weights jointly to minimize the combined photometric and UDF consistency losses.
7.  **Total Training Duration:** The entire process runs for **30,000 iterations**.

### Formulas
*   **UDF Output Constraint:**
    $$ \hat{d} = \text{Softplus}(f_u(\mathbf{x})) $$
    Where $\mathbf{x}$ is the query point and $\hat{d} \ge 0$.
*   **Color Input:**
    $$ \mathbf{c}_{pred} = f_c(\text{PE}(\mathbf{x}), \mathbf{v}) $$
    Where $\mathbf{v}$ is the view direction, and normals $\mathbf{n}$ are explicitly excluded from the input features.

### Parameters
*   **Hidden Units:** 256 for all MLPs.
*   **Layers:** $f_{w1}$ (3), $f_{w2}$ (6), $f_u$ (8), $f_c$ (2).
*   **Skip Connections:** Included in $f_{w2}$ and $f_u$.
*   **Ray Sampling:** 64 points per ray.
*   **Batch Size:** 2,048 rays.
*   **Training Schedule:** UDF training starts at iteration 9,000. Refinement phase is iterations 20,000–30,000.

### Edge Cases and Pitfalls
*   **Normal Ambiguity:** Including normal vectors in the color network input causes errors because UDFs are non-differentiable at the zero-level set, making normals ill-defined. Always remove normals from $f_c$ input [S1].
*   **Convergence Parameters:** Excessively large convergence parameters (e.g., the "flip factor" in NeuralUDF) can cause the UDF to flip into an SDF prematurely, before the true surface is reached, leading to depth rendering errors [S1].
*   **Sampling Inaccuracies:** Random hierarchical sampling and the ill-defined nature of UDF fields near surfaces can cause rendering inaccuracies. This is a common issue in volume rendering frameworks, even with theoretically unbiased mappings like NeuS [S1].
*   **Thin Structures:** Standard methods may struggle to recover extremely thin surfaces (e.g., eggshells, branches). VRP-UDF improves this but may still require advanced UDF extraction techniques for optimal results on very thin structures [S1].

## Operating characteristics
*   **Time Complexity:** The addition of MLP evaluations for UDF and color prediction increases per-iteration cost compared to standard Gaussian Splatting. The total training time is fixed at 30,000 iterations, with the UDF overhead active for the last 21,000 iterations.
*   **Memory Complexity:** Requires storing the weights of four MLPs (total parameters $\approx 4 \times (256 \times \text{input\_dim} + 256)$). The ray sampling batch size of 2,048 with 64 points requires memory for intermediate ray queries.
*   **Performance:** Achieves best accuracy among compared renderers for UDFs and SDFs on ShapeNet and real scans. Outperforms NeUDF and VRPrior on complex indoor scenes and thin structures (e.g., ship masts, plant branches) [S1].
*   **Strengths:** Provides unbiased learning of UDFs; significantly improves reconstruction of thin and complex structures; reduces depth errors compared to standard Gaussian Splatting.
*   **Limitations:** Sensitive to convergence parameters; still subject to sampling inaccuracies near surfaces; requires careful scheduling of UDF network activation.

## Background
VRP-UDF was introduced in the paper "VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors" [S1]. It builds upon the principles of differentiable volume rendering and unsigned distance functions, extending them to the context of 3D Gaussian Splatting. The method addresses limitations in existing UDF-based methods (like NeuralUDF and VRPrior) which struggle with thin surfaces and depth accuracy. It leverages the efficiency of Gaussian Splatting while incorporating the geometric priors of implicit surface representations.

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
