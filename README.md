# udptun VPN Usage Guide

## Overview

`udptun` is a simple Python-based VPN solution that creates a tunnel over UDP. It allows you to route IP traffic between a client and a server through a secure (authenticated via shared secret) UDP channel. Each client gets its own virtual TUN interface. The server can also synchronize routing information to clients, potentially enabling P2P communication between clients under certain NAT conditions.

**Note**: This script provides authentication via a shared secret but does not encrypt the tunnelled traffic itself. If you need strong encryption, consider tunnelling protocols like SSH or HTTPS through `udptun`, or use it in conjunction with other VPN solutions like WireGuard or OpenVPN.

## Prerequisites

*   **Python 3.8+**: The script has been updated to run with Python 3.
*   **Root/Administrator Privileges**: Required to:
    *   Create and configure TUN network interfaces.
    *   Modify routing tables.
    *   Bind to privileged ports (if using ports < 1024, though not default).
*   **Operating System**:
    *   **Linux**: Well-supported. Requires the `tunctl` utility or for the `/dev/net/tun` device to be available and kernel support for TUN interfaces.
    *   **macOS**: Supported. TUN interfaces are typically named `tun0`, `tun1`, etc., and are located in `/dev/`.
*   **Shared Secret**: The `SHARED_PASSWORD` variable within `udptun.py` must be identical on both the server and any clients that connect to it. It's highly recommended to change this from the default value.

## Setup

1.  **Get the Code**:
    Clone the repository or download `udptun.py`.
    ```bash
    # Example: git clone <repository_url>
    # cd <repository_directory>
    ```

2.  **Modify Shared Password (Recommended)**:
    Open `udptun.py` in a text editor and change the `SHARED_PASSWORD` variable:
    ```python
    # Near the top of udptun.py
    SHARED_PASSWORD = hashlib.sha1(b"YourStrongSecretPasswordHere").digest()
    ```
    Ensure this is the same on the server and all clients.

3.  **Make Scripts Executable (Optional but Recommended)**:
    If you plan to use `client.sh` or `server.sh`:
    ```bash
    chmod +x client.sh server.sh udptun.py
    ```

## Running the Server

The server listens for incoming client connections, assigns TUN interface IP addresses, and relays traffic.

**Command**:
```bash
sudo python3 udptun.py -s <port> -l <server_tun_ip> [-p <default_client_tun_ip_for_server_config>] [-d]
```
or using the shell script (edit `server.sh` to set your desired parameters):
```bash
sudo ./server.sh
```

**Server Arguments**:

*   `-s <port>`: (Required) The UDP port number the server will listen on (e.g., `1194`).
*   `-l <server_tun_ip>`: (Required) The IP address for the server's side of the TUN interface (e.g., `10.0.0.1`). This will be the gateway for clients.
*   `-p <default_client_tun_ip_for_server_config>`: (Optional) The peer IP address for the server's TUN interface configuration (e.g., `10.0.0.2`). When a client connects, the server effectively sets its `tunX` interface to be `10.0.0.1` with destination `10.0.0.2` (if these IPs are used). The client should then use `10.0.0.2` as its local TUN IP and `10.0.0.1` as its peer.
*   `-d`: Enable debug output for more verbose logging.
*   `-h`: Show the help message.

**Example Server Invocation**:
```bash
# On your VPS (e.g., 192.227.177.171), listening on port 1111
# Server's tunnel IP: 10.0.0.1
# Client's expected tunnel IP (for server's config): 10.0.0.2
sudo python3 udptun.py -s 1111 -l 10.0.0.1 -p 10.0.0.2
```
This starts the server listening on UDP port `1111`. The server's TUN interface will be configured with IP `10.0.0.1`, and it will expect clients to use IPs in a way that `10.0.0.2` would be a typical first client.

The server will print messages like:
```
Server listen at port 1111
Configuring interface tun0 with ip 10.0.0.1
```
When a client connects:
```
[Timestamp] Created new tun tun0, 10.0.0.1 -> 10.0.0.2 for ('client_public_ip', client_port)
```

## Running the Client

The client connects to the `udptun` server, establishes a local TUN interface, and routes traffic through it.

**Command**:
```bash
sudo python3 udptun.py -c <server_host>,<server_port> -l <client_tun_ip> -p <server_tun_ip_as_peer> [-d]
```
or using the shell script (edit `client.sh` to set your desired parameters):
```bash
sudo ./client.sh
```

**Client Arguments**:

*   `-c <server_host>,<server_port>`: (Required) The hostname or IP address of the `udptun` server and its listening port, separated by a comma (e.g., `vpn.example.com,1111`).
*   `-l <client_tun_ip>`: (Required) The IP address for the client's side of the TUN interface (e.g., `10.0.0.2`). This should correspond to what the server expects.
*   `-p <server_tun_ip_as_peer>`: (Required) The IP address of the server's side of the TUN interface, which will be the client's gateway (e.g., `10.0.0.1`). This should match the server's `-l` argument.
*   `-d`: Enable debug output.
*   `-h`: Show the help message.

**Example Client Invocation**:
```bash
# On your local machine (e.g., public IP 79.164.232.46)
# Connecting to server at 192.227.177.171 on port 1111
# Client's tunnel IP: 10.0.0.2
# Server's tunnel IP (peer for client): 10.0.0.1
sudo python3 udptun.py -c 192.227.177.171,1111 -l 10.0.0.2 -p 10.0.0.1
```
This connects the client to `192.227.177.171:1111`. The client's TUN interface will be configured with IP `10.0.0.2`, and its peer (gateway) will be `10.0.0.1`.

