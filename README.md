**“Kubernetes Networking Explained – Part 1: The Real Foundations”**

### 1. Why Networking Used to Be a Mess
- Long ago → every company had its own networking system (like different phone chargers).
- Computers from Company A couldn’t talk to computers from Company B.
- Solution → everyone agreed on common rules so everything can talk to everything else.

### 2. The Two Big Rulebooks (Networking Models)
| Model | Layers | What People Actually Use |
|-------|--------|--------------------------|
| OSI Model | 7 layers (theoretical, taught in school) | Rarely used directly |
| **TCP/IP Model** | **4 layers (the one the real internet uses)** | **This is what runs the whole internet and Kubernetes** |

### 3. The 4 Layers of TCP/IP (Super Simple)

| Layer | Nickname | What It Does | Real-Life Example |
|-------|----------|--------------|-------------------|
| 1. Link Layer | Physical + MAC | Moves electricity/bits over cables or Wi-Fi. Uses MAC addresses (like the unique ID printed on every network card) | Ethernet cable, Wi-Fi, the `eth0` interface on your computer |
| 2. Internet Layer | IP Layer | Gives every device an IP address (e.g., 10.0.0.5) and decides the best path to the destination | IP addresses, routing between networks |
| 3. Transport Layer | TCP/UDP Layer | Makes sure data gets to the right app. TCP = reliable (guarantees delivery, re-sends lost packets). UDP = fast but no guarantees | TCP → web browsing (HTTP/HTTPS). UDP → video calls, online games |
| 4. Application Layer | The actual apps | Web browser, email, Kubernetes API, Netflix app, etc. | HTTP, DNS, your favorite website |

Every single packet inside a Kubernetes cluster travels down these 4 layers on the sender and up the 4 layers on the receiver.

### 4. Linux Networking – The Real Foundation of Kubernetes

Before Kubernetes even starts, everything happens inside the Linux kernel.

#### Key Linux Networking “Building Blocks” You Must Know

| Thing | What It Is | Why Kubernetes Loves It |
|-------|------------|--------------------------|
| **Network Interface** | A “door” to the network (real or virtual) | `eth0` (physical), `lo` (loopback), and many virtual ones for containers |
| **Linux Bridge** (e.g., `cni0` or `docker0`) | A virtual Layer-2 switch inside one machine | Connects all containers on the same host so they can talk to each other using MAC addresses |
| **Netfilter** | Hooks inside the kernel where we can inspect or change packets | The engine behind firewalls and NAT |
| **iptables / nftables** | Tools that put rules into Netfilter hooks (DROP, ACCEPT, MASQUERADE, etc.) | Classic way to build firewalls and do NAT (so containers can share one IP) |
| **conntrack** (connection tracking) | Remembers which packets belong to which connection | Makes “stateful” firewalls possible (e.g., allow return traffic automatically) |
| **Routing Table** | Tells the kernel “if destination is X, send packet out this interface” | Basic routing inside a node |
| **IPVS** (IP Virtual Server) | Super-fast in-kernel load balancer (replaces slow iptables for kube-proxy in big clusters) | Used when you have thousands of services – much faster than plain iptables |

### 5. Quick Picture in Your Head (One Kubernetes Node)

```
Physical NIC (eth0) → Linux Bridge (cni0 or docker0)
                             ↑      ↑      ↑
                        Container A   B   C   (each has its own virtual ethernet pair)
```

All containers on the same machine are plugged into the same virtual switch (the bridge).  
That’s why two containers on the same node can ping each other instantly using their pod IPs.

### 6. Packet Journey Inside One Linux Machine (Simplified)

1. Packet arrives on eth0 →  
2. Goes through Netfilter hooks (PREROUTING) →  
3. Kernel checks routing table →  
4. If destination is local container → sent to the Linux bridge →  
5. Bridge looks at destination MAC → delivers to correct container →  
6. On the way out → more Netfilter hooks (POSTROUTING) → often NAT (masquerade) so the packet appears to come from the host’s IP.

### 7. Summary – What You Really Need to Remember Right Now

- The internet runs on the 4-layer TCP/IP model (not the 7-layer one).
- Every Kubernetes node is just a Linux machine with clever virtual networking.
- Containers on the same node talk via a Linux bridge (virtual switch).
- Netfilter + iptables/nftables + conntrack = the firewall & NAT engine.
- IPVS = the high-performance load balancer used in big clusters.

---

# **📘 eBPF Notes**

---

# **1. Introduction to eBPF**

* **Origins & Industry Excitement:** eBPF is gaining rapid adoption for observability, security, and networking due to its efficiency and safety.
  * Linux kernel basics
  * eBPF fundamentals
  * Benefits & use cases
  * Hands-on demos (system call tracing & packet filtering)
