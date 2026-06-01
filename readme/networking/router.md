# Router

#### **What is a Network Router?**

This video provides an introduction to the role of routers in networking. Below are the key takeaways from the lesson:

**The Role of a Switch vs. Router**

* **Switches:** These are used to connect devices on the same local network. They operate at Layer 2 of the OSI model, primarily using MAC addresses to ensure data packets reach the correct destination within a local segment.
* **Routers:** These are essential for connecting two or more different networks. They operate at Layer 3 and use IP addresses to direct data traffic between these distinct networks.

**Key Networking Concepts**

* **IP Addresses:** These act as the addresses for individual devices. Devices on the same network typically belong to the same range of IP addresses.
* **Default Gateway:** This is the router's IP address on a local network. When a device realizes it needs to send data to an IP address outside of its own network range, it forwards the traffic to its gateway (the router) to be passed along to the destination.
* **ARP (Address Resolution Protocol):** This is the process used by devices to find the MAC address of a destination when they only have the IP address. A broadcast message is sent to all devices on the network, and the device that owns the target IP address responds with its MAC address.

**How Data Travels**

* When a device sends data to a different network, it uses both Layer 2 (for the switch to find the router's interface) and Layer 3 (for the router to determine the path to the final destination).
* Routers maintain a "routing table" or a map that tells them which interface to use to reach specific destination networks.

**The Role of DNS**

* DNS (Domain Name Service) is the system that translates user-friendly names (like a website URL) into the actual IP addresses that computers use to communicate across networks.

**Common Commands**

* **show ip route:** This command is used on a Cisco router to display the routing table, which shows how the router is connected to various networks and how it routes traffic between them.
