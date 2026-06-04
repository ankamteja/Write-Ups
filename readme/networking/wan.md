# WAN

## Wide Area Network (WAN) Fundamentals

#### Overview

WANs are the infrastructure used to connect geographically dispersed sites—such as corporate offices, data centers, and branch locations—to facilitate centralized services like telephony, databases, and payroll systems. A critical distinction is that a WAN is not inherently the public internet; it is a private network architecture designed for enterprise connectivity.

#### Legacy Technologies

* **Leased Lines:** Dedicated, point-to-point connections (e.g., T1 or T3 lines) between two sites. While highly secure and providing guaranteed performance, they are expensive, difficult to scale, and cumbersome for connecting multiple branches.
* **Frame Relay/ATM:** Older technologies that previously served as alternatives to leased lines but are now considered obsolete in modern networking curricula.

#### Industry Standards

* **MPLS (Multi-Protocol Label Switching):** A long-standing standard that operates at Layer 2.5. It uses labels to create private virtual circuits across a carrier's network, keeping enterprise traffic segregated from other customers. While reliable and capable of Quality of Service (QoS) prioritization, it is currently being phased out in favor of more modern solutions.
* **Metro Ethernet:** A high-speed, fiber-based Layer 2 service typically used for connecting major sites like data centers and corporate campuses. It offers various architectures:
  * **E-Line:** Point-to-point connectivity.
  * **E-Tree:** Hub-and-spoke connectivity.
  * **E-LAN:** Full-mesh connectivity (functioning like a massive, distributed switch).

#### Modern Approaches

* **Site-to-Site VPNs:** An economical method that utilizes the public internet to connect branch offices. Traffic is encrypted using VPN headers to ensure security. Because it traverses the public internet, it lacks the inherent reliability and performance guarantees of private WAN circuits.
* **SD-WAN (Software-Defined WAN):** The modern, primary competitor to MPLS. It optimizes standard internet connections to provide the benefits of private WANs without the high costs of legacy hardware. It is increasingly relevant as enterprises shift application traffic from on-premises data centers to public cloud environments like AWS and Azure.

#### Key Takeaways

* **QoS (Quality of Service):** Essential for prioritizing latency-sensitive traffic, such as voice-over-IP (VoIP), ensuring high-quality communications even when bandwidth is constrained.
* **Network Evolution:** Enterprise networking is moving away from rigid, private, backhauled circuits toward more flexible, cloud-integrated, and software-defined solutions.

<br>
