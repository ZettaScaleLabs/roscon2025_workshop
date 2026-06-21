# Exercise 6 - Tuning for wireless networks

As robots often depend on wireless networks like WiFi and 5G for remote control, supervision, and data management, engineers must address key challenges: handling unpredictable network traffic, optimizing limited bandwidth, and mitigating TCP congestion—all of which can degrade performance, introduce latency, or disrupt critical operations.

At first, for comparison purpose, measure the Zenoh traffic between RViz and the robot using the same setup as the previous exercise (without shared memory between containers!):

- In the robot container, run:

  - `just router`
  - `just rox_simu`
  - `just rox_nav2`

- In the control container, run:

  - `just rviz_nav2`
  - `just iftop_router`  
     to measure the traffic with the robot's router.

## Enable compression on the router and RViz

Zenoh supports transparent compression of payloads, using [LZ4](https://en.wikipedia.org/wiki/LZ4_(compression_algorithm)) algorithm for faster compression and decompression and less impact on latency.

To be effective, compression has to be enabled on the both end of a Zenoh link. In our case we want to enable the compression between the robot's router and RViz running in the controler host. As the robot's nodes won't have compression enabled, the peer-to-peer communication between nodes and with the router won't be compressed.

1. In robot container's `~/container_data/ROUTER_CONFIG.json5` file, search for the `transport/unicast/compression/enabled` setting and set it to `true`.
2. In the control container's `~/container_data/SESSION_CONFIG.json5` file, search for the `transport/unicast/compression/enabled` setting and set it to `true`.
3. Restart the robot's router
4. Restart Rviz

See the impact on the traffic on `iftop_router`.

## Blocking topics and services via Access Control

To optimize bandwidth and security, it’s usually a good practice to restrict network exposure of topics and services. Zenoh’s Access Control feature enables you to confine sensitive data and functionality to the robot itself.  
Let’s test this by blocking the `/camera/points` topic (Realsense points cloud) for the robot’s external network interface.

1. In robot container's `~/container_data/ROUTER_CONFIG.json5` file add this section:

   ```json5
   access_control: {
     enabled: true,
     default_permission: "allow",
     rules:
     [
       {
         id: "deny_points_cloud",
         // deny any kind of messages...
         permission: "deny",
         messages: [
           "put", "delete", "declare_subscriber",
           "query", "reply", "declare_queryable",
           "liveliness_token", "liveliness_query", "declare_liveliness_subscriber",
         ],
         // ... incoming or outgoing ...
         flows:["egress", "ingress"],
         // ... on any key expressions containing "/camera/points/",
         // including the specific ones with "/@adv" used for TRANSIENT_LOCAL topics
         key_exprs: [
           "*/camera/points/**",
           "*/camera/points/**/@adv/**"
         ],
       },
     ],
     subjects:
     [
       {
         // An empty subject means that the rule will apply to ALL peers/clients/routers
         id: "ALL",
       },
     ],
     policies:
     [
       {
         // Apply the rule to all
         id: "deny_points_cloud_to_all",
         rules: ["deny_points_cloud"],
         subjects: ["ALL"],
       },
     ]
   },
   ```

2. Restart the robot's router

See the impact on the traffic and check that the points cloud are no longer received by RViz.

> [!IMPORTANT]
>
> Denying the `camera/points` topic on the router means it won't route any message for this topic. Still the nodes inside the robot can use this topic in peer-to-peer communications.

## Downsampling

For further traffic reduction, you can downsample data by lowering the frequency of publications routed to the remote hosts. This approach helps minimize network load while preserving essential information for remote control and supervision.

You can add the following configuration to `ROUTER_CONFIG.json5` for the robot container, which make the frequency of the camera image (The topic is `camera/image_raw`) drop to 3 Hz. After restarting the Zenoh Router, you can see the camera image on the Rviz is obviously lagging.

1. In robot container's `~/container_data/ROUTER_CONFIG.json5` file add this section:

    ```json5
    downsampling: [
      {
        // Downsampling "put" (publications) and "reply" (in case of TRANSIENT_LOCAL topic) ...
        messages: ["put", "reply"],
        // ... outgoing ...
        flows: ["egress"],
        rules: [
          // 3 Hz for any key expressions containing "/camera/image_raw/"
          { key_expr: "*/camera/image_raw/**", freq: 3.0 },
          // For TRANSIENT_LOCAL topics, the Subscriber sends a query to retrieve
          // missed publications. Those publications are sent as individual replies
          // in a burst. You might want to downsample the replies, or not...
          { key_expr: "*/camera/image_raw/**/@adv/**", freq: 3.0 },
        ],
      },
    ],
    ```

2. Restart the robot's router

See the impact on the traffic and check the camera refresh rate in RViz.

---
[Solution](solutions/ex-6/) 💡

[Next exercise ➡️](ex-7.md)
