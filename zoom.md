## Design a system Like ZOOM

Inspired from
https://www.codekarle.com/system-design/Zoom-system-design.html


### Functional Requirements
* 1-to-1 and group video/audio calls. 
* Screen sharing, text chat, and real-time reactions.  
* Cloud recording, live transcriptions. (out of scope here)

### Non-Functional Requirements
* Ultra-Low Latency: Latency must be under 200ms to maintain natural human conversation.  
* High Availability: The system must allow users to join instantly without downtime.
* **Audio must be prioritized over video**. A frozen video with clear audio is tolerable; clear video with choppy audio is not. 
* Needs to seamlessly support thousands of concurrent meetings, with up to 1,000 participants in a single meeting.

## Media Routing Architecture
The biggest trap in this design is assuming users connect peer-to-peer (P2P). 

While P2P works for two people, a group call with 10 people requires each client to upload 9 video streams and download 9 video streams. This quickly crushes client bandwidth and CPU.

Instead, we use a server-assisted model. There are two primary server types:  
1. **MCU (Multipoint Control Unit):** The server receives all streams, mixes them into a single video/audio feed, and sends it back to everyone. 
    * Drawback: Massive server CPU overhead due to real-time rendering/transcoding.
2. **SFU (Selective Forwarding Unit):** The client uploads one stream to the server. The SFU then forwards that stream to all other participants without modifying it. 

For a Zoom-like architecture, we choose the SFU model. It shifts the decoding/rendering workload to the clients, allowing the backend media servers to scale easily and handle thousands of streams concurrently.

## High-level Architecture
The system is fundamentally decoupled into two distinct planes: 
1. the Signaling Plane (which uses TCP/WebSockets for reliability) and 
2. the Media Plane (which uses UDP for raw speed).

```
[ Client 1 ] <--- WebSocket (Signaling) ---> [ Signaling Service ] <---> [ DB / Cache ]
     |
     +--------- UDP / WebRTC (Media) --------> [ SFU Media Server ]
                                                     |
[ Client 2 ] <--------- UDP / WebRTC (Media) --------+
```

1. **Web Servers & API Gateway:** Handles REST operations like user authentication, profile management, and scheduling meetings.  
2. **Signaling Service:** A persistent connection layer (via WebSockets) that coordinates meeting states. When you click "Join," it passes session metadata (codecs, encryption keys, network info) between you and the media server. 
3. **SFU Media Server Cluster:** Distributed globally in regional data centers to receive and route raw video/audio packets. 
4. **Live Recording Service:** An asynchronous consumer. It connects to the SFU like a silent participant, copies the stream, stitches it, and pushes the finished file to S3/Object Storage.

### Handling Network & Scale Challenges
1. All media traffic is routed over UDP (via WebRTC/RTP). We intentionally choose a lossy protocol. If a video packet drops, we don't want the network to freeze waiting for a retransmission (like TCP would). We just skip it and render the next incoming frame.  
2. If UserA has a fiber internet but another UserB is on a spotty mobile network, sending the same 1080p video stream to both breaks the mobile user's experience. We can solve this in two ways:
    * **Simulcast:** The sender client encodes its video into three different resolutions (e.g., 1080p, 480p, 240p) and uploads all three to the SFU. The SFU dynamically routes the 1080p version to the fiber user and the 240p version to the mobile user.
    * **SVC (Scalable Video Coding):** The client sends a single multi-layered stream (base layer + enhancement layers). The SFU strips away enhancement layers for users with poor connections.
3. Meetings should be bound to regional media servers. If five people in London are having a meeting, their media streams shouldn't route through a data center in New York. The API Gateway uses GeoDNS to assign users to the closest SFU broker, drastically minimizing round-trip times.


### Data and storage Chouces
1. **User/Meeting Metadata:** Relational DB (PostgreSQL) or NoSQL for flexible meeting structures, heavily cached by Redis since read-to-write ratios for room configurations are high.
2. **Chat Messages:** A NoSQL database optimized for heavy writes and low-latency retrieval (Cassandra or DynamoDB), decoupled completely from the media servers.
3. **Recorded Videos:** Cloud Object Storage (AWS S3) paired with a CDN for user downloads.