* **Main Goal:** Understand how eBPF safely extends kernel functionality without modifying kernel source code.

---

# **2. Linux Kernel & eBPF Basics**

## **Linux Kernel Basics**

* Acts as the **intermediary** between user space (applications) and hardware.
* Provides a unified way for apps to communicate with:

  * Network cards
  * Storage
  * Other hardware
* Prevents direct hardware access by user-space apps → **security + abstraction**.

## **What is eBPF?**

* **Extended Berkeley Packet Filter**
* Allows running **small, safe programs inside the Linux kernel**.
* Written in **restricted C**, compiled to bytecode, then **JIT-compiled** for near-native performance.
* Can be invoked from languages like **Python** via libraries.

## **Safety Features**

* Runs in a **sandboxed environment**.
* **Kernel verifier** checks:

  * No unsafe loops
  * Memory safety
  * Program termination
* Prevents kernel crashes or vulnerabilities.

## **Core Mechanics**

* eBPF programs attach to:

  * **System calls**
  * **Network events**
  * **Tracepoints**
  * **Function entry/exit**
* Triggered by **events** like packet arrival or mkdir call.
* Uses **helper functions** for:

  * Socket operations
  * Collecting metadata
  * Working with maps

## **eBPF Maps**

* Shared **key-value stores** between kernel and user space.
* Used for:

  * Storing state
  * Metrics/telemetry
  * Runtime rules/policies

## **Common Use Cases**

* **Networking:** packet filtering, DDoS mitigation.
* **Security:** runtime enforcement, exfiltration prevention.
* **Observability:** tracing syscalls, performance profiling.

---

# **3. Running a Simple eBPF Program **

## **Objective**

Trace system calls, specifically `mkdir`.

## **Tools Used**

* **Python + BCC (BPF Compiler Collection)**

## **Code Concept**

* Python script loads an embedded C eBPF program.
* Program prints a log when **mkdir** syscall occurs.

## **Demo Steps**

1. Run script:
   `sudo python3 ebpf_system_calls.py`
2. In another terminal:
   `mkdir test`
3. Output appears in real-time showing syscall trace.

## **Visuals**

* Split-screen terminals.
* Immediate output on events.

## **Key Insight**

Real-time kernel-level monitoring without modifying the kernel → ideal for **observability pipelines**.
---
# **4. Packet Filtering with eBPF (5:05 – 6:00)**

## **Objective**

Block specific traffic:

* SSH on port 22
* ICMP (ping)

## **Lab Setup**

* Two Ubuntu VMs:

  * Ubuntu-1 → Source
  * Ubuntu-2 → Target (runs eBPF filter)

## **How It Works**

* eBPF program inspects packet headers:

  * IP
  * TCP/UDP ports
  * Protocol
* Uses eBPF map rules to decide:

  * **Allow**
  * **Drop**

## **Demo Steps**

1. Check baseline → ping & SSH work.
2. Load filter:
   `sudo python3 ebpf_packet_filter.py`
3. After filter:

   * Ping times out
   * SSH fails
4. Stop script → traffic resumes.

## **Key Insight**

Enables runtime policy enforcement for:

* Microsegmentation
* Zero-trust networking
* Security without iptables/kernel changes
---
# **5. Conclusion & Next Steps**

## **Recap**

* eBPF offers:

  * Safe kernel programmability
  * High performance via JIT
  * No need to patch kernel
  * Wide use in networking, observability, security

## **Resources**

* Cisco U. eBPF Tutorial – *u.cisco.com/tutorials/5342*
* Official eBPF Site – *ebpf.io*
* Hands-on Labs – *isovalent.com/labs*
* Books: *What is eBPF*, *Learning eBPF*
* Packet Filter Tutorial – *u.cisco.com/tutorials/5582*
---

# **6. Visual Elements & Diagrams **

* **Kernel Diagram:** User space → kernel → hardware.
* **Demo Screens:** Terminal logs and packet filtering behavior.
* **No code onscreen** (only referenced; full examples in tutorials).

## **Key Takeaways**

* eBPF extends kernel behavior **without modifying kernel code**.
* Ensures safety via verifier & sandboxing.
* Supports advanced security, networking, and monitoring use cases.
* Integrates well with modern DevOps tools and cloud-native environments.


Architecture: Bytecode → JIT → Hooks → Helpers → Maps.
Safety & Speed: Verifier ensures no risks; runs at kernel speed.
Practical Wins: Demos show easy setup for tracing/filtering; ideal for DevOps/SecOps.
Why It Matters: Replaces risky modules; powers tools like Cilium for Kubernetes networking.
Next Steps: Try BCC in a VM; explore linked labs for deeper dives.
