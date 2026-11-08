# Remote Rendering Client/Server Architecture

Also called: Client/Server Volume Rendering, I/O subsystem, Network Rendering, Remote Rendering, Remote client, Remote server

## Overview
Remote Rendering Client/Server Architecture separates a volume rendering engine into a server component that performs computation and a client component that handles user interaction and display, connected via a network [S1][S2, unofficial]. It solves the problem of limited local hardware performance or data locality by allowing remote machines to render large datasets while the user interacts locally [S1]. Unlike local rendering, it introduces network latency and bandwidth constraints as primary bottlenecks, but enables the use of high-end dedicated servers for low-performance workstations or centralized storage for massive datasets [S1]. It serves as the foundational communication layer for higher-level distributed architectures, such as the Distributed Volume Rendering Pipeline [7166-distributed-volume-rendering-pipeline.md], where it facilitates sort-last parallel rendering across heterogeneous clusters [S2, unofficial].

## How it works
The architecture relies on a transparent proxy pattern where the client exposes an API identical to the local rendering engine, encoding function calls into network messages [S1]. The server receives these commands, interprets them, and executes the corresponding rendering or data management operations [S1].

**Inputs and Outputs:**
*   **Input:** User interaction events (viewpoint changes, parameter adjustments) and rendering requests from the client.
*   **Output:** Serialized binary image data or status updates sent from the server to the client [S1].

**Data Structures:**
The specific network packet layout is not defined in the sources, but the communication relies on serialization of function calls and binary image buffers [S1]. The I/O subsystem is designed to be replaceable, supporting various inter-process communication (IPC) protocols [S2, unofficial].

**Steps:**
1.  **Client Initialization:** The client establishes a network connection to the remote server [S2, unofficial]. The client provides an API identical to the local engine [S1].
2.  **Command Encoding:** When the user interacts with the scene (e.g., changes viewpoint or transparency), the client encodes the function call into a network message [S1].
3.  **Transmission:** The encoded command is sent over the network to the server [S1].
4.  **Server Execution:** The server receives the command, interprets it, and executes the corresponding call (e.g., modifying scene elements, assigning data files, or triggering a render) [S1].
5.  **Result Serialization:** The server serializes the result of the operation. For rendering, the result is stored in a buffer and serialized as binary data [S1].
6.  **Result Transmission:** The serialized binary data is sent back to the client over the network [S1].
7.  **Client Deserialization and Display:** The client receives the binary data, deserializes it, and uses it as if the computation were local [S1]. The image is either provided directly to the user or rendered into an OpenGL context for GUI applications [S1].

