# P2S

## Intro and problem statement:

In a world increasingly reliant on real-time communication, from Zoom calls to WhatsApp voice notes, ensuring smooth, lag-free transmission is no longer a luxury — it's a necessity. VoIP is the protocol used in these cases. However, achieving low latency, minimal jitter, and a stable, high bitrate over a shared network is far from trivial. <br>
Network congestion, jitter, and bitrate fluctuations often lead to choppy audio, lag, and poor call experience—especially under unstable network conditions.  These issues become even more critical in VoIP (Voice over IP) applications, where voice data is split into small packets and transmitted over the internet. Unlike streaming or downloads, VoIP is highly sensitive to delay and packet loss. Even a few milliseconds of lag or a dip in bitrate can degrade the clarity and flow of conversation, resulting in user frustration.

<img src="/screenshots/1.png">

### Our goal: 
Minimize latency and stabilize bitrate for VoIP packets while they traverse the Linux network stack, especially under varying network load.

## What is VoIP?
Voice over IP (VoIP) is the technology that enables voice communication over the internet rather than traditional phone networks. VoIP splits and encodes voice into packets (typically using the RTP) and transmits them over the internet. These packets travel through the Linux network stack on both sending and receiving devices. Unlike traditional data transfers, VoIP is real-time and delay-sensitive, requiring low-latency, low-jitter, and highly consistent bitrate to maintain intelligibility and quality.
VoIP quality heavily depends on:
- Jitter: Variability in packet arrival times.
- Latency: Delay between sending and receiving.
- Bitrate: Speed of transmission — higher and stable bitrate usually means better voice quality.

VoIP isn't just used in calling apps — it's foundational to services like Google Meet, Discord, Microsoft Teams, and even emergency systems.

To address the problem of high and variable latency and unstable bitrate in VoIP packet flow, we introduce a targeted solution within the Linux networking stack that:
- Monitors and prioritizes VoIP packets
- Reduces queuing delay using AQM (CoDel)
- Leverages eBPF for efficient, kernel-level filtering
- Implements traffic shaping and queuing discipline using tc

## Components of the solution:
1. XDP & eBPF:
Attach eBPF programs at XDP or tc ingress hook to identify VoIP packets early based to get UDP ports (typical for RTP), DSCP values (e.g., EF for expedited forwarding)and Protocol headers.
2. tc (Traffic Control) + qdisc:
Classify VoIP into high-priority class, ensuring low delay. 
3. AQM (Active Queue Management):
Use Hierarchical Token Bucket (HTB) to shape and isolate traffic into separate classes. Attach fq_codel (Fair Queueing Controlled Delay) as the queuing discipline to limit bufferbloat and maintain fairness. fq_codel dynamically manages queue length and applies ECN (Explicit Congestion Notification) when supported.
4. QoS Support:
DSCP-based classification to preserve and propagate QoS tags for traffic shaping.
5. Shell Scripts:
Used to automate setup of eBPF hooks, HTB hierarchy, and tc filter rules, as well as toggle priority modes dynamically and log performance metrics.

This ensures that VoIP packets receive preferential treatment and maintain a stable, high bitrate with minimum delay, even in the presence of background traffic. Additionally we try to avoid excess starvation of other services.

## What is ebpf?

Imagine you're building a powerful monitoring tool. You want to inspect every network packet or track system calls happening inside Linux — but you don’t want to write risky kernel code or recompile the entire OS. That’s where eBPF (Extended Berkeley Packet Filter) comes in — a cutting-edge kernel technology that lets you run safe, custom programs inside the Linux kernel, on the fly, enabling developers to enhance kernel capabilities dynamically without modifying kernel code or loading modules. It’s like writing “mini-plugins” for the kernel without restarting or risking a crash.

Let’s walk through how it all works :

<img src="/screenshots/2.png">
Credits: <a href="https://ebpf.io/what-is-ebpf/">ebpf.io</a>

- Development Phase: Writing eBPF Code
Everything starts with a small piece of C code — but not your everyday C but a restricted, safer version of a C program, specifying exactly what you want.

- Runtime Phase: Loading It into the Kernel
This is where the magic happens..
You use a loader to load the bytecode into the kernel. Tools like bcc handle this part for you. They call the kernel using the bpf() syscall.
But wait — the Linux kernel doesn’t just blindly trust you…
The Verifier: Kernel Safety Gatekeeper
Only if it passes the ebpf verifier’s strict audit does your code get accepted — ensuring that you enhance kernel capabilities safely.
The kernel JIT compiles the eBPF bytecode into fast native instructions and attaches it to a specific event, which later triggers this code.

## Workflow:
Here’s how a VoIP packet is handled in our system from the moment it arrives at the Network Interface Card (NIC):

<img src="/screenshots/3.bmp">

Packet Processing Flow
1. NIC receives incoming VoIP packet
2. eBPF filter (attached at the kernel level) identifies VoIP packets based on packet metadata (e.g., port, DSCP)
3. Packets are marked or tagged accordingly
4. tc (Traffic Control) receives the packet and places it in an appropriate queue
5. CoDel (Controlled Delay) AQM discipline is applied to the VoIP queue to manage bufferbloat and control latency

The packet is then passed to the user-space for application-level handling.

<img src="/screenshots/4.png">

The eBPF Program filters packets at kernel level, classifies based on predefined VoIP rules (e.g., UDP port or DSCP marking). Tc Rules attach queuing disciplines and direct packets to appropriate traffic classes. CoDel AQM dynamically manages queue to maintain low latency. HTB(Heirarchical Tocket Bucket) and Fq_Codel are used here.
HTB ensures proper implementation of prioritization and fq_Codel tries to reduce starvation of other services running. 

