# Implicit Neural Networks (Neural Fields)
Also called: Implicit Neural Representations, Neural Fields

## Overview
Implicit Neural Networks (Neural Fields) represent continuous 3D measurements or scene properties as the output of a Multilayer Perceptron (MLP) [S1, unofficial]. Unlike discrete voxel grids or explicit meshes, this method provides a continuous, differentiable function that can be evaluated at arbitrary coordinates, solving the problem of discretization artifacts and enabling gradient-based optimization of input parameters [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D coordinate vector $\mathbf{x} \in \mathbb{R}^3$ (or $\mathbb{R}^2$ for 2D slices).
*   **Output:** A scalar value representing the measurement, density, occupancy, or SDF value at that coordinate.

**Data Structures:**
*   **Network:** A standard Multilayer Perceptron (MLP). The architecture is not strictly defined in the source but typically consists of linear layers with non-linear activation functions (e.g., ReLU, Sine) [S1, unofficial].
*   **Parameters:** Weights and biases of the MLP, denoted collectively as $\theta$.

**Steps:**
1.  **Initialization:** Initialize the MLP parameters $\theta$ randomly or using a specific initialization scheme.
2.  **Forward Pass:** For a given query point $\mathbf{x}$, compute the output $f(\mathbf{x}; \theta)$ by passing $\mathbf{x}$ through the MLP layers.
3.  **Differentiability:** The network is naturally differentiable with respect to $\mathbf{x}$ [S1, unofficial]. This allows the computation of gradients $\nabla_{\mathbf{x}} f(\mathbf{x}; \theta)$ via automatic differentiation.
4.  **Optimization:** If the coordinates $\mathbf{x}$ are uncertain (e.g., sensor locations), the gradients are used to optimize $\mathbf{x}$ jointly with the network parameters $\theta$ using gradient-based methods [S1, unofficial].

**Formulas:**
The core representation is:
$$ y = f(\mathbf{x}; \theta) $$
Where $y$ is the predicted value, $\mathbf{x}$ is the input coordinate, and $\theta$ are the network weights. The differentiability property is expressed as:
$$ \frac{\partial y}{\partial \mathbf{x}} $$
This gradient is used to optimize uncertain measurement coordinates $\mathbf{x}$ [S1, unofficial].

**Parameters:**
*   **Network Depth/Width:** Not specified in the source.
*   **Activation Function:** Not specified in the source.

**Edge Cases and Pitfalls:**
*   **Optimization Time:** Implicit neural networks require more time to optimize compared to alternative methods like splines [S1, unofficial].
*   **Interpretability:** They are less simple to interpret than spline-based methods [S1, unofficial].

## Subtopics

### [Mixture of Neural Implicit Functions](18011-mixture-of-neural-implicit-functions.md)
A generative variant that represents data as a weighted sum of basis networks, offering stable modulation compared to HyperNet or FiLM.

## Operating characteristics
*   **Time Complexity:** Optimization is slower than spline-based interpolators [S1, unofficial].
*   **Memory Complexity:** Depends on the size of the MLP weights. Generally compact for smooth functions but can be large for complex scenes.
*   **Performance:** Performs as well as differentiable splines in terms of reconstruction accuracy for uncalibrated imaging tasks [S1, unofficial].
*   **Strengths:** Natural differentiability with respect to input coordinates; continuous representation; no discretization artifacts.
*   **Limitations:** Slower optimization; harder to interpret; potential for overfitting if not regularized.
*   **When to use:** When continuous representation and gradient-based optimization of input coordinates are required (e.g., uncalibrated imaging, sensor localization).
*   **When not to use:** When fast optimization or high interpretability is prioritized over continuous differentiability (e.g., use splines instead) [S1, unofficial].

## Background
Implicit Neural Networks, also known as Neural Fields, were applied to address uncertainty in measurement coordinates in imaging frameworks, such as computed tomography [S1, unofficial]. The approach leverages the natural differentiability of neural networks to jointly fit a measurement representation, optimize uncertain measurement coordinates, and perform image reconstruction [S1, unofficial]. This method was compared against differentiable spline interpolators, which were found to perform similarly but with faster optimization and better interpretability [S1, unofficial].

## Sources
[S1] Differentiable Uncalibrated Imaging, https://doi.org/10.48550/arXiv.2211.10525, fetched 2026-10-26 (unofficial)