**Parameters:**
*   **Communication Protocol:** Configurable. Supports TCP/IP, InfiniBand, or shared memory [S2, unofficial]. Shared memory is used to avoid network latency when the client and server run on the same physical machine [S2, unofficial].
*   **Network Overhead:** Considered acceptable for target use cases involving low-performance workstations or very large data volumes [S1].
*   **Image Compression:** Optional. Not strictly required if the network is fast enough, but can be applied if the interconnection is slow [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class RemoteClient:
    def __init__(self, server_address, protocol='TCP/IP'):
        self.connection = establish_connection(server_address, protocol)
        # The API mirrors the local engine
        self.scene = Scene()
        self.renderer = Renderer()

    def set_viewpoint(self, position, target, up):
        # Encode function call
        message = encode_call('set_viewpoint', position, target, up)
        # Send over network
        self.connection.send(message)
        # Wait for result (synchronous) or handle callback (asynchronous)
        result = self.connection.receive()
        # Deserialize and apply locally if needed
        deserialize(result)

    def render(self):
        # Request rendering
        message = encode_call('render')
        self.connection.send(message)
        # Receive binary image data
        binary_image = self.connection.receive()
        # Render to OpenGL context or display directly
        display_image(binary_image)

class RemoteServer:
    def __init__(self, client_connection):
        self.connection = client_connection
        self.scene = Scene()
        self.renderer = Renderer()

    def handle_request(self):
        # Receive encoded command
        message = self.connection.receive()
        # Decode command
        command, args = decode_call(message)
        
        # Execute corresponding call
        if command == 'set_viewpoint':
            self.scene.set_viewpoint(*args)
        elif command == 'render':
            # Perform rendering
            image_buffer = self.renderer.render(self.scene)
            # Serialize result
            serialized_result = serialize(image_buffer)
            # Send back
            self.connection.send(serialized_result)
```

## Subtopics

### Asynchronous UDP Client-Server Volume Rendering
A variant that uses UDP/IP for communication to minimize protocol overhead and ensure low latency/high throughput [S3, unofficial]. It implements an asynchronous protocol where the server ignores incoming packets during rendering and only processes the youngest received message after an image pair is completed, starting a new render immediately [S3, unofficial]. This strategy offers lower latency than a complete request/acknowledge cycle for each image and relies on the continuous transmission of images for idempotent updates [S3, unofficial]. It is suitable for virtual environments where delays in rendering do not impair interaction mechanisms [S3, unofficial].

### GPU Cluster Sort-last Rendering with MPI
A variant used in GPU clusters where a thin client communicates via TCP/IP as an image viewer handling user input, while server nodes communicate and synchronize using Message Passing Interface (MPI) [S4, unofficial]. Server nodes may send partial images to the client either gathered and composited or directly without compositing, with optional compression of the final image before sending [S4, unofficial]. This allows for parallel volume rendering on large-scale displays using handheld touchscreen interaction [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the computational cost of volume rendering plus network transmission time [S1].
*   **Memory Complexity:** Depends on the volume dataset size and the rendering resolution. The server must hold the volume data and rendering buffers; the client holds the image buffer [S1].
*   **Measured Performance:** Rendering a 128x256x256 volume to a 512x512 image using a 3D texture GPU renderer took 557.12 ms in a remote client/server configuration over a 100Mbit/s local network [S1]. This includes network overhead. Local rendering configurations (e.g., GPU to OpenGL local) were significantly faster (372.68 ms), indicating the network adds approximately 184 ms of overhead in this specific test case [S1].
*   **Strengths:** Enables rendering on high-end dedicated servers from low-performance workstations; allows centralized storage of large datasets without copying to each workstation; transparent API allows easy recompilation for local or remote use [S1].
*   **Limitations:** Network latency and bandwidth can degrade interactivity; reliability depends on the transport protocol (TCP vs. UDP); not suitable for real-time interaction on slow networks without compression or asynchronous techniques [S1][S3, unofficial].
*   **When to use:** When local hardware is insufficient for the dataset size or complexity; when data is too large to distribute to all clients; when using heterogeneous HPC clusters [S1][S2, unofficial].
*   **When not to use:** When low latency is critical and the network is slow/unreliable; when the dataset is small enough to render locally with acceptable performance [S1].

## Background
The architecture originated from the need to separate user interaction, scene management, and rendering in volume visualization engines [S1]. It was proposed as a straightforward extension of local rendering engines by dividing them into server and client parts with added communication capabilities [S1]. Early implementations used it to support rendering of huge datasets from projects like the Visible Human Project [S1]. It has been further generalized in software architectures for distributed volume rendering on HPC systems to support sort-last parallel rendering and heterogeneous clusters [S2, unofficial]. Variants using UDP/IP were developed for virtual environments to decouple rendering delays from interaction [S3, unofficial]. More recent implementations integrate this pattern with MPI and GPU clusters for large-scale displays [S4, unofficial].

## Sources
[S1] The VRE volume rendering engine, https://doi.org/10.1145/1925059.1925072, fetched 2026-11-05
[S2] A SOFTWARE ARCHITECTURE FOR DISTRIBUTED VOLUME RENDERING ON ... - Actapress, https://actapress.com/Abstract.aspx?paperId=454770, fetched 2026-10-27 (unofficial)
[S3] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
[S4] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