This layered pipeline ensures predictable latency and consistent bitrate—core requirements for VoIP traffic.

## Implementation:

- Speeding up the in-kernel packet processing at Traffic Control:
  When the packet is encapsulated in socket buffer (__sk_buff) , the kernel processing of the socket buffer is sped up by increasing the priority of the socket buffer. This ensures faster enqueing of RTP packets in qdisc. This is done by eBPF programs at traffic control .

- Identifying the port number : 
 	The port number of WebRTC(used by VoIP) is found using 	Netstat and is stored in BPF maps which can communicate 	the right port number to the eBPF programs with minimal 	overhead.

- Hook program to Traffic Control :  
 	The program named “egress.c” is attached to the “Classifier” 	section of egress path. Similary the “ingress.c” is attached to 	the “Classifier” section of the ingress path. It identifies RTP 	packets by parsing the packet and 	matching the port number 	with the value stored in the BPF map. The socket buffer 	priority is then set to highest possible value i.e 1. The line 	of 	code that does this is given below : 	                           	code : skb->priority = 1;
        
- Accelerating NIC Access with DSCP Marking at XDP:
  In the IPv6 header, the DSCP value of the TOS(Type OF Service) field decide the priority and type of service the network packets will receive . The DSCP value 46 (Expedited Forwarding) is the highest priority used for real-time communication. This DSCP marking ensures minimum latency and higher priority on the system NIC(Network Interface Card).
  The DSCP values are marked at the egress via Traffic Control’s “Classifier” section. On the ingress, when a packet is to be forwarded from one interface to the other, the DSCP value is marked  via XDP(Express Data Path) which is the fastest way to parse and modify a packet even before a socket buffer is created. The XDP modifies the packet even before it is copied into the main memory and resides in the NIC. This reduces any kernel processing overhead and thus speeds up the packet forwarding. 

- Traffic Shaping with AQM and DSCP-Aware HTB Queues : 
  At egress, in order to prioritize the transmission of VOIP traffic as well as minimize packet loss of general traffic, a stack of HTB(Hierarchical Tocken Bucket) followed by fqCodel qdisc is used . The traffic is redirected to a virtual interface and the bandwidth is divided into 2 classes : 
    1. 1:10 for packets with DSCP value as 46, reserving 40kbps and can borrow at max till 100kbps. This class is used by VoIP.
    2. 1:20 for general traffic with maximum 500Mbps bandwidth.<br>
    
  This guarantees a minimum bandwidth for VoIP packets and prevents starvation at NIC for general traffic. Both the HTB classes use their own fqCodel qdisc and AQM(Active Queue Management). The HTB only decides which class to transmit, it doesn’t decide which flow to transmit, thus fqCodel acts as a scheduler for each flow within a class controlling buffer bloat and fair queuing for each flow within the class. <br>
  The HTB enforces inter class scheduling for NIC but doesn’t enforce in-class scheduling. On the other hand fqCodel can manage fair queuing between different flows and minimize congestion, but cannot prioritize a particular flow. A combination of HTB and fqCodel can achieve both, inter-class prioritization and intra-class fairness.


## Testing:

The Goal of the test is to see wether the setup shows the desired effect on the latency of VoIP packets. We use 2 important metrics to analyse the performance of the setup : 
1. Jitter : A measure of variance in latency. Jitter has an adverse effect on sound quality because the RTP buffering algorithm cannot decide the window size if variance in latency is very high
2. Bitrate : a reduction in jitter should not come at the cost of packet loss, this is ensured by monitoring and reducing bitrate as well.

The test id done using a client-server system. The client sends 2 streams of packets :
1. VoIP stream: This stream contains VoIP packets generated from webRTC application.
2. Congestion stream : This is generated from iPerf3 tool which sends packets at a very high rate of 20 parallel 1Gbps streams. This simulates network congestion to see the performance of the setup in a highly congested environment.

In order to avoid any external interference and unmonitored traffic, the congestion stream and webRTC stream were redirected to a virtual interface with fixed bandwidth. 

Every test consists of two 10 minute run of monitored webRTC transmission along with a congestion stream to create stress on the NIC : 
1. The first run is done without the qDisc stack setup, the jitter and bitrate are plotted throughout the test run and the average jitter and bitrate is recorded.
2. The second run is done in the same way with the qDisc stack setup, the jitter and bitrate is plotted on a graph and the average jitter and bitrate for the entire duration is recorded.  

## Result
12 different tests were conducted and the conclusion derived from the test results are that the birate got stabilised and the ther was considerable reduction in jitter.

## Setting up
- Clone github repo:
```bash
git clone https://github.com/Ananya-A-K/P2S.git
```

- Install ebpf:
Hardware requirements: 2GB RAM, Intel Core with 2 cores
Note: The eBPF programs can run on any hardware as long as it can run Linux!!

Ubuntu commands: 
```bash
sudo apt install linux-headers-$(uname -r) libbpfcc-dev libbpf-dev llvm clang gcc-multilib build-essential linux-tools-$(uname -r) linux-tools-common linux-tools-generic
```

- Setup ebpf code files on local:
```bash
cd ebpf_files
./egress.sh
./ingress.sh
./ingress_setup.sh
./maps.sh
./tc_setup.sh
./xdp.sh
```

- To run the node application on chrome:
```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install # Fix dependencies if needed
google-chrome --unsafely-treat-insecure-origin-as-secure=http://<server-ip> --user-data-dir=/tmp/chrome-test --disable-web-security
```

- On server chrome browser:
```bash
http://localhost:3000
```

- On client chrome browser:
```bash
http://localhost:3000#1
```

- Cleanup:
```bash
./cleanup.sh
```
