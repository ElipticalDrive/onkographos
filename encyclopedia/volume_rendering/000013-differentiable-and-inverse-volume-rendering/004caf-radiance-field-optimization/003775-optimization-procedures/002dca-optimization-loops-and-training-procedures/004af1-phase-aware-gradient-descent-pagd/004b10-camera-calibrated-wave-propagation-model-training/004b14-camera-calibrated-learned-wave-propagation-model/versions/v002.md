# Camera-Calibrated Learned Wave Propagation Model

Also called: Camera-Calibrated Model, Learned Forward Model

## Overview
The Camera-Calibrated Learned Wave Propagation Model is a data-driven surrogate for numerical propagation operators (such as the Angular Spectrum Method) used within differentiable holographic optimization loops. It solves the problem of model mismatch between idealized physics simulations and real-world optical hardware, specifically accounting for spatial light modulator (SLM) non-linearities, optical aberrations, stray light, and limited diffraction efficiency [S1]. Unlike analytical models that assume perfect free-space propagation, this learned model maps input phase patterns directly to captured multi-plane intensity images, providing a more accurate gradient signal for phase-aware gradient descent (PAGD) [S1].

## How it works
The method replaces the analytical forward propagation step in the optimization loop with a neural network trained to predict the intensity distribution at multiple depth planes given an input phase pattern.

**Inputs and Outputs:**
*   **Input:** A phase-only hologram (or phase pattern) displayed on the SLM.
*   **Output:** Predicted intensity images at 8 target depth planes.

**Data Structures:**
The model architecture details are not provided in the primary source text, which refers to supplementary materials for specific network layers and hyperparameters [S1]. The training data consists of pairs:
*   **Phase:** Phase-only holograms generated from 2D and 3D targets.
*   **Target:** Multi-plane intensity images captured from a physical prototype display.

**Training Procedure:**
1.  **Dataset Acquisition:** Use the "Flying Planes" dataset, which contains 1,000 randomly generated 3D scenes in OBJ format. These scenes include corresponding complex-valued wave fields, RGB-Depth images, and phase-only holograms generated via various encoders (DPAC, SGD, PAGD with ASM) [S1]. The 3D primitives are sampled with random position, rotation, and scale from the DIV2K dataset textures [S1].
2.  **Physical Data Collection:** Display numerous phase patterns (derived from 2D and 3D targets in the dataset) on a prototype SLM. Capture the resulting light field at 8 distinct target planes using a camera [S1].
3.  **Model Adaptation:** Adapt the learned wave propagation model architecture from Choi et al. [S1]. Crucially, introduce an extra low-pass filter within the model to suppress high-frequency phase noise [S1].
4.  **Loss Minimization:** Train the model using the phase and multi-plane image pairs. The objective is to minimize the discrepancy between the model's predicted intensities and the physically captured intensities. The loss function compares the scaled reconstructed field $u_r$ with the target field $u_t$. The specific final loss formulation is not fully detailed in the provided text, but it is defined as comparing the scaled reconstructed field with the target field [S1].

**Parameters:**
*   **Target Planes:** 8 planes [S1].
*   **Dataset Size:** 1,000 scenes [S1].
*   **Low-pass Filter:** Applied to suppress high-frequency phase noise [S1].

**Edge Cases and Pitfalls:**
*   **High-Frequency Noise:** Without the extra low-pass filter, high-frequency phase noise may degrade the model's accuracy [S1].
*   **Hardware Specificity:** The model is calibrated to a specific prototype optical system; it does not generalize to different hardware configurations without retraining [S1].

## Subtopics

## Operating characteristics
*   **Accuracy:** The learned model provides a Peak Signal-to-Noise Ratio (PSNR) improvement of 2 dB compared to the analytical Angular Spectrum Method (ASM) baseline [S1].
*   **Computational Cost:** Not given in the sources.
*   **Strengths:** Captures real-world imperfections (aberrations, SLM non-linearities, stray light) that numerical models like ASM struggle to replicate [S1].
*   **Limitations:** Requires physical data collection and training; does not provide a differentiable physics-based gradient in the traditional sense but rather a learned gradient [S1].

## Background
The method originates from research into full 3D holographic displays, specifically addressing the limitations of numerical propagation operators in differentiable holography. It builds on the work of Choi et al. [S1]. The approach is part of a broader effort to mesh-represent and learning-empower hologram synthesis for full 3D holographic displays [S1].

## Sources
[S1] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
