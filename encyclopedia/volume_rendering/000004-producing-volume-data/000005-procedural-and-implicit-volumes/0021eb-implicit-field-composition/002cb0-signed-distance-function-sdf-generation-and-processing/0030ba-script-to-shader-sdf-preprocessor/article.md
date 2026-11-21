# Script-to-Shader SDF Preprocessor

Also called: SDF Script Compiler, loadTxtFragFile

## Overview
The Script-to-Shader SDF Preprocessor is a C++ utility that parses a custom high-level script language to generate GLSL code for Signed Distance Function (SDF) scene composition [S1, unofficial]. It solves the problem of manually writing complex, nested GLSL SDF logic by allowing users to define scenes using declarative markers for boolean operations, loops, and materials [S1, unofficial]. It sets itself apart from sibling methods like [Volume SDF](11033-volume-sdf.md) or [RobotSDF](13070-robotsdf.md) by operating at the shader source generation stage rather than producing volumetric data or runtime geometry; it transforms a textual description of a scene into a functional fragment shader capable of rendering implicit surfaces [S1, unofficial].

## How it works
The method takes a "dummy" GLSL fragment shader template and a custom script file as input, and outputs a complete, valid GLSL fragment shader file as output [S1, unofficial].

### Data Structures
The implementation relies on string manipulation and lambda functions within a C++ context. Key internal states include:
- `realCode`: The GLSL template string, modified in place.
- `psCode`: The script source string.
- `sceneR`: The insertion point index in `realCode` where generated SDF logic is placed.
- `lightCnt`: An integer counter tracking the number of light environments defined.
- `mtlNo`: An integer material identifier used in generated function calls.

### Steps
1. **Load Inputs**: Read the dummy GLSL fragment shader into `realCode` and the custom script into `psCode` [S1, unofficial].
2. **Define Environment Handler**: Create a lambda `setUpEnv(envName)` that:
    - Searches `realCode` for the marker `!(envName)` [S1, unofficial].
    - Scans `psCode` for the marker `#envName` [S1, unofficial].
    - Extracts the content between the opening `{` and closing `}` associated with that marker in the script.
    - Inserts this extracted content into `realCode` at the `!(envName)` position.
    - If `envName` is `"light"`, increments `lightCnt` [S1, unofficial].
3. **Process Environments**: Call `setUpEnv("material")` followed by `setUpEnv("light")` to inject material and lighting definitions into the shader template [S1, unofficial].
4. **Identify Insertion Point**: Locate the marker `!(sceneSDF)` in `realCode`. This index becomes `sceneR`, the location where the generated SDF composition code will be inserted [S1, unofficial].
5. **Define Function Parser**: Create a lambda `findFunc` that:
    - Parses markers starting with `#` in `psCode` [S1, unofficial].
    - Identifies boolean operations: `opU` (union), `opS` (subtract), `opI` (intersect), and `opUs` (smooth union) [S1, unofficial].
    - Converts script syntax into corresponding GLSL function calls.
6. **Main Parsing Loop**: Iterate through `psCode` handling specific markers:
    - `@i`: Generates `for` loop structures in the output GLSL [S1, unofficial].
    - `#`: Triggers function definition parsing via `findFunc` [S1, unofficial].
    - `!`: Inserts the subsequent line directly into the output [S1, unofficial].
7. **Insert Generated Code**: Append the code generated in steps 5-6 to `realCode` at the `sceneR` position [S1, unofficial].
8. **Return Result**: Return the modified `realCode` as the final shader source [S1, unofficial].

### Formulas
The smooth union operation `opUs` uses a blending parameter $k$ [S1, unofficial]. The GLSL implementation corresponds to the following mathematical operation for inputs $a$ and $b$:

$$ f = \min(a, b) - k \cdot \max(0, \min(a, b)) $$

Where:
- $a, b$: The signed distance values of the two primitives being combined.
- $k$: The smoothness parameter, parsed from the script using `sscanf(..., "#k%f", &k)` [S1, unofficial].

