# How to build a Network

the importance of robust network architecture, focusing on avoiding "single points of failure" and transitioning from poor, daisy-chained designs to structured, professional models.

#### The Problem with Bad Network Designs

* **Single Points of Failure:** A major design flaw where a single broken cable or failed switch brings down the entire network. Daisy-chaining switches together creates a fragile structure prone to these failures.
* **Small Office/Home Office (SOHO) Limitations:** While basic setups work for home use, they lack the redundancy and capacity required for growing businesses.

#### The Two-Tier Network Architecture (Collapsed Core)

This model provides better reliability for medium-sized businesses by separating the network into two functional layers:

* **Access Layer:** Where end-user devices (computers, servers, Raspberry Pis, etc.) connect to the network.
* **Distribution Layer:** Uses multi-layer (Layer 3) switches to route traffic, manage VLANs, and apply security policies. It acts as the backbone for the access layer.
* **Redundancy:** To minimize single points of failure, this design often employs dual distribution switches and redundant physical connections.

#### The Three-Tier Network Architecture

Used for large campus environments with multiple buildings, this model adds a dedicated "Core" layer:

* **Access Layer:** Provides connectivity for end devices.
* **Distribution Layer:** Acts as the intermediary between access and core layers, handling routing and policy enforcement.
* **Core Layer:** The high-speed backbone of the network. Its primary role is to move data with extreme speed, low latency, and high reliability between different distribution switches.

#### Key Takeaways

* **Scalability:** The three-tier model allows for much easier expansion across large campuses by preventing the need for messy, direct mesh connections between buildings.
* **Hardware Requirements:** Distribution and Core layers require enterprise-grade switches with high bandwidth capacity (backplane throughput) to handle the aggregate traffic of the entire network.
* **Redundancy is Expensive:** Designing a network that avoids single points of failure requires doubling up on critical hardware and connections, which significantly increases costs but is essential for business continuity.

<br>
