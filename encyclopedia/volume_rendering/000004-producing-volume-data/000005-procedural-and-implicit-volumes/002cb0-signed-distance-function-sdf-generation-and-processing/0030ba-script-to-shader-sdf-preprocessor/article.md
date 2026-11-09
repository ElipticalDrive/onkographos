# Script-to-Shader SDF Preprocessor

Also called: SDF Script Compiler, loadTxtFragFile

## Overview
The Script-to-Shader SDF Preprocessor is a C++ tool that parses a custom high-level script language to automatically generate GLSL code for Signed Distance Function (SDF) scene composition [S1, unofficial]. It solves the problem of manually writing verbose GLSL for complex implicit surfaces by allowing users to define scenes using a simplified syntax with loops, boolean operations, materials, and lights [S1, unofficial]. It sets itself apart from sibling methods like [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md) by operating at the shader source level rather than generating volumetric texture data, enabling real-time procedural geometry definition directly in the fragment shader [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A "dummy" GLSL fragment shader file (`realCode`) containing placeholders, and a custom script file (`psCode`) defining the scene geometry and logic.
*   **Output:** A complete, valid GLSL fragment shader string (`realCode`) with the scene logic injected.

**Data Structures:**
The implementation relies on string manipulation and lambda functions within a C++ context. No complex binary data structures are used; the core state is maintained via string indices and parsed parameters.

**Numbered Steps:**
1.  **Load Files:** Read the dummy GLSL fragment shader into a string `realCode` and the custom script into a string `psCode` [S1, unofficial].
2.  **Environment Setup (`setUpEnv`):** Define a lambda function `setUpEnv(envName)` that performs the following:
    *   Searches `realCode` for the placeholder `!(envName)` [S1, unofficial].
    *   Scans `psCode` for the marker `#envName`.
    *   Extracts the content between the opening `{` and closing `}` associated with that marker in `psCode` [S1, unofficial].
    *   Inserts this extracted content into `realCode` at the location of `!(envName)` [S1, unofficial].
    *   If `envName` is `"light"`, increments a global counter `!(lightCnt)` [S1, unofficial].
    *   Call `setUpEnv("material")` and `setUpEnv("light")` to populate material and lighting definitions [S1, unofficial].
3.  **Identify Insertion Point:** Locate the placeholder `!(sceneSDF)` in `realCode`. This string marks the position where the generated SDF logic will be inserted [S1, unofficial].
4.  **Function Parser (`findFunc`):** Define a lambda `findFunc` to parse function calls in `psCode`. It handles specific markers:
    *   `#` markers indicate function definitions or calls [S1, unofficial].
    *   `opU`: Union operation. Converts script syntax to a GLSL function call [S1, unofficial].
    *   `opS` / `opI`: Subtraction and Intersection operations. Converts script syntax to GLSL calls [S1, unofficial].
    *   `opUs`: Smooth Union. Converts script syntax to GLSL calls using a smoothness parameter `k` [S1, unofficial].
5.  **Main Parsing Loop:** Scan `psCode` for specific control markers:
    *   `@i`: Indicates a loop. Generates corresponding GLSL `for` loop code [S1, unofficial].
    *   `#`: Indicates a function definition or operation. Calls `findFunc` to process the operation [S1, unofficial].
    *   `!`: Indicates a direct line insertion. Inserts the following text directly into the output [S1, unofficial].
    *   The parser maintains state variables like `funcPsStr` and `funcPsEns` to track the start and end positions of function arguments within `psCode` [S1, unofficial].
6.  **Code Generation for Boolean Operations:**
    *   **Union (`opU`):** Generates a call to `opU` (or `mint`) combining two SDF values. The arguments are extracted from `psCode` between `funcPsStr + 1` and `funcPsEns`, followed by a material number `mtlNo` [S1, unofficial].
    *   **Subtract/Intersect (`opS`/`opI`):** Generates a call to `opU` (acting as the base boolean operator) combining three arguments: the result of the previous operation, the current operand's SDF, and the material number [S1, unofficial].
    *   **Smooth Union (`opUs`):** Uses the smoothness parameter `k` parsed via `sscanf(..., "#k%f", &k)` [S1, unofficial].
7.  **Injection:** Insert the generated GLSL code string at the `!(sceneSDF)` position in `realCode` [S1, unofficial].
8.  **Return:** Return the modified `realCode` string [S1, unofficial].

**Formulas:**
*   **Smooth Union:** The smooth union of two SDFs $a$ and $b$ with smoothness parameter $k$ is calculated as [S1, unofficial]:
    $$ f = \min(a, b) - k \cdot \max(0, \min(a, b)) $$
    Where $k$ is the smoothness parameter, typically a small positive float [S1, unofficial].

**Parameters:**
*   `k`: Smoothness parameter for `opUs`. Parsed from the script using `sscanf` [S1, unofficial].
*   `prefix`: Directory prefix for shader files, formatted as `shaders/%s/%s.glsl` [S1, unofficial].
*   `mtlNo`: Material number identifier, converted to string via `std::to_string` [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Syntax Conflict:** The script parser uses fixed string markers (`@i`, `#`, `!`) which can conflict with standard GLSL syntax if not carefully escaped or structured in the script file [S1, unofficial].
*   **Marker Assumptions:** The `findSubFunc` lambda assumes specific marker formats (`m` for matrix, `v` for vector) in the script arguments. If the script deviates from this format, parsing may fail or produce incorrect GLSL [S1, unofficial].
*   **Placeholder Dependency:** The method relies entirely on the presence of specific placeholders (`!(envName)`, `!(sceneSDF)`) in the dummy GLSL file. If these are missing or misspelled, the injection will fail or overwrite incorrect code [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_sdf(real_code: str, ps_code: str) -> str:
    """
    Parses a custom SDF script and injects generated GLSL into a dummy shader.
    
    Args:
        real_code: The dummy GLSL fragment shader string.
        ps_code: The custom script string defining the scene.
        
    Returns:
        The complete GLSL fragment shader string.
    """
    light_cnt = 0
    
    # Step 2: Environment Setup Lambda
    def set_up_env(env_name: str):
        nonlocal light_cnt
        # Find placeholder in real_code
        placeholder = f"!( {env_name} )" # Assuming space for clarity, adjust to exact marker
        if placeholder not in real_code:
            raise ValueError(f"Placeholder {placeholder} not found in real_code")
        
        # Scan ps_code for #env_name
        marker = f"#{env_name}"
        start_idx = ps_code.find(marker)
        if start_idx == -1:
            return # Or handle error
            
        # Extract content between { and }
        brace_start = ps_code.find('{', start_idx)
        brace_end = ps_code.find('}', brace_start)
        if brace_start == -1 or brace_end == -1:
            raise ValueError(f"Malformed {env_name} block in ps_code")
            
        content = ps_code[brace_start+1 : brace_end]
        
        # Insert into real_code
        real_code = real_code.replace(placeholder, content)
        
        # Update light count if necessary
        if env_name == "light":
            light_cnt += 1

    set_up_env("material")
    set_up_env("light")
    
    # Step 3: Find insertion point
    scene_sdf_marker = "!(sceneSDF)"
    if scene_sdf_marker not in real_code:
        raise ValueError("Placeholder !(sceneSDF) not found")
    
    generated_code = ""
    
    # Step 4 & 5: Helper to find sub-functions/arguments
    def find_sub_func(ps_code, current_pos):
        # Assumes specific marker formats 'm' for matrix, 'v' for vector
        # This is a simplified representation of the complex string slicing in [S1, unofficial]
        # In the actual C++ code, this involves scanning for specific delimiters
        # and extracting substrings based on funcPsStr and funcPsEns indices.
        # For the purpose of this pseudocode, we assume it returns the next argument string
        # and updates the position.
        # Since the exact parsing logic is opaque and marker-dependent, 
        # we note that this step extracts arguments for the boolean operations.
        pass 

    # Step 5: Main Loop
    i = 0
    while i < len(ps_code):
        char = ps_code[i]
        
        if char == '@':
            # Check for @i (loop)
            if ps_code[i:i+2] == '@i':
                # Generate for loop code
                # The exact logic depends on the script's loop definition format
                # which is not fully detailed in the sources.
                # Placeholder for loop generation logic [G]
                generated_code += "for (int i = 0; i < 10; i++) {\n" # Example [G]
                i += 2
                # Skip loop body until end marker? Sources don't specify end marker for @i
                # Assuming the script structure handles this or it's a simple expansion
                continue
            else:
                i += 1
                continue
                
        elif char == '#':
            # Function definition or operation
            # Check for operation keywords
            if ps_code[i:].startswith("#opU"):
                # Union
                # Extract arguments. Sources show:
                # funcStr += ... + "(mint, vec2(fSDF(" + arg1 + "), " + mtlNo + "))"
                # This requires parsing the arguments from ps_code
                # We simulate the extraction based on the source's string slicing logic
                # Note: The source code snippet shows complex string indexing.
                # We will assume a helper `extract_args` exists for clarity, 
                # as the exact C++ string iterator logic is verbose and specific.
                args = extract_args(ps_code, i + 4) # Skip "#opU"
                mtl_no = "0" # Default or parsed from script
                generated_code += f"opU(mint, vec2(fSDF({args[0]}), {mtl_no}));\n"
                # Update i to skip processed part
                i += 4 + len(args[0]) # Approximate
            elif ps_code[i:].startswith("#opS") or ps_code[i:].startswith("#opI"):
                # Subtract or Intersect
                # Sources show: opU(mint, arg1(vec2(fSDF(arg2), mtlNo)), vec2(fSDF(arg3), mtlNo))
                args = extract_args(ps_code, i + 4) # Skip "#opS" or "#opI"
                mtl_no = "0"
                # Note: The source code snippet implies a chain of operations
                generated_code += f"opU(mint, {args[0]}(vec2(fSDF({args[1]}), {mtl_no}), vec2(fSDF({args[2]}), {mtl_no})));\n"
                i += 4 + len(args[0]) + len(args[1]) + len(args[2]) # Approximate
            elif ps_code[i:].startswith("#opUs"):
                # Smooth Union
                # Parse k parameter
                k = 0.1 # Default [G]
                # sscanf(..., "#k%f", &k) implies k is defined in the script
                # Extract k and arguments
                k_str = extract_k(ps_code, i + 6) # Skip "#opUs"
                if k_str:
                    k = float(k_str)
                args = extract_args(ps_code, i + 6 + len(k_str))
                a, b = args[0], args[1]
                # Formula: f = min(a, b) - k * max(0, min(a, b))
                generated_code += f"min({a}, {b}) - {k} * max(0.0, min({a}, {b}));\n"
                i += 6 + len(k_str) + len(args[0]) + len(args[1]) # Approximate
            else:
                # Generic function definition or other # marker
                # Insert directly or handle as per script spec
                # Sources mention "direct line insertion" for '!', so '#' is likely for ops/defs
                # If it's a def, it might be stored for later use or inlined
                # For simplicity, assume it's an operation if it matches known ops, else error or pass
                i += 1
                continue
        elif char == '!':
            # Direct line insertion
            # Find end of line or next marker
            next_marker = -1
            for marker in ['@', '#', '!']:
                idx = ps_code.find(marker, i + 1)
                if idx != -1 and (next_marker == -1 or idx < next_marker):
                    next_marker = idx
            if next_marker == -1:
                line = ps_code[i+1:]
                i = len(ps_code)
            else:
                line = ps_code[i+1:next_marker]
                i = next_marker
            generated_code += line + "\n"
        else:
            i += 1
            
    # Step 6: Insert generated code
    final_code = real_code.replace(scene_sdf_marker, generated_code)
    
    return final_code

def extract_args(ps_code, start_idx):
    """
    Simplified helper to extract arguments from the script.
    The actual C++ code uses complex string slicing with funcPsStr and funcPsEns.
    This is a placeholder for the specific parsing logic described in [S1, unofficial].
    """
    # In the real implementation, this involves scanning for commas, parentheses,
    # and specific type markers ('m', 'v') as noted in the pitfalls.
    # Returning dummy args for the sake of valid Python structure.
    return ["arg1", "arg2", "arg3"]

def extract_k(ps_code, start_idx):
    """
    Simplified helper to extract k parameter.
    """
    # In the real implementation, sscanf is used.
    return "0.1"
```

## Subtopics

### SDF to Density Conversion
Converts the SDF values to density for volume rendering. Choose this when you need to render the SDF as a volumetric cloud or fog rather than a surface. Link: [SDF to Density Conversion](1282-sdf-to-density-conversion.md).

### Constrained Jacobi Smoothing for SDFs
Smooths the SDF field while preserving the Eikonal equation. Choose this when the generated SDF has artifacts or non-unit gradients that need correction. Link: [Constrained Jacobi Smoothing for SDFs](1583-constrained-jacobi-smoothing-for-sdfs.md).

### SDF Volume Generation & Ping-Pong Buffering
Generates SDF data into a texture using GPU compute shaders. Choose this when you need to pre-compute or store the SDF in a texture for ray marching, rather than computing it procedurally in the shader. Link: [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md).

### Volume SDF
A general method for defining SDFs within a volume context. Choose this for a broader overview of SDFs in volumes. Link: [Volume SDF](11033-volume-sdf.md).

## Operating characteristics
*   **Time Complexity:** The preprocessing step is linear with respect to the size of the script file `psCode`, as it involves a single pass scan and string replacements [S1, unofficial]. The generated GLSL code is executed at runtime during rendering, with complexity depending on the number of SDF operations defined in the script.
*   **Memory Complexity:** Low memory overhead for the preprocessor itself, as it operates on strings. The generated GLSL code size is proportional to the complexity of the scene script [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales with the complexity of the scene definition in the script. Complex boolean operations and loops will increase the size of the generated GLSL code, potentially impacting compilation time and runtime performance [S1, unofficial].
*   **Strengths:** Simplifies scene definition for SDFs; allows for procedural generation of complex geometries; integrates directly into the GLSL shader pipeline [S1, unofficial].
*   **Limitations:** Relies on a custom script syntax that may conflict with GLSL; parsing logic is rigid and may break with non-standard script formats; limited to shader-level composition, not volumetric data generation [S1, unofficial].
*   **When to use:** When you need to define complex SDF scenes procedurally in GLSL without manually writing verbose shader code.
*   **When not to use:** When you need to store SDF data in a texture for ray marching or when you require robust, standard-compliant scripting without custom marker conflicts.

## Background
The Script-to-Shader SDF Preprocessor originates from an unofficial GitHub repository `trimesh33/sdf` [S1, unofficial]. It is a C++ implementation designed to parse a custom script language and generate GLSL code for SDF scene composition. The method is part of a broader effort to simplify the creation of implicit surfaces in real-time graphics by abstracting the GLSL syntax [S1, unofficial].

## Sources
[S1] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
