# Implicit Neural Representation (INR) with Parametric Motion Field for 4D-CT
Also called: Differentiable Analysis-by-Synthesis 4D-CT, Warp Template Scene with Motion Field

## Overview
This method reconstructs a 4D dynamic scene (3D volume evolving over time) from limited-view sinogram data without requiring training data or phase information [S1]. It jointly estimates a static 3D scene template via an Implicit Neural Representation (INR) and a parametric motion field that warps the template over time [S1]. Unlike traditional 4D-CT methods that require sparse measurements spanning 360 degrees or phase-gating for periodic motion, this approach works with limited-view sampling (partial angular ranges) and self-supervised analysis-by-synthesis [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Captured sinogram measurements $p_{\theta}(r, z)$ corresponding to specific view angles $\theta(t)$ and times $t$ [S1].
*   **Output:** Optimized INR network weights representing the static scene template and optimized motion field parameters representing the dynamic deformation [S1].

**Data Structures:**
1.  **INR Network:** A Multi-Layer Perceptron (MLP) with positional encoding that maps 3D spatial coordinates $(x, y, z)$ to the linear attenuation coefficient (LAC) $\tilde{\sigma}$ of the static template [S1]. The network outputs a scalar value for density at any queried coordinate.
2.  **Motion Field Parameters:** A set of parameters $\phi$ defining a parametric warping function that transforms static coordinates to dynamic coordinates based on time $t$ [S1]. The specific mathematical form of the parametric field is not detailed in the sources, but it is treated as a differentiable function $W(x, y, z, t; \phi)$ [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize the INR network weights and the motion field parameters $\phi$ [S1].
2.  **Scene Generation:** For a given time $t$, query the INR to get the static template density $\tilde{\sigma}(x, y, z)$. Apply the motion field to warp the coordinates: $(x', y', z') = W(x, y, z, t; \phi)$. The dynamic scene density at time $t$ is $\sigma(x, y, z, t) = \tilde{\sigma}(x', y', z')$ [S1].
3.  **Differentiable Projection:** Apply the 3D Radon transform to the generated 4D scene frame $\sigma(x, y, z, t)$ to project it into sinogram space, producing synthesized measurements $p_{\theta}^{syn}(r, z)$ [S1].
4.  **Loss Computation:** Compute the loss function between the synthesized sinogram measurements and the ground truth (captured) sinogram measurements [S1].
5.  **Optimization:** Backpropagate the loss to update the INR network weights and the motion field parameters $\phi$ [S1].
6.  **Iteration:** Repeat steps 2-5 until convergence [S1].

**Formulas:**
The forward imaging model uses the 3D Radon transform for parallel-beam CT [S1]:

$$
p_{\theta}(r, z) = \iint \sigma(x, y, z, t) \delta(x \cos(\theta(t)) + y \sin(\theta(t)) - r) \, dx \, dy
$$

Where:
*   $p_{\theta}(r, z)$ is the synthesized sinogram measurement at view angle $\theta$, detector pixel $r$, and height $z$ [S1].
*   $\sigma(x, y, z, t)$ is the linear attenuation coefficient (LAC) of the dynamic scene at coordinates $(x, y, z)$ and time $t$ [S1].
*   $\theta(t)$ is the view angle at time $t$ [S1].
*   $\delta(\cdot)$ is the Dirac delta function [S1].

**Assumptions and Parameters:**
*   **Fixed LAC during exposure:** The scene's LAC remains fixed for a single projection exposure time $dt$ [S1]. This is mathematically expressed as:
    $$
    ||\sigma(x, y, z, t + dt) - \sigma(x, y, z, t)||_1 = 0
    $$
    [S1]
*   **Parallel-Beam Geometry:** The method assumes parallel-beam geometry, typical of synchrotron CT scanners [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Mismatch:** The method assumes parallel-beam geometry. It does not support cone-beam or helical measurements directly. To support these, the Radon transform must be replaced with a differentiable volume ray tracer [S1].
*   **Pre-processing Requirements:** The method assumes CT measurements have undergone pre-processing for beam hardening and truncation. Synchrotron systems use monochromatic sources (no beam hardening), but other systems require correction methods [S1].
*   **Limited View Ambiguity:** While robust, rapid motion combined with limited view sampling can lead to spatiotemporal ambiguities [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Validated on a synthetic dataset for parallel beam CT generated with an accurate physics simulator, and on publicly available thoracic CT reconstructions where x-ray measurements were resimulated. The method outperforms competitive baselines in these scenarios [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Works with limited-view sampling (partial angular ranges in a single revolution); does not require phase information for reconstruction; self-supervised (no training data required) [S1].
*   **Limitations:** Restricted to parallel-beam geometry; requires pre-processed sinograms (beam hardening/truncation correction); performance may degrade with very rapid motion due to spatiotemporal ambiguities [S1].
*   **When to use:** When reconstructing 4D scenes from limited-view sinogram data where phase information is unavailable and traditional 360-degree sampling is not feasible [S1].
*   **When not:** When cone-beam or helical CT data is used without modification; when beam hardening/truncation corrections are not applied [S1].

## Background
*   **Origin:** Proposed in "Dynamic CT Reconstruction from Limited Views with Implicit Neural Representations and Parametric Motion Fields" (ICCV 2021) [S1].
*   **History and Development:** The method builds on recent advances in Implicit Neural Representations (INRs) using positional encoding to map input coordinates to volume density [S1]. It addresses limitations in traditional 4D-CT methods that require sparse measurements spanning 360 degrees or phase-gating [S1].
*   **Usage:** Validated on synthetic parallel-beam CT data and resimulated thoracic CT data [S1].

## Sources
[S1] Dynamic CT Reconstruction from Limited Views with Implicit Neural Representations and Parametric Motion Fields, https://doi.org/10.1109/iccv48922.2021.00226, fetched 2026-10-26
