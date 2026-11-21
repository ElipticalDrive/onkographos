# Neural Network 3D Object Representation

Also called: MLP Implicit Surface Representation, Neural Implicit Surface

## Overview
A method where a multilayer feed-forward neural network represents a 3D object by mapping 3D coordinates in object space to a binary output indicating membership [S1]. The network is trained on generated examples using back propagation, and simple analytical shapes can be represented by manually specifying the network's parameters [S1]. It differs from sibling methods like [MLP Neural Fields](3854-mlp-neural-fields.md) by focusing specifically on binary occupancy/membership rather than continuous density or color fields [G].

## How it works
**Inputs:**
- 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ in object space [S1].
- Training dataset of labeled points $(\mathbf{x}_i, y_i)$ where $y_i \in \{0, 1\}$ indicates outside/inside or on-surface status [S1].

**Outputs:**
- A scalar value $f(\mathbf{x})$ indicating membership [S1].

**Data Structures:**
- **Neural Network:** A feed-forward multilayer perceptron $F_\Theta: \mathbb{R}^3 \rightarrow \mathbb{R}$ with parameters $\Theta$ [S1]. One network is used per volumetric description [S1]. Architecture details (layer count, hidden dimensions, activation functions) are not given in the sources.
- **Training Data:** A set of points $\{\mathbf{x}_i\}$ and labels $\{y_i\}$ [S1].

**Steps:**
1. **Network Definition:** Define a feed-forward neural network $F_\Theta: \mathbb{R}^3 \rightarrow \mathbb{R}$ with parameters $\Theta$ [S1]. For simple analytical shapes (e.g., sphere, cube), specify $\Theta$ manually [S1].
2. **Training Data Generation:** For complex shapes, generate training examples consisting of points on the object's surface, points on inclosed surfaces, and points on enclosing surfaces [S1].
3. **Heuristic Labeling:** Use a heuristic based on surface normals to determine if a point in the vicinity belongs to the inside or outside of the object [S1].
4. **Training:** Train the network on the generated examples using back propagation [S1].
5. **Evaluation:** Test the representation by repeatedly evaluating the network for points inside the object space to determine occupancy [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class NeuralObjectRepresentation:
    def __init__(self, layer_sizes):
        """Initialize MLP. Input dim=3, output dim=1. layer_sizes includes input and output."""
        self.layer_sizes = layer_sizes
        self.weights = []
        self.biases = []
        for i in range(len(layer_sizes) - 1):
            # [G] He initialization for weights
            w = np.random.randn(layer_sizes[i], layer_sizes[i+1]) * np.sqrt(2.0 / layer_sizes[i])
            b = np.zeros((1, layer_sizes[i+1]))
            self.weights.append(w)
            self.biases.append(b)

    def forward(self, x):
        """Forward pass. x shape: (N, 3). Returns logits (N, 1)."""
        a = x
        for i in range(len(self.weights) - 1):
            z = np.dot(a, self.weights[i]) + self.biases[i]
            a = np.maximum(0, z) # [G] ReLU activation
        z = np.dot(a, self.weights[-1]) + self.biases[-1]
        return z

    def generate_training_data(self, surface_points, normals, epsilon=0.01):
        """Generate training points using surface normal heuristic [S1]."""
        inside_points = surface_points - epsilon * normals
        outside_points = surface_points + epsilon * normals
        points = np.vstack([surface_points, inside_points, outside_points])
        labels = np.concatenate([np.ones((len(surface_points), 1)),
                                 np.zeros((len(inside_points), 1)),
                                 np.ones((len(outside_points), 1))])
        return points, labels

    def train(self, points, labels, epochs=1000, lr=0.001):
        """Train using backpropagation [S1]."""
        for epoch in range(epochs):
            # Forward pass
            activations = [points]
            z_values = []
            a = points
            for i in range(len(self.weights) - 1):
                z = np.dot(a, self.weights[i]) + self.biases[i]
                z_values.append(z)
                a = np.maximum(0, z)
                activations.append(a)
            output = np.dot(a, self.weights[-1]) + self.biases[-1]

            # Backward pass
            delta = 2.0 * (output - labels) / len(points)
            for i in range(len(self.weights) - 1, -1, -1):
                grad_w = np.dot(activations[i].T, delta)
                grad_b = np.sum(delta, axis=0, keepdims=True)
                self.weights[i] -= lr * grad_w
                self.biases[i] -= lr * grad_b
                if i > 0:
                    delta = np.dot(delta, self.weights[i].T)
                    delta *= (z_values[i-1] > 0).astype(float) # ReLU derivative

    def is_inside(self, query_points):
        """Evaluate if points are inside the object."""
        output = self.forward(query_points)
        # [G] Thresholding. Source implies binary decision.
        return output > 0.5
```

**Parameters:**
- Learning rate: Not given in the sources.
- Network depth/width: Not given in the sources.
- Threshold for occupancy: Not given in the sources (implied binary decision).
- Normal perturbation distance (epsilon): Not given in the sources.

**Edge Cases and Pitfalls:**
- **Simple Shapes:** For simple analytical forms, manual parameter specification is required [S1].
- **Complex Shapes:** Requires careful generation of training data to ensure the network learns the boundary correctly [S1].
- **Heuristic Limitations:** The surface normal heuristic may fail for non-manifold or highly curved surfaces if normals are unreliable [G].

## Subtopics

### [Implicit methods (SDF-based)](12421-implicit-methods-sdf-based.md)
Implicit methods parameterize scene features—such as density, color, occupancy probability, or Signed Distance Field (SDF) values—as a continuous function approximated by a Multi-Layer Perceptron (MLP) network. Unlike explicit methods (voxels, point clouds, meshes) which are discrete and limited in resolution, implicit methods are continuous and accurate at arbitrary resolutions.

## Operating characteristics
- **Time Complexity:** Evaluation is $O(L \cdot D^2)$ per point, where $L$ is the number of layers and $D$ is the hidden dimension [G]. Training time depends on the number of training points and epochs [G].
- **Memory Complexity:** $O(L \cdot D^2)$ for storing network weights [G].
- **Performance:** Hardware implementable, offering the ability for faster representation [S1].
- **Strengths:** Accurate and compact representation [S1].
- **Limitations:** Requires training data generation for complex shapes [S1]. Not given in the sources regarding specific rendering speed compared to rasterization.

## Background
Originated in the context of computer graphics for representing 3D objects using neural networks [S1]. The paper describes this as the "second step" towards creating a real-time 3D renderer based entirely on neural networks [S1]. It builds on the idea that feedforward neural networks are hardware implementable [S1].

## Sources
[S1] Affine transformations of 3D objects represented with neural networks, https://doi.org/10.1109/im.2001.924438, fetched 2026-10-26
