# Stateless Parallel Volume Rendering Communication Protocol

Also called: Best-effort Volume Rendering System, Distributed Real-Time Volume Rendering System, IPv6-UDP Volume Rendering Stack, Multicast Control / Unicast Output Protocol, Sistema de Renderización Distribuida en Tiempo Real

## Overview
The Stateless Parallel Volume Rendering Communication Protocol is a distributed system architecture for real-time photorealistic volume rendering that decouples computation from state management using a "best-effort" computing paradigm [S1, unofficial]. It solves the synchronization and bandwidth bottlenecks of traditional distributed rendering by employing a stateless model where processing nodes hold full volume copies and synchronize solely via multicast control packets, while output is streamed via unicast to integration nodes [S1, unofficial]. Unlike connection-oriented protocols or stateful cluster rendering, this method prioritizes low-latency LAN performance and fault tolerance over strict delivery guarantees, making it suitable for iterative or fault-tolerant rendering algorithms [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Input variables (state variables) from an input device, captured with timestamps; network topology information (node roles).
- **Outputs**: A distributed multimedia stream of rendered frames, segmented into Compute Segments (CS), visualized on a multi-screen system [S1, unofficial].

### Data Structures
The protocol defines two primary datagram structures: Control Information and Output Information.

**1. Control Datagram Structure**
Used for communication from the Administrator to Processors.
- **IPv6 Header + Options**: Standard IPv6 header. Destination address is the multicast group of all Processing Nodes [S1, unofficial].
- **UDP Header**: Standard UDP header [S1, unofficial].
- **CCM Header (Control Core Message)**:
    - `INPUT VAR TYPE`: Descriptor of the input variables [S1, unofficial].
    - `TIME STAMP`: Timestamp of the input variable capture [S1, unofficial].
    - `CONTROL COUNTER`: Number of control data packets sent [S1, unofficial].
    - `NO VAR COUNTER`: Number of control data packets sent with identical input variables (resets when variables change) [S1, unofficial].
    - `NEXT HEADER`: Indicates the presence of optional headers [S1, unofficial].
- **CCGo Header (Optional, Control Core Go)**:
    - `NODE TYPE`: Indicates if the target is a Processor or Integrator [S1, unofficial].
    - `NODE COUNTER`: Quantity of nodes [S1, unofficial].
    - `NODE INFO`: Type and quantity of node information [S1, unofficial].
    - `NEXT HEADER` [S1, unofficial].
- **CCEo Headers (Optional, Control Core End per Node)**: One per node.
    - `ID NODE`: 64-bit least significant bits of the IPv6 address [S1, unofficial].
    - `SEGMENT INFO`: Bit map of Compute Segments (CS) assigned to the Integrator or to be processed by the Processor [S1, unofficial].
    - `NEXT HEADER` [S1, unofficial].

**2. Output Datagram Structure**
Used for communication from Processors to Integrators.
- **IPv6 Header + Options**: Standard IPv6 header [S1, unofficial].
- **UDP Header**: Standard UDP header [S1, unofficial].
- **CIGS Header (Control Information General System)**: General system variable descriptions. Repeated in every CS for fault tolerance [S1, unofficial].
- **CIGC Header (Control Information General Frame)**: General frame information. Usually constant across all CS of the same frame [S1, unofficial].
- **CES Header (Control Information Element Specific)**: CS-specific information [S1, unofficial].
- **CS Data**: The actual rendered pixel data for the Compute Segment [S1, unofficial].

### Compute Segment (CS) Definition
The CS is the minimum unit of transmission and processing, representing a portion of a frame at a specific time [S1, unofficial].
- **Size**: Configurable. For GPU-based Processors, dimensions must be multiples of OpenCL work-group sizes to maximize performance [S1, unofficial].
- **Constraints**: All frames in the output stream must have the same resolution. All CS of the same frame must have the same size [S1, unofficial].
- **Assignment**: A single CS cannot be received by more than one Integrator to ensure non-overlapping segments [S1, unofficial].

### Node Roles
1.  **Administrator Node**: Manages input variables, transmits them to Processors via multicast, and assigns Integrator segments to Processors via control data [S1, unofficial].
2.  **Processor Node**: Performs rendering. Holds a full copy of the volume in local GPU memory. Self-assigns tasks if not explicitly assigned by the Administrator. Sends finished work to Integrators via unicast [S1, unofficial].
3.  **Integrator Node**: Receives processed tasks, integrates them, and visualizes the scene. Multiple Integrators can exist, each assigned specific non-overlapping image segments [S1, unofficial].

### Execution Steps
1.  **Announcement**: Processing nodes announce themselves to the Administrator Node [S1, unofficial].
2.  **Control Distribution**: The Administrator Node sends control data (input variables with timestamps) to the multicast group of all Processing Nodes. The transmission interval must be less than the refresh time of the visualizer system to ensure Processors always have work [S1, unofficial].
3.  **State Selection**: Each Processing Node receives control information, selects the most recent packet (based on counters), and discards others [S1, unofficial].
4.  **Rendering**: The Processing Node performs rendering using the selected state variables. It generates output traffic composed of groups of CS corresponding to each rendered frame [S1, unofficial].
5.  **Output Transmission**: Processing Nodes send output via unicast to the respective Integration Nodes as fast as processing or network bandwidth allows [S1, unofficial].
6.  **Visualization**: Integration Nodes gather the distributed multimedia transmission stream and visualize it on a multi-screen system [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Administrator Node Logic
def administrator_loop():
    # Receive input variables from device
    input_vars = get_input_device_data()
    timestamp = get_current_time()
    
    # Increment counters
    control_counter += 1
    if input_vars == last_input_vars:
        no_var_counter += 1
    else:
        no_var_counter = 0
        modification_counter += 1
        last_input_vars = input_vars

    # Construct Control Datagram
    ccm_header = {
        "INPUT_VAR_TYPE": descriptor(input_vars),
        "TIME_STAMP": timestamp,
        "CONTROL_COUNTER": control_counter,
        "NO_VAR_COUNTER": no_var_counter,
        "NEXT_HEADER": "CCGo"
    }
    
    # Assign segments to Integrators
    # Ensure no CS is assigned to more than one Integrator
    integrator_assignments = assign_segments_to_integrators()
    
    cogo_header = {
        "NODE_TYPE": "INTEGRATOR",
        "NODE_COUNTER": len(integrator_assignments),
        "NODE_INFO": integrator_assignments,
        "NEXT_HEADER": "CCEo"
    }
    
    # Send multicast packet
    send_multicast(
        dest_group=PROCESSOR_MULTICAST_GROUP,
        payload=build_control_datagram(ccm_header, cogo_header)
    )

# Processor Node Logic
def processor_loop():
    # Hold full volume in GPU memory
    volume = load_volume_to_gpu()
    
    while True:
        # Receive control packet
        packet = receive_multicast_packet()
        
        # Select most recent packet based on counters
        if is_newer(packet, current_state):
            current_state = packet
            input_vars = packet.ccm.INPUT_VAR_TYPE
            timestamp = packet.ccm.TIME_STAMP
            
            # Determine CS to process
            # If Administrator assigned, use assignment. Else, self-assign.
            assigned_cs = get_assigned_cs(packet)
            
            # Render CS
            rendered_cs = render_volume(volume, input_vars, timestamp, assigned_cs)
            
            # Send output via unicast to assigned Integrator
            integrator_id = get_integrator_for_cs(assigned_cs)
            send_unicast(
                dest=integrator_id,
                payload=build_output_datagram(rendered_cs)
            )

# Integrator Node Logic
def integrator_loop():
    while True:
        # Receive CS from various Processors
        cs = receive_unicast_packet()
        
        # Buffer and integrate CS
        buffer.add(cs)
        
        # Visualize when frame is complete
        if is_frame_complete(buffer):
            display_frame(buffer.compose())
```

### Parameters and Configuration
- **CS Size**: Must be configurable. For GPU-based Processors, dimensions should be multiples of OpenCL work-group sizes [S1, unofficial].
- **Control Packet Interval**: Must be less than the refresh time of the visualizer system [S1, unofficial].
- **Volume Storage**: Each Processing Node holds a complete copy of the volume in video memory. Sending volumes over the network is undesirable due to bandwidth constraints [S1, unofficial].
- **Flow Control**: If Ethernet flow control is enabled, Processing Nodes only process what they can send over the network, reducing energy consumption [S1, unofficial].

### Edge Cases and Pitfalls
- **CS Assignment Constraint**: A single CS cannot be received by more than one Integrator. Violating this causes overlapping segments in visualization [S1, unofficial].
- **Network Saturation**: The protocol is tolerant to link saturation. Loss of information does not affect the system because each datagram includes all necessary state information [S1, unofficial].
- **Node Failure**: The protocol tolerates the failure of a processing node and continues working, leveraging the stateless nature and redundant control info [S1, unofficial].
- **State Stability**: The system operates in two states: Stable State (variables unchanged for a long time, all nodes share same state) and Unstable State (variables change rapidly, different state variables assigned to each node) [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Rendering time depends on the computational complexity of the volume rendering algorithm and the size of the CS. Network latency is minimized by using multicast for control and unicast for output on a LAN [S1, unofficial].
- **Memory Complexity**: High memory usage on Processor nodes, as each holds a full copy of the volume in GPU memory. This avoids network bandwidth consumption for volume transfer [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with the number of Processor and Integrator nodes. Sort First (2D) and Alternate Frame Rendering (AFR) techniques are supported. Sort Last is less suitable due to the system's prioritization of low-latency LAN for output [S1, unofficial].
- **Strengths**: Fault tolerance, energy efficiency (via flow control), low latency, and compatibility with iterative/fault-tolerant rendering methods [S1, unofficial].
- **Limitations**: Relies on UDP (no delivery guarantees), requires sufficient GPU memory on all nodes, and depends on a low-latency LAN for output [S1, unofficial].
- **When to use**: Real-time photorealistic volume rendering in distributed environments where fault tolerance and low latency are prioritized over strict synchronization [S1, unofficial].
- **When not to use**: When strict delivery guarantees are required, or when GPU memory is insufficient to hold the full volume on each node [S1, unofficial].

## Background
The protocol was proposed in a system for distributed real-time volume rendering, utilizing IPv6, UDP, and Gigabit Ethernet [S1, unofficial]. It is based on the concept of "best-effort" computing, analogous to connectionless IP networks [S1, unofficial]. The system supports Sort First (2D) and Alternate Frame Rendering (AFR) techniques, distinguishing itself from Sort Last (Depth Buffer) rendering which requires composition interaction [S1, unofficial]. The architecture was designed to handle high computational demands by distributing screen segments (CS) across nodes, leveraging the increasing capacity of GPU video memory [S1, unofficial].

## Sources
[S1] Sistema de Comunicación para Renderización Paralela de Volúmenes en Tiempo Real, https://rtyc.utn.edu.ar/index.php/rtyc/article/download/445/302, fetched 2026-10-29 (unofficial)
