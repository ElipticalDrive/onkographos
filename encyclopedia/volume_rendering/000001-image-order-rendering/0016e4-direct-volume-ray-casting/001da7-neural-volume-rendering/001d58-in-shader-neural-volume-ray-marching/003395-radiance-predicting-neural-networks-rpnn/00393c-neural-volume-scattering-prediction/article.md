# Neural Volume Scattering Prediction

Also called: Backbone Neural Network for Volume Rendering, Deep scattering for volumes, Radiance-predicting neural network for volumes

## Overview
Neural Volume Scattering Prediction is a method that uses a neural network to predict the light scattering distribution at the centers of sampling templates within a volume medium, which is then used to solve the Radiative Transfer Equation (RTE) for rendering [S1, unofficial]. It is distinguished from standard ray marching by replacing the iterative integration of scattering events with a direct prediction of the scattering distribution based on pre-calculated density, transmission, and phase features [S1, unofficial]. It builds on [Radiance-Predicting Neural Networks (RPNN)](7049-radiance-predicting-neural-networks-rpnn.md) by specifically structuring the network to handle diffuse and specular components separately and using an attention mechanism to weigh channel importance [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Pre-calculated sampling features for 4129 sampling templates, derived from particle positions and template points [S1, unofficial]. These features are categorized into three types: density ($\rho$), transmission ($T$), and phase function ($f$) [S1, unofficial].
    *   Configuration parameters:
        *   Asymmetry parameters of the Henyey-Greenstein (HG) phase function ($g$). The number of parameters depends on the material type (1, 2, or 3) [S1, unofficial].
        *   Albedo ($\eta_c$) as a 3D vector for RGB channels, with values between 0 and 1. For gaseous materials, the range is 0 to 0.5; for solid, liquid, and skin materials, the range is 0.5 to 1 [S1, unofficial]. The medium is assumed homogeneous, so albedo is constant within the same material [S1, unofficial].
        *   Angle ($\alpha$) between the viewing direction ($\omega$) and light direction ($l$), calculated as $\alpha = \cos^{-1}(\omega, l)$ [S1, unofficial].
*   **Outputs:**
    *   The scattering distribution $F(p, l)$ for the center point of each sampling template [S1, unofficial].
    *   These predicted values are substituted into the Radiative Transfer Equation to render the volume [S1, unofficial].

**Data Structures**
*   **Sampling Templates:** A fixed set of 4129 templates [S1, unofficial].
*   **Feature Vectors:** For the $i$-th layer, input features $H_i^\tau$ are pooled to create an 8-dimensional vector $v_i$ for the attention module:
    $$ v_i = \{B_i^\rho, B_i^f, B_i^T, f_i^\rho, f_i^f, f_i^T, g, \alpha\} $$
    where $\tau \in \{\rho, f, T\}$ represents density, phase, and transmission respectively [S1, unofficial].
    *   $B_i^\tau = \frac{1}{n_i} \sum_{j=1}^{n_i} H_{i,j}^\tau$ (Average pooling) [S1, unofficial].
    *   $f_i^\tau = \max H_{i,j}^\tau$ (Maximum pooling) [S1, unofficial].
    *   $n_i$ is the total number of points in the $i$-th layer template [S1, unofficial].

**Algorithm Steps**
The network consists of two main branches: Diffuse Reflection and Specular Reflection. Each branch processes density, transmission, and phase features using identical sub-network structures [S1, unofficial].

1.  **Initialization:** Prepare the 4129 sampling templates and pre-calculate density, transmission, and phase features for all templates based on particle positions [S1, unofficial].
2.  **Diffuse Branch Processing:**
    *   The diffuse branch uses a sampling template with 8 layers [S1, unofficial].
    *   For each feature type (density, transmission, phase), input the features layer-by-layer into the network structure 8 times [S1, unofficial].
    *   **Layer Processing (for each layer $i$):**
        a.  Fuse the current layer's sampling feature with the configuration parameters ($g, \eta_c, \alpha$) [S1, unofficial].
        b.  Pass the fused vector into the Attention Module to determine channel importance [S1, unofficial].
        c.  Fuse the attention module's output with the next layer's sampling feature using Fully Connected layer FC-1 to match dimensions [S1, unofficial].
        d.  Pass the result through Fully Connected layer FC-2 [S1, unofficial].
        e.  Fuse the output of FC-2 with the current layer's sampling feature and output the result [S1, unofficial].
    *   After 8 iterations, the diffuse reflection sampling density features are obtained [S1, unofficial].
3.  **Specular Branch Processing:**
    *   The specular branch uses a sampling template with 4 layers [S1, unofficial].
    *   Repeat the Layer Processing steps (2a-2e) for each feature type 4 times [S1, unofficial].
    *   After 4 iterations, the specular reflection sampling density features are obtained [S1, unofficial].
4.  **Feature Fusion:**
    *   Merge the diffuse reflection sampling density features (from step 2) and specular reflection sampling density features (from step 3) using a fully connected layer [S1, unofficial].
    *   Input the fused sampling transmission features and sampling phase features (processed similarly through their respective branches) into a Multilayer Perceptron (MLP) [S1, unofficial].
5.  **Output Prediction:**
    *   The MLP outputs the scattering distribution for the center point of the sampling template [S1, unofficial].
    *   Repeat this process in parallel for all 4129 templates to obtain the scattering distribution for all template centers [S1, unofficial].
6.  **Rendering:**
    *   Substitute the predicted scattering distributions into the Radiative Transfer Equation to render the volume medium [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_volume_scattering_prediction(particles, templates, config_params):
    """
    particles: List of particle positions
    templates: List of 4129 sampling templates
    config_params: Dict containing 'g' (HG asymmetry), 'eta' (albedo RGB vector), 'alpha' (view-light angle)
    """
    # Pre-calculate features for all templates
    # Returns dicts: density_features, transmission_features, phase_features
    # Each contains features for all 4129 templates
    density_features = precalculate_density(particles, templates)
    transmission_features = precalculate_transmission(particles, templates)
    phase_features = precalculate_phase(particles, templates)
    
    # Initialize network modules
    # 6 identical network modules: 2 per feature type (Diffuse/Specular)
    # Diffuse: 8 layers, Specular: 4 layers
    diffuse_net = create_network_structure(layers=8)
    specular_net = create_network_structure(layers=4)
    
    scattering_distributions = []
    
    for template_idx in range(4129):
        # Extract features for current template
        d_feat = density_features[template_idx]
        t_feat = transmission_features[template_idx]
        p_feat = phase_features[template_idx]
        
        # Process Diffuse Branch (8 layers)
        diffuse_density_out = process_layer_by_layer(d_feat, diffuse_net, config_params)
        diffuse_trans_out = process_layer_by_layer(t_feat, diffuse_net, config_params)
        diffuse_phase_out = process_layer_by_layer(p_feat, diffuse_net, config_params)
        
        # Process Specular Branch (4 layers)
        specular_density_out = process_layer_by_layer(d_feat, specular_net, config_params)
        specular_trans_out = process_layer_by_layer(t_feat, specular_net, config_params)
        specular_phase_out = process_layer_by_layer(p_feat, specular_net, config_params)
        
        # Merge Diffuse and Specular Density
        merged_density = fully_connect(diffuse_density_out, specular_density_out)
        
        # Merge Transmission and Phase (from both branches or processed separately?)
        # Note: Source says "sampling transmission features and sampling phase features are also input by layer 12 times... fused together"
        # Assuming we use the outputs from the respective branches
        merged_trans = fully_connect(diffuse_trans_out, specular_trans_out)
        merged_phase = fully_connect(diffuse_phase_out, specular_phase_out)
        
        # Final MLP
        scattering_dist = mlp_predict(merged_density, merged_trans, merged_phase)
        scattering_distributions.append(scattering_dist)
        
    return scattering_distributions

def process_layer_by_layer(features, network, config_params):
    """
    Processes features layer-by-layer through the network structure.
    network.layers: List of layer configurations (e.g., 8 or 4)
    """
    current_features = features
    for i, layer in enumerate(network.layers):
        # 1. Fuse current layer features with config params
        fused_input = fuse(current_features, config_params)
        
        # 2. Attention Module
        # Construct 8-dim vector v_i
        # B_i^tau = avg_pool(H_i^tau), f_i^tau = max_pool(H_i^tau)
        v_i = construct_attention_vector(current_features, i)
        
        # Weight coefficient w_i^tau = Sigmoid(ReLU(v_i))
        w_i = sigmoid(relu(v_i))
        
        # Apply attention weights to features
        attended_features = apply_attention(current_features, w_i)
        
        # 3. FC-1: Fuse attended output with next layer features (dimension matching)
        # Note: Source says "fuse attention module output with next layer's sampling density features"
        # This implies a skip connection or forward pass structure. 
        # Assuming 'next layer' refers to the subsequent layer in the network hierarchy
        if i < len(network.layers) - 1:
            next_layer_features = get_next_layer_features(features, i)
            fused_fc1 = fully_connect(attended_features, next_layer_features)
        else:
            fused_fc1 = attended_features
            
        # 4. FC-2
        fc2_output = fully_connect_2(fused_fc1)
        
        # 5. Fuse with current layer feature and output
        current_features = fuse(fc2_output, current_features)
        
    return current_features

def construct_attention_vector(features, layer_idx):
    """
    Constructs the 8-dimensional vector v_i for the attention module.
    """
    # Pooling operations for density, phase, transmission
    B_rho = average_pool(features['density'][layer_idx])
    B_f = average_pool(features['phase'][layer_idx])
    B_T = average_pool(features['transmission'][layer_idx])
    
    f_rho = max_pool(features['density'][layer_idx])
    f_f = max_pool(features['phase'][layer_idx])
    f_T = max_pool(features['transmission'][layer_idx])
    
    # Config params
    g = config_params['g']
    alpha = config_params['alpha']
    
    return [B_rho, B_f, B_T, f_rho, f_f, f_T, g, alpha]
```

**Training**
*   **Loss Function:** Mean Square Error with logarithmic transformation [S1, unofficial]:
    $$ L_S = \frac{1}{B} \sum_{i \in B} \left( \log \frac{T^c(F)}{\eta_c \gamma} + 1 - \log \frac{F_i(p, l)}{\eta_c \gamma} + 1 \right)^2 $$
    where $B$ is batch size, $T^c(F)$ is estimated value, $F_i(p, l)$ is annotation value, $\eta_c$ is albedo, $\gamma \ge 1$ is hyperparameter (typically 4), and $c$ is RGB channel [S1, unofficial].
*   **Total Loss:**
    $$ L_{total} = \argmin \frac{1}{N} \sum_{i=1}^{N} L_S $$
    where $N$ is the size of the annotated dataset $Z = \{(F_1, F(u_1, l_1)), ..., (F_N, F(u_N, l_N))\}$ [S1, unofficial].
*   **Purpose of Log Transformation:** Compresses radiation range to speed up training and reduce bright mutations caused by high-frequency phase functions [S1, unofficial].

## Subtopics

### [Neural Radiative Transfer Equation Solver](8466-neural-radiative-transfer-equation-solver.md)
Replaces iterative RTE integration with a neural network for real-time rendering of high albedo anisotropic materials. Choose when real-time performance for multiple scattering is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Parallel calculation of multiple neural network models is suggested to improve real-time rendering performance [S1, unofficial].
*   **Memory Complexity:** Not given in the sources. Requires storage for 4129 sampling templates and pre-calculated features [S1, unofficial].
*   **Performance:** Designed for real-time rendering of participating media [S1, unofficial].
*   **Strengths:** Handles multiple scattering via neural prediction; uses attention mechanism to weigh feature importance; logarithmic loss stabilizes training for high-frequency phase functions [S1, unofficial].
*   **Limitations:** Requires pre-calculation of density, transmission, and phase features; assumes homogeneous albedo within materials; specific template structure (4129 points) [S1, unofficial].
*   **When to use:** Rendering participating media where multiple scattering is significant and real-time performance is desired [S1, unofficial].
*   **When not to use:** Non-homogeneous materials with varying albedo; scenes requiring exact physical integration without neural approximation [S1, unofficial].

## Background
The method was introduced in "A real-time rendering method for high albedo anisotropic materials with multiple scattering" [S1, unofficial]. It is part of the family of Radiance-Predicting Neural Networks (RPNN) [S1, unofficial]. The approach uses a backbone neural network with three identical sub-networks for density, transmission, and phase features, incorporating an attention module based on SENet [S1, unofficial].

## Sources
[S1] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
