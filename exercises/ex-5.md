# Exercise 5 - Securing communication with mTLS

Mutual Transport Layer Security (mTLS) strengthens robot communication by requiring both the client and server to authenticate each other using certificates, preventing unauthorized access and eavesdropping. However, in local area networks (LANs) where domain verification is often impractical—such as with self-signed certificates or private IP addresses—configuration can be challenging.

Here we present a simple solution, using [step](https://smallstep.com/docs/step-cli/basic-crypto-operations/) to generate:

* a root CA (Certificate Authority), for creation of privates keys and signed certificates.
* a private key and certificate for the robot's router
* a private key and certificate for RViz

Since both the robot and the control are using certificates signed by the same CA, they will be able to authenticate each other and to establish a secured TLS connection.

> [!Warning]
>
> Such solution with self-signed certificates and without domain verification provides encryption and basic authentication, but it does not protect against sophisticated attacks like certificate spoofing or man-in-the-middle (MITM) attacks. Without a trusted Certificate Authority (CA) or proper domain validation, the system remains vulnerable to impersonation if an attacker gains access to your private key or certificate files. This approach is suitable for isolated or trusted LAN environments, but it is not recommended for exposed or public networks where stronger identity verification is required.
>
> For a simple and secured deployment over the Internet, you can rely on the [Zetta Platform](https://www.zettascale.cloud/).

## Certificates and key generation

Run those commands on your host:

```bash
mkdir -p tls

# Create root CA
docker run -t --rm -v $PWD/tls:/home/step smallstep/step-cli \
  step certificate create --profile root-ca "Example Root CA" root_ca.crt root_ca.key \
  --no-password --insecure

# Create key and certificate for the robot (1 year validity)
docker run -t --rm -v $PWD/tls:/home/step smallstep/step-cli \
  step certificate create robot.local robot.crt robot.key \
  --ca ./root_ca.crt --ca-key ./root_ca.key --no-password --insecure  --not-after=8760h

# Create key and certificate for RVIz (1 year validity)
docker run -t --rm -v $PWD/tls:/home/step smallstep/step-cli \
  step certificate create control.local control.crt control.key \
  --ca ./root_ca.crt --ca-key ./root_ca.key --no-password --insecure  --not-after=8760h

# Copy root CA certificate and robot key and certificate to the robot container
cp ./tls/root_ca.crt ./tls/robot.* ./container_volumes/robot_container/
# Copy root CA certificate and RViz key and certificate to the control container
cp ./tls/root_ca.crt ./tls/control.* ./container_volumes/control_container/
```

## Robot configuration

In robot container's `~/container_data/ROUTER_CONFIG.json5` file:

1. Change the `listen/endpoints` list as such:

    ```json5
    // ...
    listen: {
      // ...
      endpoints: [
        // On lookback: listen on plain TCP for internal Nodes to connect
        "tcp/localhost:7447",
        // On external IP: listen on TCP with TLS for remote connections
        "tls/172.1.0.2:7447"
      ],
      // ...
    },
    // ...
    ```

2. Edit the `transport/link/tls` section as such:

    ```json5
      tls: {
        /// Path to the certificate of the certificate authority ...
        root_ca_certificate: "/home/ubuntu/container_data/root_ca.crt",
        /// Path to the TLS listening side private key
        listen_private_key: "/home/ubuntu/container_data/robot.key",
        /// Path to the TLS listening side public certificate
        listen_certificate: "/home/ubuntu/container_data/robot.crt",
        ///  Enables mTLS (mutual authentication), client authentication
        enable_mtls: true,
        /// Path to the TLS connecting side private key
        connect_private_key: "/home/ubuntu/container_data/robot.key",
        /// Path to the TLS connecting side certificate
        connect_certificate: "/home/ubuntu/container_data/robot.crt",
        // Whether or not to verify the matching between hostname/dns and certificate when connecting ...
        verify_name_on_connect: false,
        // ...
      },
    ```

## RViz configuration

In control container's `~/container_data/SESSION_CONFIG.json5` file:

1. Change the `connect/endpoints` list as such:

    ```json5
    // ...
    connect: {
      // ...
      endpoints: [
        // Connect to the robot's router via TCP with TLS
        "tls/172.1.0.2:7447"
      ],
      // ...
    },
    // ...
    ```

2. Edit the `transport/link/tls` section as such:

    ```json5
      tls: {
        /// Path to the certificate of the certificate authority ...
        root_ca_certificate: "/home/ubuntu/container_data/root_ca.crt",
        /// Path to the TLS listening side private key
        listen_private_key: "/home/ubuntu/container_data/control.key",
        /// Path to the TLS listening side public certificate
        listen_certificate: "/home/ubuntu/container_data/control.crt",
        ///  Enables mTLS (mutual authentication), client authentication
        enable_mtls: true,
        /// Path to the TLS connecting side private key
        connect_private_key: "/home/ubuntu/container_data/control.key",
        /// Path to the TLS connecting side certificate
        connect_certificate: "/home/ubuntu/container_data/control.crt",
        // Whether or not to verify the matching between hostname/dns and certificate when connecting ...
        verify_name_on_connect: false,
        // ...
      },
    ```

## Run all

* In the robot container, run:

  * `just router`
  * `just rox_simu`
  * `just rox_nav2`

* In the control container, run:

  * `just rviz_nav2`

## QUIC

QUIC is a modern transport protocol built on UDP that provides built-in encryption, lower connection latency, resilience to packet loss, and multiple streams to avoid head-of-line blocking. Since QUIC uses mTLS natively, the same certificates and `transport/link/tls` configuration generated above apply without any change — simply replace `tls` with `quic` in the endpoint URLs.

In robot container's `~/container_data/ROUTER_CONFIG.json5` file, change the `listen/endpoints` list as such:

```json5
// ...
listen: {
  // ...
  endpoints: [
    // On loopback: listen on plain TCP for internal Nodes to connect
    "tcp/localhost:7447",
    // On external IP: listen on QUIC for remote connections
    "quic/172.1.0.2:7447"
  ],
  // ...
},
// ...
```

In control container's `~/container_data/SESSION_CONFIG.json5` file, change the `connect/endpoints` list as such:

```json5
// ...
connect: {
  // ...
  endpoints: [
    // Connect to the robot's router via QUIC
    "quic/172.1.0.2:7447"
  ],
  // ...
},
// ...
```

Then run all as before:

* In the robot container, run:

  * `just router`
  * `just rox_simu`
  * `just rox_nav2`

* In the control container, run:

  * `just rviz_nav2`

> [!NOTE]
> When you face a connection issue with the certificate and key, you can check the file permission of the certificate and key. This might happen if your current user id is not aligned with the one inside containers.

---
[Solution](solutions/ex-5/) 💡

[Next exercise ➡️](ex-6.md)
