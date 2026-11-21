# Runtime Style/Shape Transformation via MLP Encoders
Also called: Style/Shape Transformation via MLP Encoders
## Overview
This method enables the real-time contextual translation of Gaussian Splatting (3DGS) objects by encoding multiple semantic states (e.g., burning, frozen, charcoal) into a single model [S1]. It solves the problem of managing distinct asset variants for interactive scenes by using MLP-based encoders to instantly produce new 3DGS states upon interaction [S1]. It sets itself apart from sibling methods like [Gsplat Split and Combine](8494-gsplat-split-and-combine.md) by avoiding geometric restructuring; instead, it modulates the existing splat parameters via learned latent representations, allowing for seamless state transitions without topology changes [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained 3DGS model representing the object in its default state, consisting of Gaussian parameters (position $\mathbf{p}$, covariance $\mathbf{\Sigma}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$). An interaction signal (e.g., collision event, projectile hit) containing a target state identifier $s$.
*   **Output:** Updated Gaussian parameters corresponding to the target state $s$, rendered into the scene.

**Data Structures:**
*   **MLP Encoder:** A multi-layer perceptron $f_{\theta}$ with weights $\theta$. It takes a scalar or vector state code $z_s$ (representing the target state) and outputs a set of modulation parameters $\mathbf{m}_s$. The exact architecture (layers, activation functions) is not specified in the sources [S1].
*   **Gaussian Parameters:** Stored in standard 3DGS format. The method does not specify a new data layout but implies the MLP outputs modify existing fields [S1].

**Steps:**
1.  **Initialization:** Train or load a 3DGS model for the object. Simultaneously train or initialize the MLP encoder $f_{\theta}$ to map state codes $z_s$ to modulation parameters $\mathbf{m}_s$ that transform the default Gaussian parameters into the desired state parameters [S1].
2.  **Interaction Detection:** Monitor the scene for interaction events, such as collision between a player-controlled object (e.g., confetti blaster projectile) and the 3DGS object [S1].
3.  **State Determination:** Upon interaction, determine the new target state $s$ based on the interaction type (e.g., "fire" interaction sets $s = \text{burning}$) [S1].
4.  **Encoder Activation:** Pass the state code $z_s$ for the target state $s$ into the MLP encoder $f_{\theta}$ to generate the modulation parameters $\mathbf{m}_s$ [S1].
5.  **Parameter Transformation:** Apply $\mathbf{m}_s$ to the current Gaussian parameters to produce the new state parameters. The sources do not provide the exact mathematical formula for applying $\mathbf{m}_s$ to the Gaussian parameters, only that it results in a "new 3DGS state" [S1]. [G] Typically, this involves affine transformations of position, scaling, or color coefficients based on the modulation vector.
6.  **Rendering:** Render the updated Gaussian parameters using standard volume rendering techniques.
7.  **Normalization/Counter-Effect:** If a subsequent interaction counteracts the previous state (e.g., shooting a frozen object with fire), repeat steps 2-6 with the new state code. The method supports "normalizing" the object back to a default or intermediate state by applying a counter-effect [S1].

**Parameters:**
*   **State Codes ($z_s$):** Not given in the sources.
*   **MLP Weights ($\theta$):** Trained parameters. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method is designed for real-time performance in VR, including mobile devices like Meta Quest 3, implying potential precision constraints [S1].
*   **Degenerate Input:** Not given in the sources.
*   **Boundary Conditions:** The method handles discrete state transitions. Continuous interpolation between states is not explicitly described [S1].

## Subtopics

### [Deferred Style Transformation (DST)](17759-deferred-style-transformation-dst.md)
Applies style transformation to 2D feature maps after volume rendering, solving multi-view inconsistency and high memory usage of 3D point-wise transformation. It handles style-specific affine transformations in the 2D domain.

## Operating characteristics
*   **Time Complexity:** Real-time capable. The method is designed for interactive VR applications, including standalone mobile VR (Meta Quest 3) and PC-VR [S1].
*   **Memory Complexity:** Lower than maintaining multiple distinct 3DGS models, as it encodes multiple states in a single model with an MLP [S1].
*   **Measured Performance:** Tested in Unity VR template with 3DGS objects. Photorealistic quality may be reduced for mobile devices to fit hardware constraints [S1].
*   **Strengths:** Enables dynamic, interactive object states without pre-computing or storing multiple asset variants. Instant state changes upon interaction.
*   **Limitations:** Requires training the MLP encoder for each object and set of states. Quality may be reduced on mobile hardware [S1].
*   **When to use:** Interactive VR/AR applications where objects need to change appearance dynamically based on user interaction.
*   **When not to use:** Static scenes or scenes where pre-computed variants are sufficient and real-time interaction is not required.

## Background
The method was introduced in "Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR" [S1]. It builds on the trend of photorealistic product placement in commercials and addresses limitations in data and graphics power of client devices [S1]. The technique was implemented in a Unity VR template, demonstrating interaction with 3DGS objects via collision and projectile shooting [S1].
## Sources
[S1] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
