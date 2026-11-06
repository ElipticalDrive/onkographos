# Neural Network Volume Classification

Also called: Sample-based volume classification, Neural-network-driven volume renderer, Interactive classification via painting, Painting-Based Intelligent User Interface for Volume Classification

## Overview
Neural Network Volume Classification is an interactive method that partitions volume data into material classes by training an artificial neural network on user-painted samples, mapping voxel properties to an uncertainty value that serves as the transfer function for rendering [S1, S2]. It eliminates the need for manual transfer function definition by replacing the transfer function space with a direct volume-space painting interface [S2]. Unlike traditional methods that require defining curves in derived intensity space, this approach learns a high-dimensional classification function from sparse user input, providing real-time visual feedback to guide the training process [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (scalar or color), user-provided sample points (painted regions on slicing planes), and network hyperparameters (input dimensionality, hidden node count).
*   **Output:** A classified volume where each voxel has an uncertainty value (0 to 1) indicating membership in the material of interest, or a real-time rendered visualization using hardware-accelerated ray casting.

**Data Structures:**
*   **Sample Point Table:** Stores training data for each painted voxel. For scalar data, each entry contains:
    *   Scalar value of the voxel ($v$).
    *   Gradient magnitude ($g$).
    *   Scalar values of neighbors (e.g., 6 neighbors: up, down, left, right, front, back).
    *   Position coordinates ($x, y, z$).
    *   Desired output label ($L \in \{0, 1\}$), where 1 indicates membership in the material class and 0 indicates non-membership [S1, S2].
*   **Neural Network:** A three-layer feed-forward perceptron.
    *   **Input Layer:** $m$ inputs. For scalar data with 6 neighbors, $m = 1 (\text{value}) + 1 (\text{gradient}) + 6 (\text{neighbors}) + 3 (\text{position}) = 11$ [S1]. For color data with 6 neighbors, $m = 3 (\text{RGB}) + 18 (\text{neighbor RGB}) + 3 (\text{position}) = 24$ [S1]. For color data with 12 neighbors, $m = 3 + 36 + 3 = 42$ [S2].
    *   **Hidden Layer:** $n$ nodes. Fully connected to input and output layers.
    *   **Output Layer:** 1 node, producing a scalar uncertainty value $U \in [0, 1]$ [S1, S2].
*   **Weights:** Matrix $W_{in}$ (input-to-hidden) and $W_{out}$ (hidden-to-output). Initialized randomly [S1, S2].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize network weights $W_{in}$ and $W_{out}$ with random values [S1, S2].
    *   Initialize empty sample point table [S2].

2.  **User Interaction (Painting):**
    *   User selects a slicing plane (x, y, or z-aligned).
    *   User paints regions using two colors:
        *   Color A (e.g., red/pink): Indicates voxels belonging to the material class ($L=1$).
        *   Color B (e.g., blue): Indicates voxels not belonging to the class ($L=0$) [S1, S2].
    *   Painted voxels are recorded in the sample point table with their corresponding features and labels [S1, S2].
    *   Painted regions are projected and shown on other slicing planes for context [S1, S2].

3.  **Training (Iterative):**
    *   Training occurs in the system idle loop to maintain interactivity [S1, S2].
    *   For each training iteration:
        a.  **Forward Propagation:** For each sample point $j$ in the table:
            $$ I_j = \text{FeatureVector}_j $$
            $$ H_j = \sigma(W_{in} \cdot I_j + b_{in}) $$
            $$ U_j = \sigma(W_{out} \cdot H_j + b_{out}) $$
            Where $\sigma(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid activation function [S1, S2].
        b.  **Error Calculation:**
            $$ E_j = U_j - L_j $$
            $$ \text{Total Error} = \sum E_j^2 $$
        c.  **Back-Propagation:** Adjust weights $W_{in}, W_{out}$ to minimize Total Error using the Back-Propagation Network (BPN) algorithm [S1, S2].
    *   After each iteration, update the classified volume or slice visualization [S1, S2].

4.  **Classification and Rendering:**
    *   **Software Preclassification:** Apply the trained network to all voxels in the volume to generate a scalar "uncertainty volume" [S1]. Render this volume using standard volume rendering techniques (e.g., ray casting) where opacity is mapped from the uncertainty value [S1].
    *   **Hardware-Assisted Rendering:** Implement the neural network in a pixel shader. During ray casting, for each sample point along the ray:
        a.  Sample the volume to get voxel value, gradient, and neighbors [S2].
        b.  Compute the input vector $I$ [S2].
        c.  Execute the neural network forward pass in the shader to compute uncertainty $U$ [S2].
        d.  Use $U$ as the opacity $\alpha$ in the front-to-back compositing equation [S2].
    *   **Multi-Material Classification:** Create separate neural networks for each material class. Train only the newest class. Render using multiple passes: precomputed volumes for previous classes, and hardware shader for the newest class [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sigmoid(x):
    """Sigmoid activation function."""
    return 1.0 / (1.0 + math.exp(-x))

class NeuralNetwork:
    def __init__(self, input_dim, hidden_dim):
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        # Initialize weights randomly [S1, S2]
        self.W_in = [[random.uniform(-1, 1) for _ in range(input_dim)] for _ in range(hidden_dim)]
        self.b_in = [0.0 for _ in range(hidden_dim)]
        self.W_out = [random.uniform(-1, 1) for _ in range(hidden_dim)]
        self.b_out = 0.0

    def forward(self, input_vector):
        """Compute output uncertainty."""
        hidden = []
        for i in range(self.hidden_dim):
            net_input = sum(self.W_in[i][j] * input_vector[j] for j in range(self.input_dim)) + self.b_in[i]
            hidden.append(sigmoid(net_input))
        
        net_output = sum(self.W_out[i] * hidden[i] for i in range(self.hidden_dim)) + self.b_out
        return sigmoid(net_output), hidden

    def train_step(self, samples, learning_rate=0.1):
        """One step of back-propagation training."""
        # Accumulate gradients (simplified for brevity, standard BPN)
        # In practice, this involves computing delta for output and hidden layers
        # and updating weights accordingly [S1, S2]
        pass # Implementation details of BPN are standard and omitted for brevity

def get_features(volume, x, y, z, dims):
    """Extract feature vector for a voxel."""
    # Scalar value
    val = volume[x][y][z]
    # Gradient magnitude (central difference) [S1, S2]
    gx = volume[x+1][y][z] - volume[x-1][y][z] if 0 < x < dims[0]-1 else 0
    gy = volume[x][y+1][z] - volume[x][y-1][z] if 0 < y < dims[1]-1 else 0
    gz = volume[x][y][z+1] - volume[x][y][z-1] if 0 < z < dims[2]-1 else 0
    grad_mag = math.sqrt(gx**2 + gy**2 + gz**2)
    
    # Neighbors (6 neighbors)
    neighbors = [
        volume[x+1][y][z], volume[x-1][y][z],
        volume[x][y+1][z], volume[x][y-1][z],
        volume[x][y][z+1], volume[x][y][z-1]
    ]
    
    # Position
    pos = [x, y, z]
    
    return [val, grad_mag] + neighbors + pos

def paint_volume(volume, dims, user_paints):
    """
    user_paints: list of (x, y, z, label)
    label: 1 for material, 0 for background
    """
    samples = []
    for x, y, z, label in user_paints:
        features = get_features(volume, x, y, z, dims)
        samples.append((features, label))
    return samples

def render_with_neural_network(volume, dims, network, ray_origin, ray_direction, step_size=0.001, threshold=0.99):
    """
    Hardware-assisted rendering simulation.
    For each sample along the ray, compute uncertainty via network.
    """
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    t = random.uniform(0.0, step_size) # Random offset to prevent banding [S18, unofficial]
    
    # Compute ray exit [G]
    t_end = compute_ray_exit(ray_origin, ray_direction, [0, dims[0]], [0, dims[1]], [0, dims[2]])
    
    while accumulated_opacity < threshold and t < t_end:
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        # Map to voxel coordinates
        vx, vy, vz = [int(p * d) for p, d in zip(pos, dims)]
        
        # Clamp to bounds
        vx = max(0, min(vx, dims[0]-1))
        vy = max(0, min(vy, dims[1]-1))
        vz = max(0, min(vz, dims[2]-1))
        
        # Get features for this voxel
        features = get_features(volume, vx, vy, vz, dims)
        
        # Neural network classification
        uncertainty, _ = network.forward(features)
        
        # Use uncertainty as opacity
        alpha = uncertainty
        
        # Front-to-back compositing [S2]
        # Assume base color is white or derived from volume intensity
        color = [1.0, 1.0, 1.0] 
        
        accumulated_color[0] += (1.0 - accumulated_opacity) * alpha * color[0]
        accumulated_color[1] += (1.0 - accumulated_opacity) * alpha * color[1]
        accumulated_color[2] += (1.0 - accumulated_opacity) * alpha * color[2]
        accumulated_opacity += (1.0 - accumulated_opacity) * alpha
        
        t += step_size
        
    return accumulated_color

def compute_ray_exit(origin, direction, bounds_x, bounds_y, bounds_z):
    # Slab intersection [G]
    t0 = max((bounds_x[0] - origin[0]) / direction[0] if direction[0] != 0 else float('inf'),
             (bounds_y[0] - origin[1]) / direction[1] if direction[1] != 0 else float('inf'),
             (bounds_z[0] - origin[2]) / direction[2] if direction[2] != 0 else float('inf'))
    t1 = min((bounds_x[1] - origin[0]) / direction[0] if direction[0] != 0 else float('inf'),
             (bounds_y[1] - origin[1]) / direction[1] if direction[1] != 0 else float('inf'),
             (bounds_z[1] - origin[2]) / direction[2] if direction[2] != 0 else float('inf'))
    return max(t0, t1)
```

**Edge Cases and Pitfalls:**
*   **Generalization Failure:** If position information is included in the input vector, painted voxels must be spread out across different slices. Painting only on one slice leads to poor generalization because the network learns position-specific features rather than material-specific features [S1, S2].
*   **Hardware Limits:** Hardware implementations are limited by pixel shader instruction counts and data passing limits. The number of hidden nodes is restricted (e.g., max 8 in the cited hardware) and should be divisible by 4 for vectorization efficiency [S2].
*   **Interpolation vs. Classification Order:** Software preclassification interpolates the scalar uncertainty volume, while hardware postclassification interpolates the input data before classification. This can lead to slight visual differences [S1].

## Subtopics

### Hardware vs Software Neural Network Rendering
Software classification applies the neural network to every voxel in the volume as a preprocessing step, generating a scalar uncertainty volume that is then rendered using standard hardware techniques. This allows for arbitrary network sizes and no graphics hardware dependency but is slow (e.g., 9 seconds for a 256³ volume) [S1]. Hardware classification implements the neural network in a pixel shader, computing uncertainty on-the-fly during ray casting. This enables real-time interactivity (e.g., 1.5 fps for 256³ volume) but is constrained by shader instruction limits and data bandwidth, restricting network size [S2].

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of samples and iterations. Software classification is $O(V \cdot N)$ per application, where $V$ is volume size and $N$ is network size, making it slow for real-time feedback [S1]. Hardware rendering is $O(R \cdot S \cdot N)$ per frame, where $R$ is rays, $S$ is samples per ray, and $N$ is network operations, limited by GPU rasterization speed [S2].
*   **Memory Complexity:** Software requires storing the full uncertainty volume ($O(V)$). Hardware requires storing network weights (small, <1 KB) and volume data [S2].
*   **Performance:** On a 2.8 GHz Pentium 4, software classification of a 256³ volume with 11 inputs and 8 hidden nodes took 9 seconds [S1]. On an ATI Radeon 9700 Pro, hardware rendering of a 256³ volume to a 512×512 window achieved 1.5 fps [S2].
*   **Strengths:** Intuitive user interface (painting), automatic high-dimensional classification, real-time feedback (hardware), eliminates manual transfer function tuning [S1, S2].
*   **Limitations:** Requires user input (painting), generalization depends on sample distribution, hardware limits network complexity, interpolation artifacts may differ between software and hardware modes [S1, S2].
*   **When to Use:** Interactive exploration of complex volume data where manual transfer function definition is difficult or time-consuming.
*   **When Not to Use:** When fully automatic classification is required without user input, or when extremely high network complexity is needed beyond hardware limits.

## Background
The method was introduced by Huang and Ma in 2003 [S2] and further detailed in 2005 [S1]. It builds on the concept of using artificial neural networks for volume data classification, replacing traditional transfer function interfaces with a painting-based user interface. The approach leverages the back-propagation algorithm for training and hardware acceleration for real-time visualization. It is primarily used in biomedical imaging and scientific visualization for segmenting and visualizing complex structures [S1, S2].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] A novel interface for higher-dimensional classification of volume data, https://doi.org/10.1109/visual.2003.1250413, fetched 2026-10-27
