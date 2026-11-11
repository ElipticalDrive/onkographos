# Neural Network Volume Classification

Also called: Sample-based volume classification, Neural-network-driven volume renderer, Interactive classification via painting, Painting-Based Intelligent User Interface for Volume Classification

## Overview
Neural Network Volume Classification is an interactive method that allows users to partition volume data into material classes by painting sample points on slicing planes, which an intelligent system (a machine learning classifier) uses to learn a classification function [S1, S2]. It solves the problem of interactive volume classification where users need real-time visual feedback to refine their segmentation, replacing the traditional transfer function space with a direct volume-space painting interface [S2]. It sets itself apart from its parent [Neural volume rendering](1576-neural-volume-rendering.md) by explicitly replacing the opacity mapping step with a learned neural classifier trained on user annotations, and from its sibling [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by focusing on supervised classification of discrete material boundaries rather than continuous radiance field reconstruction [S1, S2].

## How it works
The method operates in two distinct phases: an interactive training phase where the user paints samples and the network learns, and a rendering phase where the trained network classifies the volume. The core computational engine consists of four sequential stages applied to each ray [S1, S14, unofficial], but the classification logic replaces the standard neural query with a specific feed-forward network trained on user data.

### Inputs and Outputs
*   **Inputs**:
    *   Volume data (scalar or color).
    *   User-painted sample points on slicing planes (x, y, z coordinates).
    *   For each sample point, a feature vector is constructed. For scalar data, this includes the scalar value, gradient magnitude, scalar values of 6 neighbors (up, down, left, right, front, back), and the voxel's 3D position [S1, S2]. For color data with 6 neighbors, the input dimensionality is $3 + 6 \times 3 + 3 = 24$ (RGB values, neighbor RGB values, and position) [S1]. With 12 neighbors, the dimensionality is $3 + 12 \times 3 + 3 = 42$ [S2].
    *   Desired output labels: 1 for regions part of the material class (e.g., painted red/pink), 0 for regions not part of the class (e.g., painted blue) [S1, S2].
*   **Outputs**:
    *   A trained neural network (weights).
    *   A classified volume where each voxel has an uncertainty value (0 to 1) indicating membership in the material class [S2].
    *   The final rendered image, where opacity is mapped from the uncertainty value [S2].

### Data Structures
*   **Sample Point Table**: A list of records containing the feature vector for each painted voxel and its label.
    *   Fields: `scalar_value` (float), `gradient_magnitude` (float), `neighbor_values` (array of floats), `position` (vec3), `label` (float: 0 or 1).
*   **Neural Network Topology**: A three-layer perceptron (Input -> Hidden -> Output) [S1, S2].
    *   Input Layer: $m$ inputs (dimensionality of feature vector).
    *   Hidden Layer: $n$ hidden nodes.
    *   Output Layer: 1 output node (uncertainty value).
    *   Connections: Fully connected feed-forward. Every input neuron connects to all hidden neurons; all hidden neurons connect to the output neuron [S1, S2].
*   **Weights**:
    *   $W_{ih}$: Weights between input and hidden layers ($m \times n$).
    *   $W_{ho}$: Weights between hidden and output layers ($n \times 1$).
    *   Biases: $b_h$ (hidden), $b_o$ (output).

### Numbered Steps

1.  **Initialization**:
    *   Weights $W_{ih}, W_{ho}$ and biases $b_h, b_o$ are initialized to random values [S1, S2].
    *   The sample point table is empty.

2.  **User Painting and Data Collection**:
    *   The user paints on a slicing plane using two colors (e.g., red for class member, blue for non-member) [S1, S2].
    *   Painted voxels are recorded in the sample point table with their corresponding feature vectors and labels (1 for red, 0 for blue) [S1, S2].
    *   Painted regions are shown in other slicers for context [S2].

3.  **Training (Iterative)**:
    *   Training occurs in the system idle loop to maintain interactivity [S1, S2].
    *   For each iteration:
        a.  **Forward Propagation**: For each sample point $j$ in the table:
            i.  Compute hidden layer activations: $H_j = \sigma(W_{ih}^T \mathbf{x}_j + b_h)$, where $\mathbf{x}_j$ is the input feature vector and $\sigma(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid activation function [S1, S2].
            ii. Compute output: $y_j = \sigma(W_{ho}^T H_j + b_o)$ [S1, S2].
        b.  **Error Calculation**: Calculate error $E_j = y_j - t_j$, where $t_j$ is the target label (1 or 0) [S1, S2].
        c.  **Back-Propagation**:
            i.  Compute output error term: $\delta_o = E_j \cdot \sigma'(y_j)$ [S1, S2].
            ii. Compute hidden error term: $\delta_h = (W_{ho} \delta_o) \cdot \sigma'(H_j)$ [S1, S2].
            iii. Update weights: $W_{ho} \leftarrow W_{ho} - \eta \delta_o H_j^T$, $W_{ih} \leftarrow W_{ih} - \eta \delta_h \mathbf{x}_j^T$ [S1, S2].
            iv. Update biases: $b_o \leftarrow b_o - \eta \delta_o$, $b_h \leftarrow b_h - \eta \delta_h$ [S1, S2].
            *   $\eta$ is the learning rate [G].
    *   After each iteration, a classified slice is rendered to provide real-time visual feedback [S1, S2].

4.  **Classification and Rendering**:
    *   The trained network is applied to the entire volume (or current view) to classify each voxel [S1, S2].
    *   **Software Approach**: Each voxel is sent to the neural network to obtain a classified uncertainty volume. This scalar volume is then rendered using traditional volume rendering methods with linear interpolation [S1].
    *   **Hardware Approach**: A pixel shader implements the neural network during rasterization. The shader performs classification in the fragment program using interpolated data (postclassification) [S1].
        *   The pixel shader requires seven texture lookups for the center voxel and its neighbors [S2].
        *   The position is derived from texture coordinates [S2].
        *   The output uncertainty value is mapped to opacity [S2].
    *   **Compositing**: The optical properties are accumulated into the final pixel color using the front-to-back volume rendering equation [S1, S14, unofficial]. For each sample $i$ from front ($t_{start}$) to back ($t_{end}$), the accumulated color $C_{acc}$ and accumulated opacity $\alpha_{acc}$ are updated as follows:
        $$
        C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i \mathbf{c}_i
        $$
        $$
        \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
        $$
        where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [S12, S43, unofficial][S50, abstract only, unofficial]. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance [S1, S51, unofficial]. The final pixel color is often composited over a background color $C_{bg}$ using $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [S1, S14, unofficial].
    *   Early termination is used if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0) [S10, S12, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    """Sigmoid activation function."""
    return 1.0 / (1.0 + math.exp(-x))

def sigmoid_derivative(x):
    """Derivative of sigmoid function."""
    s = sigmoid(x)
    return s * (1.0 - s)

class NeuralNetwork:
    def __init__(self, input_dim, hidden_dim):
        """
        Initialize a 3-layer perceptron.
        input_dim: Number of input features (m).
        hidden_dim: Number of hidden nodes (n).
        """
        # Initialize weights randomly [S1, S2]
        self.W_ih = [[random.uniform(-1, 1) for _ in range(hidden_dim)] for _ in range(input_dim)]
        self.W_ho = [random.uniform(-1, 1) for _ in range(hidden_dim)]
        self.b_h = [0.0 for _ in range(hidden_dim)]
        self.b_o = 0.0

    def forward(self, x):
        """
        Forward propagation.
        x: Input feature vector (list of floats).
        Returns: Output uncertainty value (float).
        """
        # Hidden layer
        h = []
        for j in range(len(self.W_ho)):
            net_input = sum(x[i] * self.W_ih[i][j] for i in range(len(x))) + self.b_h[j]
            h.append(sigmoid(net_input))
        
        # Output layer
        net_output = sum(h[j] * self.W_ho[j] for j in range(len(h))) + self.b_o
        output = sigmoid(net_output)
        
        return output, h

    def train(self, samples, learning_rate=0.1):
        """
        Train the network on a set of samples.
        samples: List of tuples (input_vector, target_label).
        """
        for x, t in samples:
            # Forward pass
            output, h = self.forward(x)
            
            # Calculate error
            error = output - t
            
            # Back-propagation
            # Output layer error term
            delta_o = error * sigmoid_derivative(output)
            
            # Hidden layer error terms
            delta_h = []
            for j in range(len(h)):
                # Error from output weighted by hidden-to-output weights
                error_from_output = delta_o * self.W_ho[j]
                delta_h.append(error_from_output * sigmoid_derivative(h[j]))
            
            # Update weights and biases
            # Update W_ho
            for j in range(len(h)):
                self.W_ho[j] -= learning_rate * delta_o * h[j]
            
            # Update W_ih
            for i in range(len(x)):
                for j in range(len(h)):
                    self.W_ih[i][j] -= learning_rate * delta_h[j] * x[i]
            
            # Update biases
            self.b_o -= learning_rate * delta_o
            for j in range(len(h)):
                self.b_h[j] -= learning_rate * delta_h[j]

def classify_volume(volume_data, network, use_hardware=False):
    """
    Classify the volume using the trained network.
    volume_data: 3D array of voxel values.
    network: Trained NeuralNetwork object.
    use_hardware: If True, use hardware shader approach (postclassification).
                  If False, use software approach (preclassification).
    Returns: Classified volume (uncertainty values).
    """
    classified_volume = create_empty_volume_like(volume_data)
    
    if use_hardware:
        # Hardware approach: Classification in fragment program during rasterization
        # This is handled by the GPU shader, not CPU code.
        # The shader uses 7 texture lookups for center and neighbors [S2].
        pass
    else:
        # Software approach: Preclassification
        # Each voxel is sent to the network to obtain a classified uncertainty volume [S1]
        for x in range(volume_data.shape[0]):
            for y in range(volume_data.shape[1]):
                for z in range(volume_data.shape[2]):
                    # Construct feature vector
                    # Scalar value, gradient magnitude, neighbors, position [S1, S2]
                    scalar_val = volume_data[x, y, z]
                    grad_mag = calculate_gradient_magnitude(volume_data, x, y, z)
                    neighbors = get_neighbor_values(volume_data, x, y, z)
                    position = [x, y, z]
                    
                    input_vector = [scalar_val, grad_mag] + neighbors + position
                    
                    # Classify
                    uncertainty, _ = network.forward(input_vector)
                    classified_volume[x, y, z] = uncertainty
                    
    return classified_volume

def render_volume(classified_volume, background_color):
    """
    Render the classified volume using front-to-back compositing.
    classified_volume: Volume of uncertainty values (0-1).
    background_color: Background color to composite over.
    Returns: Final image.
    """
    # Standard volume rendering loop [S1, S14, unofficial]
    # For each pixel, cast ray, sample classified_volume, composite
    pass
```

### Parameters
*   **Network Size**:
    *   Input dimensionality ($m$): 11 for scalar data (scalar, gradient, 6 neighbors, position) [S1]. 24 for color data with 6 neighbors [S1]. 42 for color data with 12 neighbors [S2].
    *   Hidden nodes ($n$): 8 in the reported implementation [S1, S2]. Hardware implementation limited to max 8 hidden nodes due to instruction limits [S2]. Hidden nodes should be divisible by 4 for vectorization [S2].
*   **Learning Rate**: Not given in the sources. [G] Typical value 0.1.
*   **Training Iterations**: Incremental, occurring in the system idle loop [S1, S2].

### Edge Cases and Pitfalls
*   **Generalization Failure**: If position information is used in the input vector, painted voxels must be spread out across different slices to provide general samples for training. Otherwise, the network may overfit to specific locations [S1, S2].
*   **Hardware Limits**: The hardware implementation is limited by the number of texture lookups (7) and instruction count, restricting the network size to 8 hidden nodes [S2].
*   **Interpolation Differences**: Software classification performs preclassification (interpolation on scalar uncertainty volume), while hardware classification performs postclassification (interpolation of raw data before classification) [S1]. This can lead to slight differences in results [S1].

## Subtopics

### Hardware vs Software Neural Network Rendering
Hardware rendering performs classification in the fragment program using interpolated data (postclassification), allowing for real-time interactivity but limited by hardware constraints (max 8 hidden nodes, 7 texture lookups) [S1, S2]. Software rendering performs preclassification (classifying each voxel before rendering), allowing for arbitrary network sizes but suffering from slow performance (up to 9 seconds for a 256x256x256 volume on a 2.8 GHz Pentium 4), making it unsuitable for real-time training feedback [S1, S2].

## Operating characteristics
*   **Time Complexity**: Training is iterative and depends on the number of samples and network size. Software classification of a 256x256x256 volume with 11 input nodes and 8 hidden nodes takes 9 seconds on a 2.8 GHz Pentium 4 [S1, S2]. Hardware rendering achieves 1.5 frames per second for a 256x256x256 volume rendered to a 512x512 window on an ATI Radeon 9700 Pro [S2].
*   **Memory Complexity**: The network weights are small (under 1 kilobyte) and can be sent to the renderer asynchronously [S2]. The classified volume requires storage for the uncertainty values [S1].
*   **Strengths**: Interactive, intuitive user interface (painting in volume space); eliminates the need for transfer function tuning; high-dimensional classification improves material differentiation [S2].
*   **Limitations**: Requires user interaction and training time; hardware implementation is limited in network size; software implementation is too slow for real-time feedback [S1, S2].
*   **When to use**: When users have a good understanding of the data and can identify regions of interest visually, and real-time feedback is desired [S1, S2].
*   **When not to use**: When automatic classification is required without user input, or when the hardware constraints are too restrictive for the desired network complexity [S1, S2].

## Background
The method was introduced by [S1] and [S2] in the mid-2000s. [S2] presented the initial interface and hardware-assisted renderer in 2003, while [S1] provided a more detailed analysis of the intelligent system approach and software implementation in 2005. It is used in biomedical imaging and other fields where interactive volume segmentation is required [S2]. The method builds on the principles of [Direct volume ray casting](19-direct-volume-ray-casting.md) and [Neural volume rendering](1576-neural-volume-rendering.md), specifically replacing the neural query stage with a user-trained classifier [S1, S2].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] A novel interface for higher-dimensional classification of volume data, https://doi.org/10.1109/visual.2003.1250413, fetched 2026-10-27
