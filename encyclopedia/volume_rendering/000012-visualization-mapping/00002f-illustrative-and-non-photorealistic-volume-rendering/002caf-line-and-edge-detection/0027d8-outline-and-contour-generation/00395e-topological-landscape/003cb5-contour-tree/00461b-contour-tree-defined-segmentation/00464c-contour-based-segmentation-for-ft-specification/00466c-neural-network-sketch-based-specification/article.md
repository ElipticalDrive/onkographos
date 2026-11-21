# Neural Network Sketch-Based Specification

Also called: Tzeng et al. method

## Overview
A technique using a neural network to adjust the transfer function (FT) based on a sketch drawn on a volume visualization, training the network to emphasize marked regions [S1, unofficial]. It implements multidimensional transfer functions as back-propagation neural networks to decide voxel visibility, where training is supervised by user-selected voxels on 2D slices [S2, unofficial]. It sets itself apart from sibling methods by replacing manual parameter tuning or geometric contour separation with a learned mapping from visual sketches to multidimensional visibility decisions [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric dataset, user sketch (painted voxels on 2D slices indicating desired visibility), and initial network weights.
*   **Output:** A trained back-propagation neural network representing the multidimensional transfer function, which maps voxel signatures to visibility values [S2, unofficial].

**Data Structures:**
*   **Voxel Signature:** A multidimensional vector representing the properties of a selected voxel (e.g., scalar value, gradient magnitude). The exact dimensionality is not specified in the sources [S2, unofficial].
*   **Neural Network:** A back-propagation network. The architecture (number of layers, neurons per layer) is not specified in the sources [S2, unofficial].
*   **Labels:** Binary or categorical labels indicating "visible" or "invisible" for each selected voxel [S2, unofficial].

**Steps:**
1.  **Visualization:** The user visualizes 2D slices of the volumetric data [S2, unofficial].
2.  **Sketching/Painting:** The user paints voxels on the 2D slices with one color to indicate they should be visible in the final image, and another color to indicate they should remain invisible [S2, unofficial]. This constitutes the "sketch" [S1, unofficial].
3.  **Signature Extraction:** The multidimensional signatures of the selected voxels are extracted [S2, unofficial].
4.  **Training:** A back-propagation neural network is trained using the extracted signatures as input and the user-defined visibility labels as the target output [S2, unofficial].
5.  **Convergence:** Training continues until the network finds a specification capable of emphasizing the marked regions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nn_sketch_based_specification(volumetric_data, user_sketch):
    """
    Trains a back-propagation neural network to act as a multidimensional transfer function
    based on user sketches on 2D slices.
    
    Args:
        volumetric_data: The input volume dataset.
        user_sketch: A data structure containing user-painted voxels on 2D slices.
                     Each entry includes voxel coordinates, signature, and label (visible/invisible).
    
    Returns:
        trained_network: The trained back-propagation neural network.
    """
    # Step 1: Extract training data from user sketch
    training_inputs = []
    training_labels = []
    
    for slice in user_sketch.slices:
        for voxel in slice.painted_voxels:
            # Extract multidimensional signature for the voxel
            # Note: Exact signature components (e.g., value, gradient) are not specified [S2, unofficial]
            signature = extract_multidimensional_signature(volumetric_data, voxel.coordinates)
            
            training_inputs.append(signature)
            # Map user color to label: e.g., 1 for visible, 0 for invisible
            training_labels.append(map_color_to_label(voxel.paint_color))
            
    # Step 2: Initialize back-propagation neural network
    # Note: Network architecture (layers, neurons) is not specified [S2, unofficial]
    network = initialize_back_propagation_network()
    
    # Step 3: Train the network
    # Training is supervised using the extracted signatures and labels
    # Note: Specific loss function, learning rate, and number of epochs are not specified [S2, unofficial]
    trained_network = train_network(network, training_inputs, training_labels)
    
    return trained_network

def extract_multidimensional_signature(volume, coordinates):
    """
    Extracts the multidimensional signature for a given voxel.
    The specific dimensions of the signature are not defined in the sources [S2, unofficial].
    """
    # Placeholder for signature extraction logic
    # Typically involves scalar value, gradient, etc.
    pass

def map_color_to_label(color):
    """
    Maps the user's paint color to a visibility label.
    """
    if color == user_sketch.visible_color:
        return 1
    elif color == user_sketch.invisible_color:
        return 0
    else:
        raise ValueError("Unknown paint color")

def initialize_back_propagation_network():
    """
    Initializes a back-propagation neural network.
    Architecture details are not specified in the sources [S2, unofficial].
    """
    # Placeholder for network initialization
    pass

def train_network(network, inputs, labels):
    """
    Trains the network using back-propagation.
    Specific training parameters are not specified in the sources [S2, unofficial].
    """
    # Placeholder for training loop
    pass
```

**Parameters:**
*   **Learning Rate:** Not given in the sources.
*   **Number of Epochs:** Not given in the sources.
*   **Network Architecture:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** If the user provides no sketch or ambiguous labels, the training may fail or produce an undefined transfer function. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows intuitive specification via sketching; handles multidimensional transfer functions [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires supervised training; depends on user's ability to select representative voxels [S2, unofficial].
*   **When to use:** When users prefer interactive sketching over manual parameter tuning for multidimensional transfer functions [S1, unofficial][S2, unofficial].
*   **When not:** When real-time performance is critical and training time is prohibitive; when user input is ambiguous [S2, unofficial].

## Background
*   **Origin:** Proposed by Tzeng, Lum, and Ma in 2003 [S1, unofficial].
*   **History:** The method was further discussed in 2004 and 2005 [S2, unofficial]. It is part of the broader field of illustrative and non-photorealistic volume rendering, specifically within line and edge detection and outline generation techniques [S1, unofficial].
*   **Usage:** Used for specifying multidimensional transfer functions through user interaction [S1, unofficial][S2, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
