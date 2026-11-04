# Stack-Based SDF Editor

Also called: Post-order SDF construction, RPN-style SDF editor

## Overview
An interactive editor that constructs Signed Distance Fields (SDFs) from primitives and operations using a stack-based interface similar to Reverse Polish Notation (RPN), implicitly representing a tree with postorder traversal [S1, unofficial]. It differs from explicit tree editors by deferring structure resolution until compilation, allowing users to build complex scenes by pushing operations onto a linear stack where only the top item is visible until combined [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of operations pushed onto a stack, categorized as Shapes, Transformations, or CSG Operations [S1, unofficial].
- Output: A compiled GLSL shader string containing the SDF evaluation function [S1, unofficial].

**Data Structures:**
- **Operation Stack:** A list of operation objects representing the postorder traversal of the scene graph [S1, unofficial].
- **AST Tree:** A recursive structure built from the stack where nodes are either leaves (primitive shapes annotated with domain-modifying transformations) or internal nodes (CSG operations acting on range values) [S1, unofficial].

**Numbered Steps:**
1. **Push Operation:** Operations are pushed onto the stack [S1, unofficial]. The editor only considers the first item by default; subsequent items are not visible until joined via union or other operations [S1, unofficial].
2. **Invalidate Cache:** Modifying any parameter calls `#invalidate`, which clears the cached shader program to trigger recompilation on the next access [S1, unofficial].
3. **Build SDF (Tree Reconstruction):** The `buildSDF` function iterates over the stack, popping off previous operations according to the arity of the current operation to reconstruct the tree structure [S1, unofficial].
4. **Generate GLSL:** The system recursively traverses the AST, transforming nodes to appropriate calls [S1, unofficial]. At leaves, it composes domain-modifying functions in the opposite order of application [S1, unofficial]. Internal nodes generate range operations (e.g., `min` for union) [S1, unofficial].
5. **Compile:** The generated GLSL string is spliced into a template and compiled [S1, unofficial].

**Formulas:**
- **Union:** $S_{union}(p) = \min(S_A(p), S_B(p))$ [S1, unofficial].
- **Domain Transformation Composition:** If transformations $T_1, T_2, \dots$ are applied to a shape, the input position $p$ is transformed by composing them in the opposite order: $T_n(\dots T_2(T_1(p)) \dots)$ [S1, unofficial].

**Parameters:**
- **Primitive Types:** Box and Sphere (e.g., radius 1.0) [S1, unofficial].
- **Transformations:** Translate (vec3 offset) and Rotate (float angle) [S1, unofficial].
- **CSG Operations:** Union and Intersection [S1, unofficial].
- **Configuration:** Uses lazy evaluation and caching via getter/setter methods that call `#invalidate` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Visibility:** Subsequent stack items remain invisible until a union or similar operation combines them with the top item [S1, unofficial].
- **AST vs Conceptual Tree:** The generated AST differs from the conceptual operation tree because domain modifications (transformations) are attached to leaves, while range modifications (CSG operations) form internal nodes [S1, unofficial].
- **Performance:** Without `#invalidate` caching, the shader would be reassembled and compiled on every frame, causing severe inefficiency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sdf(stack):
    ast_stack = []
    for op in stack:
        if op['type'] == 'Shape':
            ast_stack.append({'type': 'Leaf', 'shape': op['shape'], 'transforms': [], 'params': op.get('params', {})})
        elif op['type'] == 'Transform':
            if ast_stack:
                ast_stack[-1]['transforms'].append(op['transform'])
        elif op['type'] == 'CSG':
            if len(ast_stack) >= 2:
                right = ast_stack.pop()
                left = ast_stack.pop()
                ast_stack.append({'type': 'CSG', 'op': op['op'], 'left': left, 'right': right})
    return ast_stack[0] if ast_stack else None

def generate_glsl(node):
    if node['type'] == 'Leaf':
        p = "p"
        for t in reversed(node['transforms']):
            if t['type'] == 'Translate':
                p = f"opTx({p}, vec3({t['x']}, {t['y']}, {t['z']}))"
            elif t['type'] == 'Rotate':
                p = f"opRotX({p}, {t['angle']})"
        radius = node['params'].get('radius', '1.0')
        return f"sd{node['shape'].upper()}({p}, {radius})"
    elif node['type'] == 'CSG':
        left_code = generate_glsl(node['left'])
        right_code = generate_glsl(node['right'])
        if node['op'] == 'Union':
            return f"min({left_code}, {right_code})"
        elif node['op'] == 'Intersection':
            return f"max({left_code}, {right_code})"
        return f"min({left_code}, {right_code})"
    return ""
```

## Operating characteristics
Caching via `#invalidate` prevents recompilation on every frame, making runtime evaluation efficient [S1, unofficial]. The Config class uses lazy evaluation and caching to avoid unnecessary shader recompilation [S1, unofficial]. The interface is described as intuitive once the input model is understood [S1, unofficial].

## Background
The Stack-Based SDF Editor was developed as a final project for COS 426, focusing on raymarching with Signed Distance Fields [S1, unofficial]. It demonstrates building SDFs from primitives and operations using a stack-based interface similar to RPN notation [S1, unofficial].

## Sources
[S1] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
