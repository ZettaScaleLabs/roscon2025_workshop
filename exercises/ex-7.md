# Exercise 7 - Cope with congestion and head of line blocking

In wireless networks, bandwidth is scarce. And when a large message (such as a point cloud or image) occupies the channel, it blocks all smaller, time-critical messages behind it. This is known as **head-of-line blocking**: the head of the queue monopolizes the link, and every other message must wait, regardless of its urgency.

Let's simulate a medium quality WiFi (2.4 GHz) connection between the two containers:

* In robot container, limit the network traffic with the control container running:  
   `just network_limit`  
* You can check it's effective with a ping to the control container:  
   `ping 172.1.0.3`  
  The latency should be above 20 ms

If you want to restore optimal communication between the containers, run:
  `just network_normal`.

If you want to simulate different network conditions, edit `network_limit.sh` and run:
  `just network_normal; just network_limit` to apply the new settings.

Now run again the simulation and Rviz:

1. In the robot container, run:

   * `just router`
   * `just rox_simu`
   * `just rox_nav2`

2. In the control container, run:  
   `just rviz_nav2`

If you come from exercise 6 where you configured Compression, Access Control and Downsampling, it might still work well. However, see what happens if you comment out the Access Control and Downsampling settings in the router configuration.

## Priorities

Configuring priority overwrites on the router for outgoing (`egress`) traffic ensures that time-critical messages are scheduled before large, lower-priority payloads.

Zenoh supports 8 priority levels, from highest to lowest:

* `"control"`
* `"real_time"`
* `"interactive_high"`
* `"interactive_low"`
* `"data_high"`
* `"data"` *(default)*
* `"data_low"`
* `"background"`

When combined with QUIC (see [Exercise 5](ex-5.md)), this becomes even more powerful: QUIC natively supports multiple independent streams, so head-of-line blocking between different priority levels is eliminated at the transport layer. Zenoh maps each of its priority levels to a dedicated QUIC stream, meaning a large image on the `background` stream cannot delay a navigation command on the `interactive_low` stream.

In robot container's `~/container_data/ROUTER_CONFIG.json5` file add this section:

```json5
qos: {
  network: [
    {
      // Set /map and /scan to highest priority as required
      interfaces: ["eth0"],
      key_exprs: ["**/map/**", "**/scan/**"],
      messages: ["put", "query"],  // also "query" as "/map" is TRANSIENT_LOCAL
      overwrite: { 
        priority: "interactive_high"
      }
    },
    {
      // Set /robot_description to high priority as important
      interfaces: ["eth0"],
      key_exprs: ["**/robot_description/**"],
      messages: ["put", "query"],  // also "query" as "/robot_description" is TRANSIENT_LOCAL
      overwrite: { 
        priority: "interactive_low"
      }
    },
    {
      // Set /camera/image_raw to lower priority as quite heavy and not required
      interfaces: ["eth0"],
      key_exprs: ["**/camera/image_raw/**"],
      messages: ["put"],
      overwrite: {
        priority: "data_low",
      }
    },
    {
      // Set /camera/points to lowest priority as very heavy and not required
      interfaces: ["eth0"],
      key_exprs: ["**/camera/points/**"],
      messages: ["put"],
      overwrite: {
        priority: "background",
      }
    },
  ],
},
```

Stop Rviz and the router, than restart them.


## Congestion Control

For all topics except those with TRANSIENT_LOCAL + KEEP_ALL QoS, `rmw_zenoh` is configured with congestion control `drop`. Meaning that in case of congestion, Zenoh will drop the messages it can't push to the network after a timeout (see `transport/link/tx/queue/congestion_control/drop` config). On a congested network, the push of large messages will probably always exceed the timeout and hence always be dropped.  

What you likely want in such case is to at least have one large message from time to time, but not impacting the latency of small messages. The solution for this if to make the router to change on-the-fly the `congestion_control` for `block_first`.
With this setting Zenoh will block on the first message to be pushed to the network and drop the other ones until this first message it sent. Thus at lease some large messages manage to be transmitted.

In robot container's `~/container_data/ROUTER_CONFIG.json5` file, within the `qos/network` section where you configured priorities, add this element:

```json5
    {
      // For any large message (>4KB) set congestion_control to block_first
      interfaces: ["eth0"],
      payload_size: "4096..",
      messages: ["put"],
      overwrite: {
          congestion_control: "block_first",
      }
    },
```

Stop Rviz and the router, than restart them. You should see some video frames and points cloud in RViz now.

## Conclusion

Working over a constrained wireless link requires a layered approach:

1. **First, minimize outgoing traffic** (Exercise 6): use Access Control to block topics that don't need to cross the network, Downsampling to reduce the rate of heavy topics, and Compression to reduce the size of the data transmitted. This is the most impactful step — no amount of QoS tuning compensates for sending unnecessary data.

2. **Then, configure priorities** — ideally with QUIC, which maps each Zenoh priority to an independent stream, giving true isolation between critical and non-critical traffic at the transport layer.

3. **For large payloads**, additional tuning is needed: switching `congestion_control` to `block_first` ensures at least some frames get through even under heavy congestion, rather than being silently dropped.

---
[Solution](solutions/ex-7/) 💡

[Next exercise ➡️](ex-8.md)
