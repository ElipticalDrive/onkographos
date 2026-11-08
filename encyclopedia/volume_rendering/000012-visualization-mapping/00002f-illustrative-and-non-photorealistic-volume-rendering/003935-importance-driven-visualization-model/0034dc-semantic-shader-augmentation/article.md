# Semantic Shader Augmentation

Also called: dynamic shader generation for semantic mapping

## Overview
Semantic Shader Augmentation is a technique that automatically extends an input GPU shader program with rule-based rendering functionality, replacing static visualization mappings (such as fixed transfer functions) with dynamic mappings defined by semantic rules and fuzzy membership functions [S1]. It solves the problem of rigid, hard-coded semantic layers by allowing arbitrary existing shaders to be dynamically augmented with the capability to evaluate illustration rules, enabling domain experts to interactively explore semantic mappings without programming skills [S1]. Unlike the original semantic-layers approach where rule-based functionality is hard-coded a priori, or UberShader/SuperShader which combine pre-coded fragments, this method generates code fragments entirely on the fly, allowing minimal overhead and quick exploration of different semantic mappings for various visualization techniques [S1].

## How it works
The method operates as a pre-compilation and dynamic loading pipeline. It takes an initial shader program (e.g., GLSL) containing special tags and a set of semantic rules, and produces an augmented shader program that can be dynamically loaded into the GPU [S1].

### Inputs and Outputs
*   **Input:** An initial shader source file (e.g., GLSL) selected by the user. This file contains the core rendering logic and special tags surrounding variables of potential interest [S1].
*   **Input:** A set of semantic rules and membership functions, typically specified via a graphical user interface (GUI) [S1].
*   **Output:** An augmented shader program (source code string) that encodes the intended semantic mapping and is loaded dynamically into the GPU context [S1].

### Data Structures and Layouts

**1. Shader Tags**
Variables in the input shader are marked with dedicated tags to define them as semantic entities. Each tag contains three pieces of information:
*   **Specifier:** Indicates if the variable is an input semantic data property or an output visual attribute [S1].
*   **Name:** The identifier of the variable [S1].
*   **Type:** The data type of the variable (e.g., float, vec3) [S1].

**2. Membership Functions**
Membership functions, which describe the contribution of attributes to the mapping, are implemented as single-channel textures [S1].
*   **Dimensionality:** The dimensionality of the texture matches the number of dimensions of the represented data properties. For a 1D attribute, a 1D texture is used. For 3D attributes, 3D textures are used analogously [S1].
*   **Domain/Co-domain:** The domain represents the scalar values of a semantic property (derived from the semantic-property image), and the co-domain represents the corresponding membership values [S1].

**3. Semantic-Property Texture**
During Step 1 of the augmentation, visualizations of semantic data properties are rendered to a texture.
*   **Channel Mapping:**
    *   For one-dimensional variables, the **blue** color channel is used [S1].
    *   For three-dimensional variables, **all three** color channels (RGB) are used [S1].

### Parameters
*   **Compositing Method:** Selectable by the user from three options: **averaging**, **maximum**, or **slicing** [S1].
*   **Mapping Factor:** A user-adjustable scalar factor multiplied with the composite value to map it to a visible data range [S1].

### Algorithm Steps

The augmentation process consists of two main automated steps performed by a pre-compiler in main memory before the shader is loaded [S1].

**Step 1: Generate Semantic-Property Visualizations**
This step inserts functionality to render semantic data properties (the domain of the mapping) to a texture.

1.  **Parse Input Shader:** Read the input shader source file and detect all dedicated tags. Identify variables marked as semantic data properties [S1].
2.  **Generate Property Rendering Code:** Create code fragments that:
    *   Perform raycasting (or use the existing raycasting loop if present).
    *   During raycasting execution, extract the values assigned to the identified semantic variables [S1].
    *   Composite these values along the ray using the selected **compositing method** (averaging, maximum, or slicing) [S1].
    *   Map the resulting composite value to a visible color range by multiplying it with the **mapping factor** [S1].
    *   Write the result to the output texture, mapping dimensions to color channels as defined in the layout section (Blue for 1D, RGB for 3D) [S1].
3.  **Insert Code:** Insert this generated functionality into the input shader at the locations indicated by the detected tags [S1].

**Step 2: Enhance with Fuzzy Rule Evaluation**
This step inserts functionality to evaluate semantic rules using fuzzy-logic arithmetics.

