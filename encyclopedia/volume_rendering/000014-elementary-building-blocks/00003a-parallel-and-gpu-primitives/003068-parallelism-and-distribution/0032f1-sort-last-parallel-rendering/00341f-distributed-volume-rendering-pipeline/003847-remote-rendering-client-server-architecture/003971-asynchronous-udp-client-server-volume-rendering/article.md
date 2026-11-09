# Asynchronous UDP Client-Server Volume Rendering

Also called: Asynchronous UDP Volume Rendering, UDP-based Distributed Volume Rendering

## Overview
This method is a distributed rendering architecture where volume rendering computation is offloaded to a separate server PC, with rendered images transferred to a client via UDP/IP [S1, unofficial]. It solves the problem of interaction latency in virtual environments by decoupling user navigation from rendering, allowing the client to handle input smoothly while the server updates the view asynchronously [S1, unofficial]. It distinguishes itself from synchronous request/acknowledge cycles by ignoring intermediate update requests during active rendering, processing only the youngest message upon completion, which yields lower latency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Visualization parameters (viewpoint, transparency) from the client; volume data on the server.
- Output: Rendered image pairs (stereo) sent to the client.

**Data Structures:**
- Message: Contains visualization parameters. Format is not specified in the sources [G].
- Image: Rendered image pair. Format is not specified in the sources [G].

**Steps:**
1. The client detects a change in visualization parameters (e.g., viewpoint or transparency).
2. The client sends an update-request message to the server via UDP/IP [S1, unofficial].
3. The server receives the message. If the server is currently rendering, it ignores the incoming packet [S1, unofficial].
4. If the server is idle, it processes the youngest received message and initiates volume rendering [S1, unofficial].
5. The server renders the image pair.
6. Upon completion, the server sends the image pair to the client via UDP/IP [S1, unofficial].
7. The server becomes idle and waits for the next message [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Server Side
class Server:
    def __init__(self):
        self.pending_message = None
        self.is_rendering = False

    def receive_message(self, message):
        # Store the youngest message; ignore if currently rendering
        if not self.is_rendering:
            self.pending_message = message
        # If rendering, the message is effectively ignored per spec
        pass

    def render_loop(self):
        while True:
            if self.pending_message is not None:
                self.is_rendering = True
                # Render volume using parameters from pending_message
                image_pair = render_volume(self.pending_message)
                self.pending_message = None
                self.is_rendering = False
                send_udp(image_pair)
            else:
                # Wait for next message
                pass

# Client Side
class Client:
    def on_parameter_change(self, params):
        send_udp(params)
```

**Parameters:**
- Protocol: UDP/IP [S1, unofficial].
- Reliability: Unreliable (packets may be lost), but idempotent updates are guaranteed by continuous transmission [S1, unofficial].

**Edge Cases and Pitfalls:**
- Packet Loss: Since UDP is unreliable, lost packets result in stale views. However, continuous transmission ensures eventual consistency [S1, unofficial].
- Network Congestion: High throughput is required; UDP minimizes overhead but does not handle congestion control [S1, unofficial].

## Subtopics

## Operating characteristics
- **Latency:** Lower than complete request/acknowledge cycles because the server does not wait for acknowledgments and processes only the youngest message [S1, unofficial].
- **Throughput:** High, due to UDP's unbuffered, message-oriented communication minimizing protocol overhead [S1, unofficial].
- **Scalability:** Supports multi-user environments by distributing client and server processes on different machines without performance reduction [S1, unofficial].
- **Reliability:** Lower than TCP; suitable for LANs where packet loss is rare [S1, unofficial].

## Background
Originated in the context of distributed software-based volume visualization in virtual environments, specifically within the "Studierstube" and "RTVR" systems [S1, unofficial]. It was developed to provide flexible coupling between Java and C++ components and to ensure that rendering delays do not impair VR interaction [S1, unofficial]. The approach builds on previous experience in distributed visualization [FLS97] but distinguishes itself by using a dedicated server for volume rendering [S1, unofficial].

## Sources
[S1] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