### Parameters
- `k`: A float value controlling the smoothness of the union operation for `opUs`. Parsed directly from the script file [S1, unofficial].
- `prefix`: A string defining the directory path for shader files, formatted as `shaders/%s/%s.glsl` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def script_to_shader(real_code: str, ps_code: str) -> str:
    """
    Parses a custom script language to generate GLSL code for SDF scene composition.
    """
    light_cnt = 0
    mtl_no = 0  # Assumed initial material number [G]
    
    # Step 2: Define setUpEnv lambda
    def setUpEnv(env_name: str):
        nonlocal light_cnt
        # Find insertion point in real_code
        marker = f"!( {env_name} )"
        if marker not in real_code:
            return # Or raise error
        
        # Find corresponding block in ps_code
        # Note: Exact parsing logic for block extraction is implied by source [S1, unofficial]
        # We assume a helper extract_block(ps_code, f"#{env_name}") exists or is implemented inline
        block_content = extract_block(ps_code, f"#{env_name}")
        
        # Insert into real_code
        real_code = real_code.replace(marker, block_content)
        
        if env_name == "light":
            light_cnt += 1

    # Step 3: Process environments
    setUpEnv("material")
    setUpEnv("light")

    # Step 4: Find insertion point
    scene_marker = "!(sceneSDF)"
    if scene_marker not in real_code:
        raise ValueError("Scene SDF marker not found in template")
    
    scene_r = real_code.find(scene_marker)
    
    # Step 5 & 6: Parse functions and main loop
    generated_sdf_code = ""
    
    # Helper to parse sub-functions/arguments based on markers 'm' (matrix) and 'v' (vector) [S1, unofficial]
    def find_sub_func(current_pos: int):
        # Implementation details for parsing 'm' and 'v' markers are not fully specified in source [S1, unofficial]
        # This is a placeholder for the specific parsing logic required to extract arguments
        pass 

    i = 0
    while i < len(ps_code):
        char = ps_code[i]
        
        if char == '@':
            # Handle @i as for loop generation [S1, unofficial]
            # Extract loop parameters and generate GLSL for loop
            generated_sdf_code += generate_for_loop(ps_code, i)
            # Advance i past the loop definition
            i = advance_past_loop(ps_code, i)
            
        elif char == '#':
            # Handle function definition [S1, unofficial]
            # Identify operation type (opU, opS, opI, opUs)
            func_type = identify_func_type(ps_code, i)
            
            if func_type == "opU":
                # Union: min(a, b)
                # Source snippet: funcStr += ... "(mint, vec2(fSDF(...), mtlNo))"
                # Parsing arguments from ps_code
                arg1_end = find_next_marker(ps_code, i)
                arg1 = ps_code[i+1:arg1_end]
                i = arg1_end
                
                # Recursively find sub-function for second arg if needed, or parse next
                # The source shows a complex string concatenation for opU
                # Simplified generation:
                generated_sdf_code += f"min({arg1}, vec2(fSDF(...), {mtl_no}))\n"
                
            elif func_type in ["opS", "opI"]:
                # Subtract/Intersect: opU(mint, opU(..., ...))
                # Source snippet shows nested opU calls for S/I
                # This requires parsing two operands
                generated_sdf_code += generate_boolean_op(func_type, ps_code, i, mtl_no)
                
            elif func_type == "opUs":
                # Smooth Union: min(a,b) - k * max(0, min(a,b))
                # Parse k parameter
                k_val = parse_k_parameter(ps_code, i)
                generated_sdf_code += generate_smooth_union(ps_code, i, k_val)
                
            i = advance_past_func(ps_code, i)
            
        elif char == '!':
            # Direct line insertion [S1, unofficial]
            line_end = ps_code.find('\n', i)
            if line_end == -1:
                line_end = len(ps_code)
            generated_sdf_code += ps_code[i+1:line_end] + "\n"
            i = line_end + 1
            
        else:
            i += 1

    # Step 7: Insert generated code
    final_code = real_code[:scene_r] + generated_sdf_code + real_code[scene_r + len(scene_marker):]
    
    return final_code

def extract_block(text: str, marker: str) -> str:
    """Helper to extract content between { and } after marker."""
    start = text.find(marker)
    if start == -1: return ""
    brace_start = text.find('{', start)
    if brace_start == -1: return ""
    
    depth = 0
    end = brace_start
    for j in range(brace_start, len(text)):
        if text[j] == '{':
            depth += 1
        elif text[j] == '}':
            depth -= 1
            if depth == 0:
                end = j
                break
    return text[brace_start+1:end]

