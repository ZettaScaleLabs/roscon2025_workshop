# Exercise 3 - Shared Memory

With the default configuration `rmw_zenoh` is using TCP over the loopback interface for inter-processes communication. Enabling shared memory allows to get lower latency for big messages (>512 bytes).

The usage of shared memory with `rmw_zenoh` is fully transparent to your ROS applications:

- No need of a daemon to managed the shared memory. Each session manages its own shared memory segment for publications.
- No need of loaned buffers - when enabled, `rmw_zenoh` serializes big messages directly in shared memory.
- If shared memory cannot be used (allocation fails or remote peer doesn't use shared memory), Zenoh automatically falls back to TCP.
- Zenoh automatically performs garbage collection and defragmentation of shared memory chunks.
- If a shared memory file is not correctly cleaned up (crash), any router or Zenoh session will cleanup the dangling files at next startup.

## Measuring the latency

To measure the benefit of shared memory, we will first measure the latency of the points cloud publication with the default configuration.

1. Run the router:  
   `just router`
2. Run the simulation, using wall time (i.e. system clock) for timestamping:  
   `just rox_simu use_wall_time:=True`
3. Run a Node measuring the points cloud latency:  
   `just cam_latency`  
   You can also measure the images latency running:  
   `just cam_latency image`

## Configuring shared memory

While enabling shared memory for all processes is not mandatory, it is highly beneficial.  
Therefore, the simplest is to edit your 2 configuration files `~/container_data/ROUTER_CONFIG.json5` and `~/container_data/SESSION_CONFIG.json5` to search for the `transport/shared_memory` section and set `enabled: true` within:

```json5
   // ...

    /// Shared memory configuration.
    /// NOTE: shared memory can be used only if zenoh is compiled with "shared-memory" feature, otherwise
    /// settings in this section have no effect.
    shared_memory: {
      /// Whether shared memory is enabled or not.
      /// ...
      enabled: true,

      // ...
    },

   //...
```

After that, execute the previous commands once more and see the difference in latency.

You can confirm shared memory is enabled by listing the files in `/dev/shm` each `.zenoh` file corresponds to a shared memory allocated by a Zenoh process.

## Configuring the SHM size

The SHM size can be configured independently for each session and for the router, via the `transport/shared_memory/transport_optimization/pool_size` setting.

The SHM acts as a queue that holds messages published by a session and not yet consumed by subscribers. Its size must be large enough to accommodate all in-flight messages at any given time.

A best practice is to configure it to at least twice the total size of the messages published by the session — this ensures a new message can be published even if the previous one has not yet been consumed by all subscribers. For the router, set it to twice the total size of the messages received from external sources and intended to be routed to the SHM.

Example:

- A Node is publishing HD video as 6.22 MB raw image frames. Its SHM size should be at least 12.44 MB (2 * 6.22 MB): this reserves space for one published image not yet consumed by subscribers, plus one new image being published concurrently.

- A router is receiving 2 HD streams (6.22 MB per frame) and a 8 MB points cloud topic. Its SHM size should be at least 20.44 MB (2 * (6.22 + 6.22 + 8))

Make sure that the host's shared memory space (/dev/shm on Linux) is large enough for all the processes you run to allocate the configured amount of memory. As rmw_zenoh is pre-commiting the memory on startup, a process will fail if the shared memory is not available.

> [!Note]
>
> ***Is it possible to use shared memory between containers ?***
>
> *Yes! But first let's see how to establish TCP communication between containers and hosts in the next exercise...*

---
[Solution](solutions/ex-3/) 💡

[Next exercise ➡️](ex-4.md)