The client will print messages like:
```
Configuring interface tun0 with ip 10.0.0.2
[Timestamp] Created client tun0, 10.0.0.2 -> 10.0.0.1 for ('192.227.177.171', 1111)
[Timestamp] Do login ...
Logged in server succefully!
```

## How it Works (Simplified)

1.  **Authentication**: The client initiates a connection by sending an `AUTH` message containing its desired TUN IPs and the `SHARED_PASSWORD` (hashed) to the server.
2.  **TUN Creation & Configuration**:
    *   **Server**: If authentication is successful, the server creates a new TUN interface (e.g., `tun0`), assigns it the specified server-side IP (from its `-l` arg), and sets its peer address (often derived from client's `-l` or server's `-p`).
    *   **Client**: The client also creates a TUN interface, assigns its local IP (from its `-l` arg), and sets its peer to the server's TUN IP (from its `-p` arg).
3.  **Packet Forwarding**:
    *   When the client sends IP packets to its TUN interface's peer IP (the server's TUN IP), the client script reads these packets.
    *   It encapsulates them into UDP datagrams and sends them to the server's public IP and port.
    *   The server receives these UDP datagrams, extracts the original IP packets, and writes them to its corresponding TUN interface. From here, the server's OS routes them as normal.
    *   The process is reversed for traffic from the server to the client.
4.  **Routing Table Synchronization (`RTBL`)**: The server periodically sends its routing table (including connected clients and their virtual IPs) to all connected clients. This allows clients to potentially send packets directly to other clients if their NATs permit (Full-Cone NAT is mentioned), or route through the server.

## Routing

After the tunnel is up, you'll need to configure your system's routing table to send traffic through the TUN interface.

**Example: Route all traffic through the VPN (Client-side)**
If the server's TUN IP is `10.0.0.1` and your client's TUN interface is `tun0`:
```bash
# 1. Add a route for the VPN server's public IP (e.g., 192.227.177.171)
#    to go through your current default gateway.
#    Find your gateway: ip route show default (e.g., 192.168.1.1)
#    sudo ip route add 192.227.177.171 via your_original_gateway_ip dev your_ethernet_interface

# 2. Change the default route to go through the VPN tunnel.
#    (Using 0.0.0.0/1 and 128.0.0.0/1 overrides the default route more safely)
sudo ip route add 0.0.0.0/1 via 10.0.0.1 dev tun0
sudo ip route add 128.0.0.0/1 via 10.0.0.1 dev tun0

# To revert (example, order might matter depending on specifics):
# sudo ip route del 0.0.0.0/1 via 10.0.0.1 dev tun0
# sudo ip route del 128.0.0.0/1 via 10.0.0.1 dev tun0
# sudo ip route del 192.227.177.171 # If you added it specifically
```
**Important**:
*   Replace `192.227.177.171` with your actual server's public IP in the routing commands if needed.
*   Replace `your_original_gateway_ip` and `your_ethernet_interface` with your actual network configuration.
*   The command `sudo ip route add 192.227.177.171 ...` *before* changing the default route is crucial to ensure that the encrypted UDP packets to the VPN server itself don't try to go through the tunnel, which would create a routing loop.
*   Adjust `tun0` if your TUN interface is named differently (e.g., `tun1`).

## Troubleshooting

*   **Permissions**: Most issues stem from not having root/administrator privileges. Always run with `sudo`.
*   **Firewall**: Ensure the UDP port used by the server (e.g., `1111`) is open on the server's firewall (for `192.227.177.171`).
*   **`SHARED_PASSWORD` Mismatch**: If client and server `SHARED_PASSWORD` in `udptun.py` don't match, authentication will fail. The client will show "Logged failed: Incorrent password."
*   **TUN Device Issues**:
    *   Linux: Ensure the `tun` module is loaded (`sudo modprobe tun`).
    *   macOS: TUN devices should generally be available. If not, you might need to install a TUN/TAP driver (e.g., `tuntaposx`).
*   **IP Conflicts**: Ensure the TUN IP addresses (`10.0.0.1`, `10.0.0.2`) do not conflict with existing network interfaces on either machine.
*   **Debug Mode**: Use the `-d` flag on both client and server for verbose output.
*   **`ifconfig` / `ip addr`**: Use these commands to check if the TUN interfaces (`tun0`, `tun1`, etc.) are created and configured with the correct IP addresses.
*   **`ping`**: After setup, try pinging the server's TUN IP from the client (e.g., `ping 10.0.0.1`) and vice-versa. If pings work, the tunnel is up.

## Security Considerations

*   **No Traffic Encryption**: As stated, `udptun` authenticates the connection but does **not** encrypt the data transiting the tunnel. The data is sent as clear IP packets encapsulated in UDP. For sensitive data, use applications that encrypt their own traffic (HTTPS, SSH) or layer `udptun` with another encryption mechanism.
*   **Shared Secret**: The security of the authentication relies entirely on the secrecy of the `SHARED_PASSWORD`. Choose a strong, unique password and change it from the default.
*   **Pickle Usage**: The script uses `pickle` to serialize some control messages. While protected by the shared password, `pickle` can be a security risk if processing untrusted data. In this client-server model with a shared secret, the risk is somewhat mitigated but still present if the shared secret is compromised.
*   **IP Spoofing**: The basic setup doesn't inherently prevent IP spoofing from a compromised client that is already authenticated.
*   **Open Server Port**: Exposing any port to the internet has inherent risks. Ensure your server (`192.227.177.171`) is hardened and monitored.

This guide should help you get started with `udptun`. Remember to adapt IP addresses and configurations if your setup differs from the examples.