def generate_boolean_op(op_type: str, ps_code: str, pos: int, mtl_no: int) -> str:
    """
    Generates GLSL for opS (subtract) or opI (intersect).
    Based on source [S1, unofficial]:
    opS/I uses nested opU calls.
    """
    # This is a structural approximation based on the provided C++ snippet
    # The snippet shows: opU(mint, opU(..., vec2(fSDF(...), mtlNo)), vec2(fSDF(...), mtlNo))
    # Exact argument parsing logic is not fully detailed in the source notes [S1, unofficial]
    return f"// op{op_type} generation placeholder based on source structure [S1, unofficial]"

def generate_smooth_union(ps_code: str, pos: int, k: float) -> str:
    """
    Generates GLSL for opUs.
    Formula: min(a, b) - k * max(0, min(a, b))
    """
    # Argument parsing placeholder [S1, unofficial]
    return f"// opUs generation with k={k} [S1, unofficial]"
```

### Pitfalls
- **Marker Conflicts**: The script parser uses fixed string markers (`@i`, `#`, `!`) which can conflict with standard GLSL syntax if not carefully escaped or structured in the script file [S1, unofficial].
- **Sub-function Assumptions**: The `findSubFunc` lambda assumes specific marker formats (`m` for matrix, `v` for vector) and may break if the script deviates from these expected formats [S1, unofficial].
- **Precision**: The smooth union parameter $k$ is parsed as a float. Very small or very large values of $k$ may lead to visual artifacts or numerical instability in the distance field [S1, unofficial].

## Subtopics

### SDF to Density Conversion
Converts the scalar SDF field into density values for volume rendering. Choose this when you need to render the SDF as a volumetric cloud or fog rather than a surface. [SDF to Density Conversion](1282-sdf-to-density-conversion.md)

### Constrained Jacobi Smoothing for SDFs
Applies smoothing to the SDF field to reduce aliasing and improve normal estimation. Choose this when the generated SDF is too noisy or jagged for ray marching. [Constrained Jacobi Smoothing for SDFs](1583-constrained-jacobi-smoothing-for-sdfs.md)

### SDF Volume Generation & Ping-Pong Buffering
Generates the SDF volume data using GPU ping-pong buffers. Choose this when you need to pre-compute the SDF into a texture for ray marching, rather than computing it in the shader. [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md)

### Volume SDF
Renders SDFs directly as volumes. Choose this for specific volumetric rendering techniques that utilize the SDF as a density proxy. [Volume SDF](11033-volume-sdf.md)

### RobotSDF
Specialized SDF generation for robotic environments. Choose this when the scene consists of robot kinematics or specific industrial geometries. [RobotSDF](13070-robotsdf.md)

### Volume Fairing via Signed Distance Functions
Smooths and fairing volumetric data using SDF properties. Choose this for cleaning up noisy volumetric scans. [Volume Fairing via Signed Distance Functions](14492-volume-fairing-via-signed-distance-functions.md)

## Operating characteristics
- **Time Complexity**: The preprocessing time is linear with respect to the size of the script file `psCode`, as it involves a single pass parsing and string concatenation [S1, unofficial].
- **Memory Complexity**: Linear with respect to the size of the input script and the output GLSL shader [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Strengths**: Allows for rapid prototyping of complex SDF scenes using a high-level script; integrates boolean operations and loops directly into the shader source [S1, unofficial].
- **Limitations**: The custom script syntax is rigid and prone to errors if markers are misused; debugging requires inspecting the generated GLSL code rather than the script itself [S1, unofficial].
- **When to use**: When you need to compose complex SDF scenes in a shader without manually writing nested GLSL functions.
- **When not to use**: When you need to pre-compute SDF data into a texture (use [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md)) or when the scene is static and better suited for mesh-based rendering.

## Background
The method originates from an unofficial C++ implementation found in the `trimesh33/sdf` repository [S1, unofficial]. It is designed to work with GLUT 3.1+ and Mesa 2.5+ environments, leveraging early GLUT 4 routines and signal handling [S1, unofficial]. The approach is part of a broader effort to simplify SDF scene composition in real-time graphics by abstracting the GLSL boilerplate [S1, unofficial].

## Sources
[S1] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
