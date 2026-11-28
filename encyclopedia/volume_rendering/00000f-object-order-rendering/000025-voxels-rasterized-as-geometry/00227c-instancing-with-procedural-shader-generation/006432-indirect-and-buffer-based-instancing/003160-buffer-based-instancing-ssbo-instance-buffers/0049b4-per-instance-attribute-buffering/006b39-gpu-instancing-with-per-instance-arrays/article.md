# GPU Instancing with Per-Instance Arrays

Also called: DrawMeshInstanced Arrays, MaterialPropertyBlock Instancing

## Overview
This method enables GPU instancing by passing variable-length or fixed-size per-instance data (such as arrays of floats) via `MaterialPropertyBlock` (MPB) rather than standard vertex buffers. It solves the limitation where standard instancing buffers require fixed-size attributes, allowing shaders to access arbitrary per-instance parameters. It differs from [Per-Instance Attribute Buffering](12563-per-instance-attribute-buffering.md) by using MPB for data transfer instead of SSBOs/UBOs, and from [GPU Instancing with Texture Buffer Objects](22065-gpu-instancing-with-texture-buffer-objects.md) by relying on engine-specific property blocks rather than raw texture buffers.

## How it works
**Inputs:** A mesh, a material, and a list of per-instance data arrays (e.g., `float[]` of size $N$).
**Outputs:** Rendered instances with unique per-instance attributes.

**Data Structures:**
1.  **Instance Data Buffer:** A flattened 1D array of floats. If each instance requires $K$ values, the buffer size is $K \times I$, where $I$ is the instance count.
2.  **MaterialPropertyBlock:** Holds the flattened array as a property (e.g., `FloatArray`).

**Steps:**
1.  **Flatten Data:** For $I$ instances, each needing $K$ values, create a single array `data` of size $I \times K$.
    *   For instance $i$ and value index $j$ ($0 \le j < K$), the value $v_{i,j}$ is stored at index $i \times K + j$.
2.  **Assign to MPB:** Use the engine's API (e.g., `MaterialPropertyBlock.SetFloatArray`) to assign the flattened array to a named property on the material.
3.  **Draw Instanced:** Call the instanced draw command (e.g., `Graphics.DrawMeshInstanced`) passing the MPB and instance count $I$.
4.  **Shader Access:** In the vertex/fragment shader, use engine-specific macros (e.g., `UNITY_ACCESS_INSTANCED_PROP`) to read the property.
5.  **Indexing:** Calculate the specific value for instance $i$ and component $j$ using manual indexing: `index = i * K + j`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# C# Side (Engine API)
def render_instanced_with_arrays(mesh, material, instance_count, per_instance_data):
    """
    per_instance_data: List of lists, where each inner list has K floats.
    """
    K = len(per_instance_data[0])
    
    # Flatten the 2D data into a 1D array
    flattened_data = []
    for i in range(instance_count):
        for j in range(K):
            flattened_data.append(per_instance_data[i][j])
            
    # Create MPB and set the array
    mpb = MaterialPropertyBlock()
    mpb.SetFloatArray("_PerInstanceArray", flattened_data)
    
    # Draw instanced
    # Note: Standard instancing does not support different array sizes per instance.
    # All instances must use the same K.
    Graphics.DrawMeshInstanced(mesh, 0, material, Matrix4x4.identity, instance_count, mpb)

# Shader Side (HLSL/Unity)
// Accessing the flattened array
#define UNITY_ACCESS_INSTANCED_PROP(propName, varName) \
    float4 varName : PROPERTY(propName)

void vert(appdata_full v, out float4 outPos, out float instanceID) {
    instanceID = v.instanceID; // Provided by instancing hardware
    
    // Manual indexing for per-instance array
    // Assuming K=8 for this example
    int K = 8;
    int baseIndex = instanceID * K;
    
    // Access specific components
    float val0 = UNITY_ACCESS_INSTANCED_PROP("_PerInstanceArray", arr)[baseIndex + 0];
    float val1 = UNITY_ACCESS_INSTANCED_PROP("_PerInstanceArray", arr)[baseIndex + 1];
    
    // ... use val0, val1 ...
}
```

**Edge Cases and Pitfalls:**
*   **Fixed Size Requirement:** GPU instancing does not natively support different array sizes per instance. All instances must have the same number of array elements $K$ [S1, unofficial].
*   **Manual Indexing:** Accessing flattened data requires manual calculation of indices ($i \times K + j$) in the shader [S1, unofficial].
*   **Render Queue:** Changing the Render Queue to control draw order breaks instancing. There is no documented solution for per-object render queue in instanced rendering [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(I)$ for data upload and draw call, where $I$ is instance count.
*   **Memory Complexity:** $O(I \times K)$ for the flattened data array.
*   **Performance:** MPB updates can be slower than SSBO/UBO updates for large datasets due to engine overhead. Suitable for moderate $K$ and $I$.
*   **Strengths:** Easy to implement in engines like Unity; supports variable-length data conceptually (via flattening).
*   **Limitations:** No native support for variable-sized arrays per instance; render queue control is limited; manual indexing in shaders.

## Background
Originated from Unity's `MaterialPropertyBlock` system and `Graphics.DrawMeshInstanced` API. Used in real-time rendering for effects like volume clouds, particles, and dynamic objects with unique per-instance parameters.

## Sources
[S1] ShuchangXu/VolumeCloud, https://github.com/ShuchangXu/VolumeCloud, fetched 2026-10-26 (unofficial)
