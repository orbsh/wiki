# Long-Lived Connection Engineering: Physical Limits and Solutions in a Storage-Compute-Integrated Architecture

> **Languages:** [English](long-lived-connection-engineering-en.md) (primary) · [中文](long-lived-connection-engineering.md)

**Status**: Draft
**Date**: 2026-07-29

## Introduction

WebSocket / WebTransport long-lived connections are the core channel of the [Flux Architecture](flux-architecture-en.md), and also the physical carrier of the booth in the storage-compute-integrated architecture of [Aura Architecture](aura-architecture-en.md). Short-lived HTTP requests avoid the complexity of connection state management, but once long-lived connections bind the connection lifecycle to a compute unit, they introduce a series of physical-level engineering challenges. This article surveys the essence of these challenges and modern solutions.

## 1. Backpressure

### Problem

When the output rate of the producer (stream compute engine / booth event processing) exceeds the consumption rate of the consumer (client connection), data accumulates at the gateway layer.

The backpressure model of traditional reactive frameworks assumes that the logical channel is a one-way pipe—when the consumer slows down, the backpressure signal propagates upstream in reverse, slowing down the sender. But in the actual network stack, data passes through multiple layers of buffering:

```
Application-Layer Queue → Tokio Channel → Tokio Write Buffer → Kernel TCP Send Buffer → Network
```

If the gateway only senses the full/empty state of the application-layer queue and ignores the accumulation in the kernel-layer Send Buffer, this leads to two consequences:
- **Memory bloat**: the application layer keeps writing to the Channel, the Channel grows unbounded, and eventually OOM occurs
- **Latency spikes**: when the Send Buffer accumulates until the TCP window shrinks, the latency perceived by the client suddenly jumps up

### Solution

Network-Aware Backpressure:

- **Rust/Tokio path**: when `tokio::io::AsyncWrite::poll_write` returns `Poll::Pending`, the Tokio runtime automatically pauses the write task of that connection, and the upstream Channel naturally applies backpressure. The key is to ensure the Channel is bounded (`tokio::sync::mpsc::bounded`) rather than unbounded.
- **Application-layer degradation**: when the Channel backlog exceeds a threshold, skip low-priority events (such as heartbeats and state synchronization) and keep only business-critical events. This is "lossy backpressure"—discarding non-critical data to protect system stability.
- **WebTransport advantage**: WT is based on QUIC and has built-in flow control. Each Sub-Stream has an independent flow window, and backpressure signals propagate automatically at the transport layer without manual implementation at the application layer. This is a structural advantage of WT over WS.

## 2. Memory Fragmentation

### Problem

The typical lifecycle of a resident long-lived connection: event arrival → state mutation → new data generation → write to local storage (Fjall). This cycle runs continuously under high concurrency, causing two memory problems:

- **Heap fragmentation**: frequent alloc/dealloc (event objects, serialization buffers) causes heap memory fragmentation; RSS keeps growing while the actually available memory is insufficient
- **GC pauses**: languages with GC (Go, Java, Python) trigger a Full GC when fragmentation is severe, causing long pauses

### Solution

- **GC-free language**: Rust's ownership system manages memory lifetimes at compile time, so there are no runtime GC pauses. As a native Rust storage engine, Fjall's memory management is unified with the booth runtime.
- **Slab allocator**: use a Slab allocator for fixed-size objects (such as event messages and connection state), avoiding the fragmentation problems of a general-purpose heap allocator. Tokio's `BytesMut` and `bytes::Bytes` internally use reference counting + zero-copy slices, reducing unnecessary allocations.
- **Off-heap memory**: in extreme scenarios (millions of connections), allocate connection metadata and state data off-heap (`std::alloc::alloc` or `mmap`), bypassing GC and heap fragmentation. The cost is manual lifetime management.

## 3. Connection Skew

### Problem

HTTP requests are stateless, and a load balancer can distribute any request to any node. But WS/WT long-lived connections are stateful—once established, the client is bound to a specific node.

When traffic surges, autoscaling adds capacity, but the new nodes cannot share the existing connections. If a disconnect-and-reconnect strategy is adopted, a large number of clients disconnecting and reconnecting at the same time produces a thundering herd effect, overwhelming the new nodes.

### Solution

- **Connection migration**: WebTransport is based on QUIC and supports Connection Migration—when the client IP changes (such as switching from Wi-Fi to 4G), the connection automatically migrates to a new path without needing to disconnect and reconnect. This alleviates connection skew in network-switching scenarios.
- **Graceful migration layer**: deploy a connection proxy layer (such as Envoy or a custom Rust proxy) in front of the gateway, maintaining a "connection → backend node" mapping table. When scaling out, the proxy routes new connections to new nodes while existing connections remain unchanged. Existing connections naturally decay during idle or off-peak periods.
- **Partitioning strategy**: assign connection nodes by hashing the user ID, ensuring that the connections of the same user are concentrated on one node. When scaling out, only one range on the hash ring is affected, and the migration scope is controllable.

