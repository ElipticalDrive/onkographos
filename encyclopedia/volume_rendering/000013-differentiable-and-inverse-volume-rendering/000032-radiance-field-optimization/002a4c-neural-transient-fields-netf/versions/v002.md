# Neural Transient Fields (NeTF)
## Overview
Neural Transient Fields (NeTF) is a method for Non-Line-of-Sight (NLOS) imaging that models the plenoptic transient field using a Multi-Layer Perceptron (MLP) to predict transient arrival times $\tau$ from sparse Single-Photon Avalanche Diode (SPAD) measurements on a relay wall [S1]. Unlike [NeRF](4972-neural-radiance-fields.md), which relies on dense high-resolution images and 1D ray sampling, NeTF operates on sparse 2D hemispherical spot measurements, making occlusion handling and sampling bias critical challenges [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** Sparse SPAD measurements on a relay wall, represented as coordinates $(x', y')$ and time $t$, yielding measured transient arrival times $\tau_m(x', y', t)$ [S1].
*   **Output:** A continuous function $\tau(x', y', t)$ predicted by the MLP, representing the transient field [S1].

**Data Structures:**
*   **MLP:** A standard Multi-Layer Perceptron [S1]. Input coordinates are transformed into Cartesian coordinates before network evaluation [S1]. The network outputs the transient arrival time $\tau$ [S1].

**Algorithm Steps:**
1.  **Coordinate Transformation:** Transform the input SPAD spot coordinates $(x', y')$ and time $t$ into Cartesian coordinates suitable for the MLP input [S1].
2.  **Forward Pass:** Evaluate the MLP to predict the transient arrival time $\tau(x', y', t)$ for the given input coordinates [S1].
3.  **Loss Calculation:** Compute the L2 norm loss between the predicted transient $\tau$ and the measured transient $\tau_m$ [S1]:
    $$
    L = \sum_{x', y', t} (\tau_m(x', y', t) - \tau(x', y', t))^2
    $$
4.  **Optimization:** Minimize the loss $L$ using gradient descent to update the MLP parameters [S1].
5.  **Two-Stage Training:**
    *   **Stage 1:** Train using all available samples to obtain an initial reconstruction [S1].
    *   **Stage 2:** Calculate the loss at every measurement spot to guide resampling, over-sampling spots with higher loss to address sampling bias and occlusion artifacts [S1].

**Parameters:**
*   **Loss Function:** L2 norm is the standard choice, though arbitrary differentiable losses can be minimized [S1].
*   **Sampling:** Hierarchical sampling is employed to address sampling bias [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** In NLOS, SPAD captures sparse spots [S1]. Occluded points may be captured from very few viewpoints, leading to strong reconstruction artifacts if not handled [S1]. The two-stage training strategy is specifically designed to mitigate this [S1].
*   **Sampling Bias:** The sparse nature of SPAD data creates a long-tailed classification-like problem where some regions are under-sampled [S1].
## Subtopics

### [Non-confocal Neural Transient Field (NeTF) Formulation](6348-non-confocal-neural-transient-field-netf-formulation.md)
Formulates the transient signal for non-confocal NLOS setups where illumination spot $P$ and detection spot $P'$ are distinct. It differs from the confocal formulation by using ellipsoidal coordinates to handle asymmetric optical paths, allowing the transient to be computed as a double integral over the ellipsoidal surface parameters. Choose it when the optical setup lacks a shared focal point between illumination and detection.

## Operating characteristics
*   **Time and memory complexity:** Not given in the sources.
*   **Measured performance with hardware, resolution, and scene:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Can model the plenoptic transient field with an MLP, allowing for the minimization of arbitrary differentiable losses [S1].
*   **Limitations:** Highly susceptible to occlusion artifacts due to sparse viewpoint coverage; requires specialized two-stage training and hierarchical sampling to handle sampling bias [S1].
*   **When to use:** Non-Line-of-Sight imaging scenarios with sparse SPAD data [S1].
*   **When not to use:** Dense viewpoint settings where occlusion is less significant, such as standard NeRF applications [S1].
## Background
NeTF was introduced in the paper "Non-line-of-Sight Imaging via Neural Transient Fields" [S1]. It adapts the neural field concept to the specific constraints of NLOS imaging, where data is acquired via sparse SPAD spots on a relay wall rather than dense camera images [S1].
## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