1.  **Generate Rule Evaluation Code:** Create code fragments that:
    *   Implement fuzzy-logic arithmetics (as described by Rautek et al.) [S1].
    *   Sample the membership function textures (single-channel textures matching the data property dimensions) [S1].
    *   Evaluate the illustration rules based on the membership values of the semantic data properties [S1].
    *   Map the evaluated rule outputs to the visual attributes (the co-domain), which were also identified by tags in the input shader (e.g., color, opacity, sparseness parameters) [S1].
2.  **Insert Code:** Insert this generated functionality into the input shader at the locations indicated by the tags for visual attributes [S1].

**Execution:**
The resulting augmented code is used as a dynamically loaded shader program. This allows the input-shader code to be edited during runtime, and the domain expert can use the GUI to adjust rules and membership functions without modifying the shader source directly [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def semantic_shader_augmentation(input_shader_source, semantic_tags, rules, membership_functions, compositing_method, mapping_factor):
    """
    Augments an input shader with semantic rule evaluation capabilities.
    
    Args:
        input_shader_source (str): The original GLSL source code.
        semantic_tags (list): List of dicts with keys 'specifier', 'name', 'type'.
                              Specifier is 'input' (data property) or 'output' (visual attribute).
        rules (list): List of semantic rules to evaluate.
        membership_functions (dict): Maps input semantic property names to texture references.
        compositing_method (str): 'averaging', 'maximum', or 'slicing'.
        mapping_factor (float): Factor to scale composite values to visible range.
        
    Returns:
        str: The augmented shader source code.
    """
    
    # 1. Parse the input shader to find tags
    # [G] Implementation detail: Use regex or AST parser to find specific tag patterns
    # in the source code. Assume tags are comments or specific preprocessor directives.
    parsed_variables = parse_tags(input_shader_source, semantic_tags)
    
    input_properties = [v for v in parsed_variables if v['specifier'] == 'input']
    output_attributes = [v for v in parsed_variables if v['specifier'] == 'output']
    
    # 2. Generate Step 1 Code: Semantic Property Rendering
    # This code renders the input properties to a texture for rule evaluation.
    step1_code = generate_property_rendering_code(
        input_properties, 
        compositing_method, 
        mapping_factor
    )
    
    # 3. Generate Step 2 Code: Fuzzy Rule Evaluation
    # This code evaluates rules and maps to output attributes.
    step2_code = generate_rule_evaluation_code(
        output_attributes, 
        rules, 
        membership_functions
    )
    
    # 4. Insert Code into Shader
    # [G] Implementation detail: Insert step1_code at the location of the first input tag.
    # Insert step2_code at the location of the first output tag.
    # The exact insertion points are determined by the tag positions in the original source.
    augmented_shader = insert_code_at_tags(input_shader_source, step1_code, step2_code)
    
    return augmented_shader


def generate_property_rendering_code(properties, compositing_method, mapping_factor):
    """
    Generates GLSL code to compute and render semantic properties.
    """
    code = "// Semantic Shader Augmentation: Step 1 - Property Rendering\n"
    
    # Define compositing accumulator
    # [G] Assuming a standard raycasting loop context where 'color' or 'value' is accumulated.
    # We need to inject logic into the raymarching loop.
    
    for prop in properties:
        var_name = prop['name']
        var_type = prop['type']
        
        # Determine channel mapping
        if var_type == 'float':
            channel = 'b' # Blue channel for 1D
        elif var_type == 'vec3':
            channel = 'rgb' # All channels for 3D
        else:
            continue # Unsupported type
            
        # Generate compositing logic
        # [G] The specific compositing logic depends on the raycasting structure.
        # We assume the variable 'prop_value' holds the current sample value for this property.
        
        if compositing_method == 'averaging':
            # Accumulate sum and count
            code += f"prop_sum_{var_name} += prop_value;\n"
            code += f"prop_count_{var_name} += 1.0;\n"
        elif compositing_method == 'maximum':
            code += f"prop_max_{var_name} = max(prop_max_{var_name}, prop_value);\n"
        elif compositing_method == 'slicing':
            # [G] Slicing logic typically involves thresholding or masking.
            # Specific implementation depends on the definition of 'slicing' in the context.
            # Assuming it means keeping only values above a threshold or similar.
            code += f"if (prop_value > threshold) prop_slice_{var_name} = prop_value;\n"
            
    # Final mapping to texture
    code += f"final_color.{channel} = composite_value * {mapping_factor};\n"
    
    return code


def generate_rule_evaluation_code(attributes, rules, membership_functions):
    """
    Generates GLSL code to evaluate fuzzy rules and map to visual attributes.
    """
    code = "// Semantic Shader Augmentation: Step 2 - Rule Evaluation\n"
    
    # [G] Fuzzy logic arithmetic (AND, OR, NOT) needs to be implemented.
    # Standard fuzzy logic: AND = min, OR = max, NOT = 1 - x.
    
    for rule in rules:
        # Evaluate rule conditions using membership functions
        # [G] Membership functions are sampled from textures.
        # textureLookup(membership_texture, value)
        
        rule_value = evaluate_fuzzy_rule(rule, membership_functions)
        
        # Map rule value to visual attribute
        # [G] The mapping might be direct or involve another membership function for the attribute.
        for attr in attributes:
            attr_name = attr['name']
            # Apply mapping
            code += f"{attr_name} = map_rule_to_attribute(rule_value, {attr_name});\n"
            
    return code
```

### Edge Cases and Pitfalls
*   **Tagging Complexity:** Identifying and tagging suitable semantic variables in the input shader can be time-consuming. This process is performed only once by a visualization expert before the domain expert uses the abstract control [S1].
*   **Precision:** The mapping of composite values to visible color ranges uses a simple multiplication factor. More advanced mappings could be used, but the simple method was found sufficient for the purpose [S1].
*   **Dimensionality Mismatch:** Membership function textures must match the dimensionality of the data properties (1D texture for 1D attribute, etc.). Mismatches will lead to incorrect sampling [S1].

## Subtopics

### Graphically Specified Visualization Rules
A method for determining the mapping from semantic data properties to visual attributes using rules that are graphically specified and evaluated using fuzzy-logic arithmetics [S1]. It adds functionality to evaluate fuzzy visualization rules to the augmented shader, interpreting attributes of interest as fuzzy sets where membership functions describe contributions to the mapping [S1]. Unlike formal and textual rule formulations which are considered rigid and non-exploratory, this method combines rule-based rendering with graphical user interfaces for defining multi-dimensional mappings [S1]. Visual attributes (output entities) such as color, opacity, or sparseness parameters are mapped based on these rules [S1].

## Operating characteristics
*   **Time Complexity:** The augmentation process (parsing, code generation, insertion) occurs in main memory before shader loading. The runtime cost is the evaluation of fuzzy rules and membership function lookups during raycasting. The overhead is minimal compared to hard-coded solutions because code fragments are generated on the fly rather than combined from pre-coded libraries [S1].
*   **Memory Complexity:** Requires storage for the augmented shader source, the semantic-property textures (rendered during Step 1), and the membership function textures [S1].
*   **Strengths:** Allows quick exploration of different semantic mappings for different visualization techniques without individual hard-coding. Reduces implementation time for adding functionality to a given input shader. Improves code readability by separating semantic rules from core rendering logic. Enables domain experts to control visualizations without programming skills [S1].
*   **Limitations:** Identifying suitable semantic variables requires initial expert effort. The method relies on fuzzy logic, which may not capture all types of complex semantic relationships. The simple compositing methods (averaging, maximum, slicing) may not be sufficient for all visualization needs [S1].
*   **When to use:** When interactive exploration of semantic mappings is required, and the underlying rendering technique (shader) is already implemented but lacks semantic flexibility. When domain experts need to adjust visualizations without programmer intervention [S1].
*   **When not to use:** When the semantic mapping is static and known a priori, as hard-coding would be more efficient. When complex, non-fuzzy semantic relationships are required [S1].

## Background
The method was proposed by Rautek et al. as an extension to their earlier work on "semantic layers" and "interaction-dependent semantics" [S1]. The original semantic-layers approach required rule-based rendering functionality to be hard-coded in a volume rendering program a priori, restricting the ability to quickly explore different semantic mappings for different visualization techniques [S1]. The semantic shader augmentation technique was introduced to overcome this rigidity by using dynamic shader generation [S1]. It builds on the concept of using fuzzy logic for visualization mapping, as described by Rautek et al. [1,2], and combines it with graphical user interfaces for defining multi-dimensional mappings, addressing the limitations of formal and textual rule formulations [S1]. The approach differs from UberShader and SuperShader, which use pre-coded fragments, by generating code fragments entirely on the fly [S1].

## Sources
[S1] Semantics by analogy for illustrative volume visualization, https://doi.org/10.1016/j.cag.2011.10.006, fetched 2026-10-26