## 4. Multiplexing Topology (WebTransport Sub-Streams)

### Problem

WS is a logical single channel—all messages on one connection share the same channel. The Publisher/Subscriber abstraction of reactive frameworks is based on this assumption.

WebTransport supports multiple independent Sub-Streams, each with independent flow control and ordering guarantees. This introduces a two-layer topology of "one physical connection, multiple logical channels":

```
WT Connection (Physical Layer)
├── Sub-Stream 1: Group A messages
├── Sub-Stream 2: Group B messages
├── Sub-Stream 3: Global broadcast
└── Sub-Stream 4: Control commands
```

The Topic/Channel abstraction of traditional reactive frameworks cannot be mapped directly to this topology—they assume that one Topic corresponds to an independent connection or queue, rather than to a sub-stream within a connection.

### Solution

- **Two-layer routing**: the gateway maintains a "Topic → Sub-Stream ID" mapping table. When a message arrives, it first looks up the mapping table to determine the target Sub-Stream, then sends it to the corresponding sub-stream through the WT API.
- **Fine-grained permission control**: Sub-Stream-level flow control allows independent rate limiting or pausing for a single group without disconnecting the entire connection.
- **Backward compatibility**: WS clients still use the logical single channel, while WT clients can choose multi-stream mode. The gateway layer performs protocol adaptation, transparent to the upstream business layer.

## 5. Memory Topology: Direct Addressing and Reverse Index

Under the storage-compute-integrated architecture, connection instances, state data, and group relationships all live in the memory space of the same process. This eliminates the cross-network queries of the traditional architecture, in which "the gateway is stateless and relationships live in Redis/MySQL", but introduces memory topology design problems.

### Global Control (Ban / Password Change)

On an edge node, a user connection is a resident booth instance that holds the connection handle and state data in memory. The global controller maps directly to that instance via userId:

```
Global Controller → userId Mapping Table → Booth Instance → Connection Handle
```

When a ban occurs, the controller locates the instance by userId, releases the connection (calling the OS-level `close(fd)` to destroy the Socket), and clears the state. There is no middleware dispatch latency.

### Group Interaction: Reverse Index

Group management requires two data structures:

- **Forward index**: the group object holds the member list (`Room.members = [A, B, C]`)
- **Reverse index**: the user object holds the list of groups joined (`A.joined_rooms = [Room_99, Room_42]`)

The physical path of a kick operation:

```
1. The controller finds user A's booth instance by userId
2. Read A.joined_rooms (reverse index)
3. Iterate over each group and remove A from Room.members
4. Release A's connection and destroy the booth instance
```

Broadcast operation:

```
1. The group object iterates over Room.members (forward index)
2. Call push on the connection handle of each surviving member
3. A is no longer in the list, so it is physically impossible for it to receive the message
```

There is no `if(isBanned)` check—the kicked user disappears from the list, and the business logic naturally never touches the deleted instance.

### Consistency Issues

The above model assumes that all operations complete within the same process. But when group members are distributed across multiple nodes:

- Room_99 on node 1 and Room_99 on node 2 are two independent replicas
- The kick operation only modifies node 1's replica; node 2's replica still holds A's connection
- Messages for A on node 2 will still be pushed

This is the classic problem of distributed consistency, and the solution depends on the consistency requirements:

- **Eventual consistency**: broadcast the kick event through a message queue, and each node executes it asynchronously. Within the latency window A may still receive messages, but it will eventually be cleaned up.
- **Strong consistency**: the kick operation releases the connection after confirmation by a consensus service (etcd-like). The cost is one additional RTT per kick.

The data plane of the Aura architecture has no cross-node replication (state follows the owning node); group-type strong-consistency coordination is attached to an external consensus service on demand, or handed to the owning node for single-writer arbitration—no consensus path is built into the engine.

---

## Cross-References

- **[Flux Architecture](flux-architecture-en.md)**: long-lived connections are the core channel of the Flux Architecture; this article discusses the engineering challenges of that channel.
- **[Aura Architecture](aura-architecture-en.md)**: the storage-compute-integrated engine; the memory topology model in this article is based on the in-process architecture of booth + Fjall.
- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: in traditional architectures, connection state and group relationships live in Redis; this article shows how storage-compute integration eliminates this layer.
