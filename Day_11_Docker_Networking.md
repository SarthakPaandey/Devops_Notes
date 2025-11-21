# Day 11: Docker Networking Deep Dive & Packet Flow
**Date:** 21st November

## 1. Deep Dive into Docker Bridge Network
The default networking mode.
*   **Virtual Ethernet Bridge (`docker0`):** Docker creates a software bridge on the host. It acts like a virtual switch.
*   **Veth Pairs:** Each container gets a virtual interface pair.
    *   One end is in the container namespace (`eth0`).
    *   One end is plugged into the bridge (`veth...`).
*   **NAT (Network Address Translation):**
    *   Can containers talk to the internet? Yes, via **Masquerading** (Source NAT). The Host helps the container pretend to have the Host's IP.
    *   Can the internet talk to containers? Only via **Port Mapping** (Destination NAT). `iptables -t nat -A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80`.

### Visualizing VETH Pairs
```bash
# On Host
ip link show
# You see veth1234@if5

# Inside Container
ip link show
# You see eth0@if6
```

---

## 2. Overview of Host and None Network
### Host Network (`--net=host`)
*   **Concept:** Container shares the Host's networking namespace. It sees the Host's `eth0` directly.
*   **Pros:** Maximum performance (No NAT overhead). Good for VoIP/Streaming.
*   **Cons:** Port conflicts. If app uses Port 80, you can only run *one* instance. Security risk (container has full network access).

### None Network (`--net=none`)
*   **Concept:** Loopback (`lo`) interface only. No internet, no LAN.
*   **Pros:** Strict security for batch jobs that just process data (e.g., Image resizing, ML model training).
*   **Cons:** Cannot `apt-get install` anything.

---

## 3. Creating Your Own Custom Network
The default `bridge` does not support DNS resolution by container name. Custom bridges do.
1.  **Isolation:** Containers on `net-a` cannot talk to `net-b`.
2.  **DNS:** Automatic service discovery.

```bash
# Create
docker network create --driver bridge --subnet 192.168.0.0/24 my-net

# Connect running container
docker network connect my-net my-container
```

---

## 4. Advanced: Macvlan & IPVlan
Legacy applications sometimes need a real IP on the physical network (no NAT).

### Macvlan
*   Each container gets a unique MAC address.
*   To the switch, it looks like a physical device.
*   **Promiscuous Mode:** Requires the physical switch to allow multiple MACs on one port.
*   **Use Case:** Legacy apps that require L2 visibility.

### IPVlan (L2 or L3)
*   Shares the Host MAC address but has different IPs.
*   **L2 Mode:** Acts like a switch.
*   **L3 Mode:** Acts like a router (No broadcast/multicast).
*   **Use Case:** High scale (thousands of containers) where Macvlan hits MAC table limits.

---

## 5. DNS Resolution Internals
How does `ping db` work?
1.  Container has `/etc/resolv.conf` pointing to `127.0.0.11` (Embedded DNS Server).
2.  Docker Daemon listens on this address.
3.  **If name == container_name in same network:** Return internal IP.
4.  **If name == google.com:** Forward to Host's DNS (`8.8.8.8`).

---

## 6. Packet Flow Analysis (Troubleshooting)
**Scenario:** App can't connect to DB.
1.  **Check DNS:** `docker exec app nslookup db` -> Should return IP.
2.  **Check IP routing:** `docker exec app route -n`.
3.  **Tcpdump (Sniffing):**
    *   **On Host:** `tcpdump -i docker0 port 3306` (Capture bridge traffic).
    *   **Inside Container:** `tcpdump -i eth0`.
    *   *Note:* You usually need `nicolaka/netshoot` image for tools.

```bash
# Debugging with Netshoot
docker run -it --net container:app nicolaka/netshoot
# Now you are inside 'app' network namespace with all tools (tcpdump, curl, nmap).
```

---

## 7. Interview Questions
1.  **Q: What is the difference between `-p 8080:80` and `-P`?**
    *   *A:* `-p` maps a specific port. `-P` (uppercase) maps ALL exposed ports to random high ports on the host (Ephemeral ports).
2.  **Q: Can containers on different bridge networks communicate?**
    *   *A:* Not by default (Layer 2 isolation). You must route traffic through the host (Layer 3) or connect the container to both networks.
3.  **Q: What is the difference between Overlay and Bridge?**
    *   *A:* **Bridge:** Single Host networking. **Overlay:** Multi-Host networking (Swarm/K8s). Overlay encapsulates packets (VXLAN) to send them across the physical network to another Daemon.
4.  **Q: Why does `ping google.com` fail inside my container?**
    *   *A:* Check Host firewall (UFW/IPTables) blocking forwarding. Check if DNS (`127.0.0.11`) is crashing. Check if `--net=none` was used.
